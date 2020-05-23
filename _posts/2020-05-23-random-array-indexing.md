---
layout: post
title: "Random-based indexing for arrays"
tags: [julia, array, maths]
---

![image](https://imgs.xkcd.com/comics/donald_knuth.png)

*Image credit: "[xkcd: Donald Knuth](https://xkcd.com/163/)" ([CC-BY-NC
2.5](https://creativecommons.org/licenses/by-nc/2.5/))*

Let me introduce a package I recently wrote:
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
subjective arguments (like elegance and ugliness) rather than really compelling
reasons.  That said, 0-based indexing is certainly useful in many situations.

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

## Usage

Let’s come to `RandomBasedArrays.jl`.  You can install it with [Julia built-in
package manager](https://julialang.github.io/Pkg.jl/stable/).  In a Julia
session, after entering the package manager mode with `]`, run the command

```julia
pkg> add RandomBasedArrays
```

After that, just run `using RandomBasedArrays` in the REPL to load the package.
It defines a new `AbstractArray` type, `RandomBasedArray`, which is a thin
wrapper around any other `AbstractArray`.

```julia
julia> using RandomBasedArrays

julia> A = RandomBasedArray(reshape(collect(1:12), 3, 4))
3×4 Array{Int64,2}:
 10  2  6   6
  8  9  1  11
  7  8  8   7
```

Every time you access an element, you conveniently get a random element of the
parent array, making any doubt about first index go away.  This means that you
can use any `Int` as index, including negative numbers:

```julia
julia> A[-35]
6

julia> A[-35]
9

julia> A[-35]
4
```

You can also set elements of the array in the usual way, just remember that
you’ll set a random element of the parent array:

```julia
julia> A[28,-19] = 42
42

julia> A
5×5 Array{Int64,2}:
 13  16   3  25   9
 23  20  16  18   1
  5  17  21   6   8
  5   3  42  10  13
 25   6  23   4  11

julia> A
5×5 Array{Int64,2}:
  9  25  25   3   3
  4  14   9   7  18
 22  14  13  21   2
 11  12  19  14  19
 19   2  21   2  21

```

## Related projects

There are other Julia packages that play with different indexing of arrays, e.g.:

* [`OffsetArrays.jl`](https://github.com/JuliaArrays/OffsetArrays.jl):
  Fortran-like arrays with arbitrary, zero or negative starting indices
* [`TwoBasedIndexing.jl`](https://github.com/simonster/TwoBasedIndexing.jl):
  Two-based indexing (note: this is currently out-of-date and doesn’t work in
  Julia 1.0)


<!-- Local Variables: -->
<!-- ispell-local-dictionary: "british" -->
<!-- End: -->
