---
layout: post
title: How multiple dispatch is useful to me
tags: [julia, draft, multiple dispatch, math]
---

Multiple dispatch is the ability to dispatch a function based on the number and
the run time types of its arguments.  The Julia programming language uses
[multiple dispatch](https://en.wikipedia.org/wiki/Multiple_dispatch) as its main
paradigm, and this has been a central design concept of the language from its
inception.

I already talked about multiple dispatch when I showed a simple implementation
of the [rock-paper-scissors game](2017-11-03-rock-paper-scissors.md).  Multiple
dispatch is useful not only to develop games, but also to write mathematically
correct code.

A key idea of multiple dispatch is that methods don’t belong to a single class,
but they are shared by all the arguments.  Quoting from [Julia
documentation](https://docs.julialang.org/en/v1/manual/methods/):

> does the addition operation in `x + y` belong to `x` any more than it does to
> `y`?  The implementation of a mathematical operator generally depends on the
> types of all of its arguments.  Even beyond mathematical operations, however,
> multiple dispatch ends up being a powerful and convenient paradigm for
> structuring and organizing programs.

## Complex division

An interesting example of how multiple dispatch allows one to more easily write
mathematically correct code is the division of a complex number.  In particular,
corner division by 0.  Is the 0 real or complex?  Yes, because the answer is
different depending on the nature of the 0.  When you divide a complex number
$$z$$ by (positive) real 0, you’re computing the limit:

$$ \lim_{x \to 0^{+}} \dfrac{z}{x} $$

and it is clear which is the direction in which you take the limit: the real
axis.  Thus, for example, from the division $$(3 - 14i)/0$$ we should expect as
a result $$\infty - i\infty$$, assuming that the limit is taken from the
positive part of the real axis.  When the 0 is complex, instead, the direction
of the limit in the complex plane is not well-defined, and so the result of the
division by a complex zero should be undefined.  This behaviour is also mandated
by the [ISO/IEC 10967 standard](https://en.wikipedia.org/wiki/ISO/IEC_10967)
(part 3, section 5.2.5.5 "Fundamental complex floating point arithmetic").


<!-- comments -->

<!-- * https://github.com/JuliaLang/julia/issues/22983 -->
<!-- * you can achieve something similar with a bunch of `if` -->
<!-- * you can achieve something similar in C++, with the difference that dispatch is -->
<!--   static on the arguments -->

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "british" -->
<!-- End: -->
