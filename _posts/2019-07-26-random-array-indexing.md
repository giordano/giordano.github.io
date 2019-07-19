---
layout: post
title: "Random-based indexing for arrays"
tags: [julia, array, maths, draft]
---

![image](https://imgs.xkcd.com/comics/donald_knuth.png)

*Image credit: "[xkcd: Donald Knuth](https://xkcd.com/163/)" ([CC-BY-NC
2.5](https://creativecommons.org/licenses/by-nc/2.5/))*

Let me introduce my latest effort:
[`RandomBasedArrays.jl`](https://github.com/giordano/RandomBasedArrays.jl), a
hassle-free package in the [Julia programming language](https://julialang.org/)
for dealing with arrays.  Every time you access an element of an array, the
first index is random, so this package relieves you from having to remember
whether Julia uses 0- or 1-based indexing: you simply cannot ever know what the
initial element will be.  As an additional benefit, you can use any `Int` to
index a `RandomBasedArray`.

## Motivation

This package takes a new stance in the longstanding debate whether arrays should
have [0-based](https://en.wikipedia.org/wiki/Zero-based_numbering) or 1-based
indexing.

The main motivation for this package is that I'm sick of reading about this
debate.  It is incredibly hard to convince people that there is no "one size
fits all" indexing in programming, both alternatives have their merits:

* 0-based indexing is natural every time you deal with offsets, e.g. when
  referencing memory addresses, or when doing modular arithmetic
* 1-based indexing is natural when you are counting elements: the 1st element is
  "1", the 2nd element is "2", etc...

It is pointless to claim the superiority of one indexing over the other one, as
they’re useful in different situations.

As a matter of fact, many "math-oriented" languages (e.g., Fortran, Julia,
Mathematica, MATLAB, R), that are less likely to fiddle with pointers'
addresses, default to 1-based indexing, even though probably the majority of the
programming languages nowadays uses 0-based indexing.

Many people claim superiority of 0-based indexing over 1-based indexing because
of a popular note by [Edsger
W. Dijkstra](https://en.wikipedia.org/wiki/Edsger_W._Dijkstra): ["Why numbering
should start at
zero"](http://www.cs.utexas.edu/users/EWD/transcriptions/EWD08xx/EWD831.html).
However, I personally find this note unconvincing and partial, mainly based on
subjective arguments (like elegance and ugliness) rather than practical reasons.
That said, 0-based indexing is certainly useful in many situations.

A good programming language, whatever indexing convention it uses, should
provide an abstraction layer to let users forget which is the initial index.
For example, Fortran has [`lbound`](http://fortranwiki.org/fortran/show/lbound)
to reference the first element of an array.  Besides the
[`first`](https://docs.julialang.org/en/v1/base/collections/#Base.first)
function to reference the first element of a collection, the Julia programming
language has different utilities to iterate over collections:

* arrays are iterables, this means that you can write a `for` loop like
  ```julia
  for element in my_array
	  # do things with the `element`...
  end
  ```
  without using indices at all
* the [`length`](https://docs.julialang.org/en/v1/base/collections/#Base.length)
  function gives you the length of a collection, so that Dijkstra's argument
  about the length of a sequence remains aesthetic rather than practical
* [`eachindex`](https://docs.julialang.org/en/v1/base/arrays/#Base.eachindex) is
  a function that returns an iterable object with all the indices of the array.

Some times you need to use the indices of an array and know which is the first
one.  In this case, the abstraction layer above is not useful.  Thus, I think it
is important for a programming language to provide also a way to easily switch
to the most appropriate indexing for the task at hand.  In Fortran you can set a
different initial index for an array with the
[`dimension`](https://docs.oracle.com/cd/E19957-01/805-4939/6j4m0vn8a/index.html)
statement.  Julia allows you to define custom indices for you new array-like
type, as described in the
[documentation](https://docs.julialang.org/en/v1/devdocs/offset-arrays/).  The
most notable application of custom indices is probably the
[`OffsetArrays.jl`](https://github.com/JuliaArrays/OffsetArrays.jl) package.
Other use cases of custom indices are shown in [this blog
post](https://julialang.org/blog/2017/04/offset-arrays).

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "british" -->
<!-- End: -->
