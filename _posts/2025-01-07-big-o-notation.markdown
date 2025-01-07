---
layout: post
title:  "(re)defining Big O notation"
date:   2025-01-07 11:15:00 +0000
categories: jekyll update
katex: true
---

Talking about CS here - sorry to the 0.1 mathematicians reading this.

Disclaimer: math (right after shooing away all the mathematicians lol),

Big O notation is usually used to determine the "time complexity" of an algorithm, or the growth of the running time. But what does that actually mean?

## Preliminaries (skippable if you know these)
### Sets
For a fuller introduction to sets go to [the Wikipedia page](https://en.wikipedia.org/wiki/Set_(mathematics)).
A set is a collection of things, usually mathematical objects like numbers, sets, shapes, functions, and so on of any size. An analogous object are `set`s in Python.

Denoting an element $$x$$ in a set $$S$$ can be denoted as $$x \in S$$.

### Functions
A function is a mathematical object similar to pure functions in computer science, that define a mapping from its "domain" (a set of possible inputs) to its "codomain" (a set of possible outputs).

## Back to big O
Big O notation describes how a function grows asymptotically. In simpler terms, it describes the approximate rate that a function $$f(n)$$ grows with $$n$$. In computer science, $$n$$ is usually used as the *size* of the input. Although big O notation has been defined before, Knuth [\[1\]](#knuth) redefined the functions $$O(f(n))$$, $$\Omega(f(n))$$ and $$\Theta(f(n))$$ as:

> $$O(f(n))$$ denotes the set of all $$g(n)$$ such that there exist positive constants $$C$$ and $$n_O$$ with $$\lvert g(n) \rvert \le Cf(n)$$ for all $$n \ge n_O$$.

> $$\Omega(f(n))$$ denotes the set of all $$g(n)$$ such that there exist positive constants $$C$$ and $$n_O$$ with $$g(n) \ge Cf(n)$$ for all $$n \ge n_O$$.

> $$\Theta(f(n))$$ denotes the set of all $$g(n)$$ such that there exist positive constants $$C$$, $$C'$$ and $$n_O$$ with $$Cf(n) \le g(n) \le C'f(n)$$ for all $$n \ge n_O$$.

Woah, what does that mean? Backtracking a bit, let's focus specifically on this line:

> $$O(f(n))$$ denotes the set of all $$g(n)$$ such that there exist positive constants $$C$$ and $$n_O$$ with $$\lvert g(n) \rvert \le Cf(n)$$ for all $$n \ge n_O$$.

Here, Knuth is defining $$O(f(n))$$ as a *function* whose *domain*, its input, is an expression ($$f(n)$$). Note that $$n$$ here is an indeterminate, similar to the $$x$$es in polynomials and other expressions, except $$f(n)$$ does not necessarily need to be a polynomial (as an example, $$\sqrt{n}$$ isn't a polynomial).

He defines the *codomain*, the output, as a set of expressions $$g(n)$$ that satisify the condition that all of these expressions satisify the condition $$\lvert g(n) \rvert \le Cf(n)$$. This essentially means all positive expressions $$g(n)$$ are less than or equal to $$f(n)$$ multiplied by a coefficient, or multiplier, $$C$$, where $$C$$ is the same for every value of $$n$$. In other words, $$g(n)$$ grows up to as fast as $$Cf(n)$$. In other words, $$O(f(n))$$ is a set whose elements are expressions of asymptotic growth that is as most as fast as $$f(n)$$.

The constraint $$n \ge n_O$$ means that it is true for practical values of $$n$$, or above $$n_O$$.

One thing to note is if, as an example, an algorithm grows at the speed of $$f(n) = O(n^3)$$, then $$f(n)$$ also is $$O(n^{100})$$, as big O is an upper bound.

Although $$O(f(n))$$ is a set, if $$g(n)$$ is contained within it, it is denoted $$g(n) = O(f(n))$$.

From here, we can make a clearer definition of big O notation and the other functions. This can be written as:
> Let $$f(n)$$ be a function whose codomain is an expression of indeterminate $$n$$

> Let $$n_O$$ be a practical lower bound of $$n$$,

> Let $$k$$ be a non-zero integer constant.

> $$O(f(n))$$ is the set of expressions $$g(n)$$ such that, given a constant factor $$k$$, $$g(n)$$ is a lower bound of $$k \cdot f(n)$$ and $$n_O \le n$$

> $$\Omega(f(n))$$ is the set of expressions $$g(n)$$ such that, given a constant factor $$k$$, $$g(n)$$ is a upper bound of $$k \cdot f(n)$$ and $$n_O \le n$$

> $$\Theta(f(n))$$ is the set of expressions $$g(n)$$ such that, given two constant factors $$k$$ and $$k'$$, $$f(n)$$ is between $$k \cdot g(n)$$ and $$k' \cdot g(n)$$ and $$n_O \le n$$

An alternative definition written with limits [\[2\]](#leighton) is:

$$O(f(n)) = \{g(n) \mid \limsup\limits_{n \rightarrow \infty} \frac{f(n)}{g(n)} \le k\}$$

$$\Omega(f(n)) = \{g(n) \mid \liminf\limits_{n \rightarrow \infty} \frac{f(n)}{g(n)} \ge k\}$$

$$\Theta(f(n)) = \{g(n) \mid \lim\limits_{n \rightarrow \infty} \frac{f(n)}{g(n)} = k\}$$

where $$k$$ is defined as the constant factor in the previous definitions.

Another thing to note here is that big Theta notation here has the tightest bounds of the two, requiring $$f(n)$$ to grow the same rate $$g(n)$$.

## Big O in computer science
In mathematics, big O (and Theta and Omega) are used to define the asymptotic complexity of a function, or how fast it grows. Within computer science and analysis of algorithms, its often used to analyze how the running time of an algorithm grows as the input **size** grows. A common misconception is that the indeterminate is the value. As an example, consider this algorithm:

{% highlight python %}
def multiply(a: int, b: int):
    out = 0
    for i in range(b):
        out += a
    return out
{% endhighlight %}

At first glance, ignoring the additional complexity from the addition and resizing from the bigint logic, the running time seems to grow linearly as $$b$$ grows, or $$T(n) = O(n)$$. However, in analysis of algorithms, the indeterminate represents the size, not the value. Usually, the size is defined in the size in bits of the input, $$n = log_2(b)$$, therefore the time complexity is $$T(n) = O(2^n)$$.

Another thing to note is that although it determines the asymptotic complexity of an algorithm, a "better" asymptotic complexity does not necessarily mean an increase in speed. Asymptotically optimal algorithms exist that are practically unlikely to be used, called [galactic algorithms](https://en.wikipedia.org/wiki/Galactic_algorithm).

## Conclusion
big o goog

also stop using big o when big theta more precise

idk what you want me to say here

## Bibliography
Read these below if you want to know more and don't want to get all your info from some random blog.

<a name="knuth">
D. E. Knuth, "Big Omicron and big Omega and big Theta", *SIGACT News*, vol. 8, no. 2, pp 18-24, Apr. 1796, doi: 10.1145/1008328.1008329.
</a>

<a name="leighton">
T. Leighton, R. Rubenfeld, The Asymptotic Cheat Sheet [PDF document]. Available: <https://web.mit.edu/broder/Public/asymptotics-cheatsheet.pdf>.
</a>

## Disclaimer
The info in this blog post has tried to be accurate, yet there may be some issues and mistakes. If you find any, mention me on Mastodon at somehybrid@hachyderm.io and I might have enough motivation to actually fix any of it.

Also, I probably messed up some of the math here. Take everything with lots of seasonings.

## Changelog (woops me error)
push first, proofread after

2025-01-07 (can you really believe its 2025? almost wrote 2022): fix markup errors, fix wording in big O in CS, make definiitions clearer, fix typos, fix definition of limits.
2025-01-07 (again xd): fix typo (my space bar is painnnnnnn)
