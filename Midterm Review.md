## 1. Context-Free Grammars

#### Question (a)

$$
S\to (S)\ | \ [S]\ |\  \{S\}\ | \ SS \ | \ \varepsilon
$$
#### Question (b)
Identify that $j$ must be odd, because $3j+1$ must be even.
$$
S \to aaaSbb \ | \ aab
$$
#### Question (c)
Analyze L:

| $\#_a(L)$ | $\#_b(L)$ allowed |
| --------- | ----------------- |
| 0         | any               |
| 1         | any               |
| 2         | any \ 1           |
| 3         | any               |
| 4         | any               |
| 5         | any \ 3           |
| 6         | any               |
| 7         | any               |
| 8         | any \ 5           |
| 9         | any               |
| 10        | any               |
| 11        | any \ 7           |
So pretty much if `i - 2 % 3 == 0`, not allowed

$S \to aaaSbb$ 
$S \to aaaX$
$S \to \ T$ 

