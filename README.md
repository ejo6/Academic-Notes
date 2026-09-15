This repository contains my notes for all of year 3. As of now, handwritten notes are not taken in obsidian, but are imported in regularly. 
## Course Overviews
### Semester 1
- [[COMPSCI 3307A – Object-Oriented Design & Analysis]]
- [[COMPSCI 3331A – Theory of Computing]]
- [[COMPSCI 3350A – Computer Organization]] 
- [[DATASCI 3000A – Intro to Machine Learning]]
- [[COMPSCI 3305A - Operating Systems]]
### Semester 2
- [[MATH 2500A – Calculus III]]
- [[COMPSCI 3340B - Analysis of Algorithms I]] 
- [[COMPSCI 3342B - Organization of Programming Language]] 
- [[COMPSCI 3377A - Software Project Management]] 
- [[WRITING 2111G - Intro to Professional Writing]] 

## Other Notes 
- [[SEM2/MATH2500A/Math Review/README|README]] - Homepage of Math Notes
- [[Locado/README|README]] - Homepage of Locado Notes

## For GitHub Users
A number of issues arise with GitHub's built-in markdown renderer:
- Many forms of LaTeX math will break, notably $\#$, some matrices. Obsidian renders these correctly
- Tips, examples, etc. that are rendered well in obsidian do not work on GitHub
- Internal links to other files will not work
- Images which are stored in `attachments` will not render
I recommend cloning this repository, and **just opening it with obsidian**. You can clone it to iCloud if you'd like concurrency between devices as well. 

## Formatting Rules
- All files under a parent folder will be ordered (chronologically, magnitudinally, etc.) by a prefix in format "$n$.". For example: *1. Basic Notes on Differential Equations*, *2. Linear Differential Equations*

- Header format "h2", "h3", "h4" are used to subsection documents. Where a secondary "h1" is required, a new note should be created. After the first "h2", all sections where $h(\text{section}) < 2,$ should fall directly under a $h(\text{section}) + 1$. 

- Files with notes majority based off of relevant sources should start with: 
	**Notes based off of:**
	-  {Link/title of source}

- Files should end with a footer in format: 
	**Prev:** \[\[Previous File\]\], **Next** \[\[Next File\]\]. Brackets will auto link.
