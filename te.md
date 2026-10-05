# Order Edit / Cancel Design

Status: draft for review (Nico, Brayden)

## Decisions so far

|#|Decision|
|---|---|
|1|Archive the previous version in a new `OrderSnapshots` table. The `Orders` row keeps its `Id`/`OrderNumber` and always holds the current version.|
|2|Add an `OrderPaymentTransactions` ledger, because one order can now have several PaymentIntents and refunds.|
|3|Locado-Ops owns `admin."DeliveryCycles"`, auto-locks cycles and sets `CutoffLocked`.|
|4|Inventory types: `Products.Quantity` (live stock, decremented at checkout) and `Products.WeeklyRecurringInventory` (the value Ops resets `Quantity` to when a new cycle is created).|
|5|Logged-in users only (`Order.UserId` set to the caller).|
|6|**Full cancel:** refund the money, restock, and restore promos/credits unless they have expired.|
|7|**Edit that costs more:** the user re-confirms through a payment sheet. The edit stays `Pending` until `payment_intent.succeeded`.|

## Rules

- **Edit/cancel window:** `UtcNow < cycle.CutoffDate − 60 min`, where `cycle.Status == Open`, `order.Status == OrderPlaced` and `order.PaymentStatus == succeeded`.
    - `CutoffDate` is `timestamptz`, and Ops writes it as UTC after applying the Toronto offset, so comparing in UTC is DST-safe.
    - Always read the live cycle row, because Ops `ExtendCutoffAsync` can move the cutoff.
- **Edit validation:** at least one line, `SubtotalBeforeDiscount ≥ minimum_order_cad`, and every line active and in stock for its _delta_. Uses the same `CartTotalsCalculator` as checkout.
- **Fees on edit:** `DeliveryFee` and `ServiceFee` are copied from the current order and never recomputed.
- **Money on edit:** `delta = newTotal − currentTotal`.
    - `> 0`: new PaymentIntent (PI) for the delta, confirmed with a payment sheet.
    - `< 0`: partial refund of `|delta|` against the order's charges, newest first.
    - `= 0`: apply immediately.
- **Rate limit:** new authenticated policy `OrderMutations`. Today only guests are limited.

## 1. Data model

Migration notes:

- Backend owns the `public` tables. Backfill one `Charge` ledger row per existing paid order.
- Ops needs `Consumer*` mirror entities for the 3 new tables.

## 2. State

## 3. Eligibility check (shared by edit and cancel)

## 4. Cancel

Stripe (connected acct)PostgresOrderServiceOrdersControllerStripe (connected acct)PostgresOrderServiceOrdersControllerloop[each Succeeded charge, newest first]UserPOST /api/orders/{id}/cancelCancelOrderAsync(userId, orderId)BEGIN, SELECT order FOR UPDATEeligibility check (section 3)Products.Quantity += item.Quantity (each line)restore credits from CreditOrderUsage (RemainingAmount += AmountUsed, Status=available) if not expiredPromoClaim.RedeemedAt = null if claim and campaign not expired / static redemption soft-deletedOrder.Status = Cancelled, CancelledAt = nowinsert Refund ledger rows (Pending), one per charge with refundable balanceCOMMITRefund.Create(PI, remaining refundable)refund idledger row Succeeded (or Failed + log for manual follow-up)CancelResult200 + cancellation emailUser

The refunds run after commit, the same way `EventRegistrationService.CancelRegistrationAsync` does. A Stripe failure leaves a `Failed` ledger row instead of rolling back the cancel.

## 5. Edit, total goes down or stays the same

StripePostgresCartTotalsCalculatorOrderServiceOrdersControllerStripePostgresCartTotalsCalculatorOrderServiceOrdersControlleropt[delta < 0]UserPOST /api/orders/{id}/edits/quote (lines)QuoteEditAsynctotals(lines, fees from order, discounts)new totals + deltaPOST /api/orders/{id}/edits (lines, quoteTotal)SubmitEditAsyncBEGIN, lock order, eligibility, re-quote (reject if != quoteTotal)insert OrderSnapshots (Version = n, edit_archive)Products.Quantity -/+ per-line delta (atomic ExecuteUpdate)replace OrderItems, update totals, Version = n+1, EditedAtOrderEdit Applied, Refund ledger row Pending (if delta < 0)COMMITRefund.Create(newest charge PI, |delta|)ledger Succeeded / Failedupdated order + "order updated" emailUser

## 6. Edit, total goes up (payment sheet)

StripePostgresOrderServiceOrdersControllerWeb/AppStripePostgresOrderServiceOrdersControllerWeb/Appalt[valid and stock available][window closed / stock gone / version moved]UserPOST /api/orders/{id}/edits (lines, quoteTotal)SubmitEditAsynceligibility + re-quote, delta > 0PaymentIntent.Create(delta, metadata: purpose=order_edit, order_id, order_edit_id)OrderEdit Pending (BaseVersion=n, ExpiresAt), ledger EditCharge Pendingclient secret + ephemeral keyconfirm payment sheetwebhook payment_intent.succeeded (purpose=order_edit)ApplyOrderEditAsync(PI) [also called from confirm endpoint, idempotent on OrderEdit.Status]BEGIN, lock order, edit still Pending and order.Version == BaseVersion?snapshot v=n, stock deltas, replace items, totals, Version=n+1OrderEdit Applied, ledger EditCharge SucceededCOMMIT"order updated" emailOrderEdit Failed, COMMITRefund.Create(edit PI, full)"edit couldn't be applied, refunded" emailUser

The webhook needs a new `purpose=order_edit` branch before the cart-order path. Otherwise [OrdersController.cs](https://file+.vscode-resource.vscode-cdn.net/Users/ericoneil/repos/Locado/Locado-Backend/Controllers/OrdersController.cs) treats an edit PI as a new order, since it carries `user_id` metadata.

## Ops (Locado-Ops) touch points

- **`ReconciliationService.LoadConsumerOrdersAsync`** matches only on `Orders.StripePaymentIntentId`. Edit PIs need to match through `OrderPaymentTransactions`, or they show as unattributed. The QuickBooks daily sync reads from reconciliation.
- **`FinancialService`, `PayoutService` and `StripePayoutService`** read the `Orders` row (current version) and already exclude `Cancelled`. No change is expected, but this needs a check.
- **`PackingService` and `VendorPreOrderService`** run after the cutoff, so they always see the final version.
- **Admin order detail page:** show the version history (`OrderSnapshots`) and the payment ledger.

## Still open

1. **Discounts on edit:** keep the original absolute credit amounts (capped at the new subtotal) and recompute only percentage promos from `PromoDiscountPercentage`? _(assumed in the diagrams)_
2. **Referral on cancel:** if the cancelled order was the referee's first order, is the referrer's reward revoked?
3. **`UserFeeExemption` on cancel:** reactivate it like a promo?
4. **Guest orders:** nothing currently links a guest order (`UserId = null`, `GuestEmail`) to an account created later with the same email. Are we adding a claim-by-verified-email step, or are those orders simply not editable?
5. **Pending-edit timeout:** is 30 min OK? A stale PI is cancelled when the edit expires.