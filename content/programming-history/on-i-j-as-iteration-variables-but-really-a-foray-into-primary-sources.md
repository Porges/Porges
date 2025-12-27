---
title: On i, j, … as iteration variables (but really a foray into primary sources)
date: 2012-01-10
---

This question was recently asked on StackOverflow:

> I know this might seem like an absolutely silly question to ask, yet I am too curious not to ask…
>
> **[Why did “i” and “j” become THE variables to use as counters in most control structures?](http://stackoverflow.com/questions/4137785/why-are-variables-i-and-j-used-for-counters)**

The question has generated many answers, from scholarly to spurious — but the thing that has struck me is that no one has attempted to cite their sources or do any research. Why is this, when we live in a time when primary sources are more widely available than ever?

Let’s start with the claims that FORTRAN was the original source for their use in programming languages—while perhaps not the ultimate origin, it may have been the reason that they became widespread in the programming community.

The original manual for Fortran[^1] for the IBM 704 is [readily available online](http://www.fh-jena.de/~kleine/history/languages/FortranAutomaticCodingSystemForTheIBM704.pdf). The first thing I notice is the glorious cover:

[^1]: It isn’t written FORTRAN here. I’m not sure of the nuances of its capitalization.

![[FortranCover.png]]

And sure enough, we can find the definition for the integral variables:

![[FortranVars.png]]

Unfortunately, the path stops here. I can’t find any references by Backus (or anyone else) as to why they chose IJKLMN as the integer variables. However, due to the fact that integer variables in Fortran “are somewhat restricted in their use and serve primarily as subscripts or exponents”,[^2] I am forced to the conclusion that they were used in imitation of those in mathematics. I don’t think we’ll ever know exactly who or when they were introduced to Fortran itself.

[^2]: J.W. Backus, R.J. Beeber, S. Best, R. Goldberg, L.M. Haibt, H.L. Herrick, R.A. Nelson, D. Sayre, P.B. Sheridan, H.J. Stern, I. Ziller, R.A. Hughes, and R. Nutt, [The FORTRAN automatic coding system](http://archive.computerhistory.org/resources/text/Fortran/102663113.05.01.acc.pdf). Pages 188-198. In _Proceedings Western Joint Computer Conference_, Los Angeles, California, February 1957.

What we can do, however, is have a look at when they arose in mathematics. The usual place that i, j, etc. arise is in ‘sigma notation’, using the summation operator Σ. For example, if we write:

$$
\sum_{i=1}^{100} i
$$

We mean $i (= 1) + i (= 2) + i (= 3)$, until $i = 100$, and we can calculate the answer as $1 + 2 + 3 + 4 + \cdots = 5050$. So where did this notation itself come from?

The standard work on the history of mathematical notations is _A History of Mathematical Notations_ by Florian Cajori.[^3] He states that Σ was first used by Euler, in his [_Institutiones calculi differentialis_](http://books.google.co.nz/books?id=sYE_AAAAcAAJ) (1755). We can see the part in question here:

[^3]: Unfortunately, only [the first volume](http://www.archive.org/details/historyofmathema031756mbp) appears to be readily available online. You can see [some of the second volume](http://books.google.co.nz/books?id=bT5suOONXlgC) on Google Books.

![[Euler1755.png]]

This reads (translation by Ian Bruce, from [17thCentryMaths.com](http://17thcenturymaths.com/)):

> 26: Just as we have been accustomed to specify the difference by the sign _Δ_, thus we will indicate the sum by the sign _Σ_; evidently if the difference of the function _y_ were _z_, there will be _z_ = _Δy_; from which, if _y_ may be given, the difference _z_ is found we have shown before. But if moreover the difference _z_ shall be given and the sum of this _y_ must be found, _y_ = _Σz_ is made and evidently from the equation _z_ = _Δy_ on regressing this equation will have the form _y_ = _Σz_, where some constant quantity can be added on account of the reasons given above; […]

Evidently this is not the Σ we are looking for, as Euler uses it only in opposition to Δ (for finite differencing). In fact, Cajori notes that Euler’s Σ “received little attention”, and it seems that only Lagrange adopted it. Here is an excerpt from his [Œuvres](http://www.archive.org/details/oeuvrespublies03lagruoft) (printed [MDCCCLXIX](http://www.wolframalpha.com/input/?i=MDCCCLXIX)):

![[LagrangeŒuvres.png]]

Again, we can see Σ is only used in opposition to Δ. Cajori next states that Σ to mean “sum” was used by Fourier, in his [_Théorie Analytique de la chaleur_](http://www.archive.org/details/thorieanalytiq00four) (1822), and here we find what we’re looking for:

![[Fourier1822.png]]

> The sign $Σ$ affects the number $i$ and indicates that the sum must be taken from $i = 1$ to $i = \frac{1}{0}$. One can also contain the first term $1$ under the sign $Σ$, and we have:
> $$
> 2 \pi \phi (x, t) = \int d \alpha f \alpha \sum_{-\frac{1}{0}}^{+\frac{1}{0}} \cos i (\alpha - x) e^{-ikt}
> $$
>
> It must then have all its integral values from $-\frac{1}{0}$ up to $+\frac{1}{0}$; that is what one indicates by writing the limits $-\frac{1}{0}$ and $+\frac{1}{0}$ next to the sign $Σ$, that one of the values of $i$ is $0$. This is the most concise expression of the solution.[^4]

[^4]: Note that Fourier has no qualms about writing $-\frac{1}{0}$ and $+\frac{1}{0}$ to represent infinities!

Since Fourier explains Σ several times in the book, and not just once, we can assume that the notation is either new or unfamiliar to most readers.[^5] In any case, it doesn’t really matter who invented it, because while we have found our Σ, Fourier doesn’t explain why he uses $i$. In fact, since he uses it to index sequences in other places it appears it must be an already-existing usage.[^6]

[^5]: Knuth also states that the notation arrived with Fourier, so I guess I’m not in bad company.
[^6]: While $i$ is often used as (one of) the indices for a matrix, true matrices weren’t developed until after Fourier’s book was published, we must look elsewhere.

A quick glance at the text by Euler above shows that he uses indexing very rarely (despite the subject of the text being a prime candidate!), and when he does, he uses $m$.

And, this is as far as I got. Time to publish this.
