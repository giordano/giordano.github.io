---
layout: post
title: "What you can learn from accessing an element of an array in Julia"
tags: [draft, julia, performance, benchmarks, array]
---

This innocent question in the [official Slack
workspace](https://julialang.org/slack/) of the Julia programming language

> How can I extract the `Float64` element from a 1x1 `Array{Float64, 1}`?

spurred a series of more or less serious answers:

* use the [`only`](https://docs.julialang.org/en/v1/base/iterators/#Base.Iterators.only) function: `only(a)`
* get the element with index 1: `a[1]`
* [omit all indices](https://docs.julialang.org/en/v1/manual/arrays/#Omitted-and-extra-indices): `a[]`
* use the [`first`](https://docs.julialang.org/en/v1/base/collections/#Base.first) function: `first(a)`
* get the element with index 1, 1: `a[1, 1]`
* use the [`last`](https://docs.julialang.org/en/v1/base/collections/#Base.first) function: `last(a)`
* get the element with index 1, 1, 1, 1, 1, ...: `a[ones(Int, 100)...]` (the
  `...` operator is called
  [splatting](https://docs.julialang.org/en/v1/manual/faq/#The-two-uses-of-the-...-operator:-slurping-and-splatting))
* get the last element with `end`: `a[end]`
* get the element in the first row and last column: `a[begin, end]`
* [reduce](https://en.wikipedia.org/wiki/Reduction_Operator) the array with the
  `sum` function: `reduce(sum, a)`
* get the element with index `floor(Int, -real(exp(π * im)))`: `a[floor(Int, -real(exp(π * im)))]`, or the variant `x[floor(Int, -real(cispi(1)))]`
* [dereference](https://en.wikipedia.org/wiki/Dereference_operator) the pointer
  of `a`
  ([remember?](https://giordano.github.io/blog/2019-05-03-julia-get-pointer-value/)):
  `unsafe_load(pointer(a))`
* use the [`StarWarsArrays.jl`](https://github.com/giordano/StarWarsArrays.jl)
  package and get the fourth element (mentioned in [the post about random
  indexing](https://giordano.github.io/blog/2020-05-23-random-array-indexing/)):
  `StarWarsArray(a)[4]`
* use the
  [`getindex`](https://docs.julialang.org/en/v1/base/collections/#Base.getindex)
  function (the `a[n, ...]` is a synctactic sugar for `getindex(a, n, ...)`):
  `getindex(a, firstindex(a))`
* get a random element from the array: `rand(a)`

## Benchmarking

We can compare the performance of all these versions.  To do this, we can use
the [`BenchmarkTools.jl`](https://github.com/JuliaCI/BenchmarkTools.jl) package
(benchmarks kindly provided by [`@Seelengrab`](https://github.com/Seelengrab)):

```julia
using BenchmarkTools # to install it: ]add BenchmarkTools
using StarWarsArrays # to install it: ]add https://github.com/giordano/StarWarsArrays.jl
using Random # it's a standard library

only_bench(x)         = only(x)
one_idx_bench(x)      = x[1]
omit_bench(x)         = x[]
first_bench(x)        = first(x)
two_ones_idx_bench(x) = x[1, 1]
splat_bench(x)        = x[ones(Int, 100)...]
end_bench(x)          = x[end]
begin_end_bench(x)    = x[begin, end]
reduce_sum_bench(x)   = reduce(sum, x)
floor_bench(x)        = x[floor(Int, -real(exp(π * im)))]
cispi_bench(x)        = x[floor(Int, -real(cispi(1)))]
dereference_bench(x)  = unsafe_load(pointer(x))
getindex_bench(x)     = getindex(x, firstindex(x))
starwars_bench(x)     = StarWarsArray(x)[4] # ಠ_ಠ
rand_bench(x)         = rand(x)

for f in [
    only_bench,
    one_idx_bench,
    omit_bench,
    first_bench,
    two_ones_idx_bench,
    splat_bench,
    end_bench,
    begin_end_bench,
    reduce_sum_bench,
    floor_bench,
    cispi_bench,
    dereference_bench,
    getindex_bench,
    starwars_bench,
    rand_bench,
]
    println(f, ':')
    @btime $f(x) setup=(x=rand(1,1))
end
```

On my computer I get

```
only_bench:
  2.800 ns (0 allocations: 0 bytes)
one_idx_bench:
  1.631 ns (0 allocations: 0 bytes)
omit_bench:
  2.796 ns (0 allocations: 0 bytes)
first_bench:
  1.630 ns (0 allocations: 0 bytes)
two_ones_idx_bench:
  1.685 ns (0 allocations: 0 bytes)
splat_bench:
  4.524 μs (5 allocations: 1.75 KiB)
end_bench:
  1.378 ns (0 allocations: 0 bytes)
begin_end_bench:
  1.644 ns (0 allocations: 0 bytes)
reduce_sum_bench:
  1.901 ns (0 allocations: 0 bytes)
floor_bench:
  1.630 ns (0 allocations: 0 bytes)
cispi_bench:
  1.631 ns (0 allocations: 0 bytes)
dereference_bench:
  1.369 ns (0 allocations: 0 bytes)
getindex_bench:
  1.371 ns (0 allocations: 0 bytes)
starwars_bench:
  1.371 ns (0 allocations: 0 bytes)
rand_bench:
  9.599 ns (0 allocations: 0 bytes)
```

Unsurprisingly, the fastest solution is dereferencing a pointer, which compiles
down to the following code

```julia
julia> x = rand(1,1);

julia> @code_native debuginfo=:none dereference_bench(x)
        .text
        movq    (%rdi), %rax
        vmovsd  (%rax), %xmm0                   # xmm0 = mem[0],zero
        retq
        nopl    (%rax,%rax)
```

This is pretty much what you'd expect from the [corresponding C
code](https://godbolt.org/z/freGKzaof).

What's remarkable is that the `StarWarsArrays.jl` package achieves about the
same performance: accessing an element of a custom data structure using a funny
custom indexing, and written without having performance in mind, is just as
efficient as dereferencing a pointer.

However, subnanoseconds differences aren't credible: most functions that clocked
in the range 1.3-1.6 nanoseconds have basically the same performance.  If you
run the benchmarks again you'll find that their performance get more or less
close to that of dereferencing.  The difference with dereferencing is that all
other functions do [bounds
checking](https://en.wikipedia.org/wiki/Bounds_checking), which on average lead
to slightly larger runtime.  However, in high-performance computing you usually
want to make sure to be within bounds at a higher level and skip bounds checks
when accessing indivual elements.  If you're 100% sure that the element of the
array you want to access is within its bounds, you can use the macro
[`@inbounds`](https://docs.julialang.org/en/v1/base/base/#Base.@inbounds) to
skip runtime bounds checks.

To see the performance when skipping bounds checks in the above functions, you
can either use `@inbounds` in their definitions (e.g., `one_idx_bench(x) =
@inbounds x[1]`, etc...), or start Julia with the `--check-bounds=no` flag (this
flag is _not_ recommended for general use, as it will skip _all_ bounds checks,
not only where you know it's safe to do so).  With `--check-bounds=no` I get

```
only_bench:
  2.725 ns (0 allocations: 0 bytes)
one_idx_bench:
  1.369 ns (0 allocations: 0 bytes)
omit_bench:
  1.369 ns (0 allocations: 0 bytes)
first_bench:
  1.369 ns (0 allocations: 0 bytes)
two_ones_idx_bench:
  1.369 ns (0 allocations: 0 bytes)
splat_bench:
  4.510 μs (5 allocations: 1.75 KiB)
end_bench:
  1.369 ns (0 allocations: 0 bytes)
begin_end_bench:
  1.630 ns (0 allocations: 0 bytes)
reduce_sum_bench:
  1.960 ns (0 allocations: 0 bytes)
floor_bench:
  1.408 ns (0 allocations: 0 bytes)
cispi_bench:
  1.369 ns (0 allocations: 0 bytes)
dereference_bench:
  1.369 ns (0 allocations: 0 bytes)
getindex_bench:
  1.368 ns (0 allocations: 0 bytes)
starwars_bench:
  1.369 ns (0 allocations: 0 bytes)
rand_bench:
  9.606 ns (0 allocations: 0 bytes)
```

As expected, most functions now more consistently perform as well as
dereferencing.  This is confirmed by the fact they are all compiled down to the
same efficient code, even the most bizarre ones like `floor_bench` and
`starwars_bench`:

```julia
julia> @code_native debuginfo=:none omit_bench(x)
        .text
        movq    (%rdi), %rax
        vmovsd  (%rax), %xmm0                   # xmm0 = mem[0],zero
        retq
        nopl    (%rax,%rax)

julia> @code_native debuginfo=:none floor_bench(x)
        .text
        movq    (%rdi), %rax
        vmovsd  (%rax), %xmm0                   # xmm0 = mem[0],zero
        retq
        nopl    (%rax,%rax)

julia> @code_native debuginfo=:none getindex_bench(x)
        .text
        movq    (%rdi), %rax
        vmovsd  (%rax), %xmm0                   # xmm0 = mem[0],zero
        retq
        nopl    (%rax,%rax)

julia> @code_native debuginfo=:none starwars_bench(x)
        .text
        movq    (%rdi), %rax
        vmovsd  (%rax), %xmm0                   # xmm0 = mem[0],zero
        retq
        nopl    (%rax,%rax)
```

## Lessons learned

* To seriously answer the question that led to this digression, I think good
  ways to access the only element of a 1x1 array are `only(a)` (which however
  always does a runtime check to make sure that's the only element of the array,
  incurring a performance hit), `a[1]`, `a[begin]` `a[1, 1]`, or `a[]`
* Custom Julia data types are efficient!  The fact that a custom data type with
  custom indexing like `StarWarsArray` has basically the same performance as
  dereferencing a pointer (_exactly_ same native code when skipping all bounds
  checks) is a testament to the power and expresiveness of the language: you can
  write your code in a high-level form, and have it compiled down to very
  efficient code
* Splatting a long array can be _bad_ for performance, both at compile- and
  run-time.  Splatting is a convenient syntax, but use it with great care,
  especially when you have many elements
* Reduction has relatively little overhead
* `rand(a)` performed quite bad, but actually most of the time is spent in
  accessing the global default random number generator (RNG): if you care about
  performance (and reproducibility, too) you should _always_ pass in the RNG as
  argument (see also [The need for rand
  speed](https://bkamins.github.io/julialang/2020/11/20/rand.html) by Bogumił
  Kamiński):
  ```julia
    julia> @btime rand(x) setup=(x=rand(1,1))
    9.596 ns (0 allocations: 0 bytes)
  0.08218288442687438

  julia> @btime rand($(Random.default_rng()), x) setup=(x=rand(1,1))
    4.770 ns (0 allocations: 0 bytes)
  0.8112313537560083
  ```
  In the second benchmark we explicitly passed the RNG to `rand` and can see
  that avoiding accessing the global RNG saves more than 50% of runtime.
  Performance is still far from ideal though.  As an aside, if the collection is
  a short tuple (<= 4 elements), we can get again the same performance as
  derefercing a pointer:
  ```julia
  julia> @btime rand($(Random.default_rng()), Ref(x)[]) setup=(x=(rand(),))
    1.369 ns (0 allocations: 0 bytes)
  0.4308805082102649
  ```

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "british" -->
<!-- End: -->
