---
layout: post
title: Computing the hexadecimal value of pi
tags: julia maths pi
---

The
[Bailey–Borwein–Plouffe formula](https://en.wikipedia.org/wiki/Bailey%E2%80%93Borwein%E2%80%93Plouffe_formula) is
one of the
several
[algorithms to compute $$\pi$$](https://en.wikipedia.org/wiki/Approximations_of_%CF%80).
Here it is:

$$ \pi = \sum_{k = 0}^{\infty}\left[ \frac{1}{16^k} \left( \frac{4}{8k + 1} -
\frac{2}{8k + 4} - \frac{1}{8k + 5} - \frac{1}{8k + 6} \right) \right] $$

What makes this formula stand out among other approximations of $$\pi$$ is that
it allows one to directly extract the $$n$$-th fractional digit of the
hexadecimal value of $$\pi$$ without computing the preceding ones.

[![Digits of pi](https://cloud.githubusercontent.com/assets/52289/23835253/0277a45e-075c-11e7-9431-14b8613e15ce.png)](https://cloud.githubusercontent.com/assets/52289/23835253/0277a45e-075c-11e7-9431-14b8613e15ce.png)

_Image credit: [Cormullion](https://github.com/cormullion), Julia
code
[here](https://gist.github.com/cormullion/e979d819e478da73280faaeb67490888)._

The Wikipedia article about the Bailey–Borwein–Plouffe formula explains that the
$$n$$-th fractional digit (well, actually it’s the $$n+1$$-th) $$d_{n}$$ is
given by

$$ d_{n} = 16 \left[ 4 \Sigma(n, 1) - 2 \Sigma(n, 4) - \Sigma(n, 5) - \Sigma(n,
6) \right] $$

where

$$ \Sigma(n, j) = \sum_{k = 0}^{n} \frac{16^{n-k} \bmod (8k+j)}{8k+j} + \sum_{k
= n+1}^{\infty} \frac{16^{n-k}}{8k+j} $$

Only the fractional part of expression in square brackets on the right side of
$$d_{n}$$ is relevant, thus, in order to avoid rounding errors, when we compute
each term of the finite sum above we can take only the fractional part.  This
allows us to always use ordinary double precision floating-point arithmetic,
without resorting to arbitrary-precision numbers.  In addition note that the
terms of the infinite sum get quickly very small, so we can stop the summation
when they become negligible.

## Implementation in Julia language

Here is a [Julia](https://julialang.org/) implementation of the algorithm to
extract the $$n$$-th fractional digit of $$\pi$$:

{% highlight julia linenos %}
# Return the fractional part of x, modulo 1, always positive
fpart(x) = mod(x, one(x))

function Σ(n, j)
    # Compute the finite sum
    s = 0.0
    denom = j
    for k in 0:n
        s = fpart(s + powermod(16, n - k, denom) / denom)
        denom += 8
    end
    # Compute the infinite sum
    num = 1 / 16
	while (frac = num / denom) > eps(s)
        s     += frac
        num   /= 16
        denom += 8
    end
    return fpart(s)
end

pi_digit(n) =
    floor(Int, 16 * fpart(4Σ(n-1, 1) - 2Σ(n-1, 4) - Σ(n-1, 5) - Σ(n-1, 6)))

pi_string(n) = "0x3." * join(string.(pi_digit.(1:n), base = 16)) * "p0"
{% endhighlight %}

The `pi_digit` function gives the $$n$$-th hexadecimal fractional digit of
$$\pi$$ as a base-10 integer, and the `pi_string` function returns the first
$$n$$ hexadecimal digits of $$\pi$$ as a valid hexadecimal floating-point
literal:

```julia
julia> pi_digit(1)
2

julia> pi_digit(6)
10

julia> pi_string(1000)
"0x3.243f6a8885a308d313198a2e03707344a4093822299f31d0082efa98ec4e6c89452821e638d01377be5466cf34e90c6cc0ac29b7c97c50dd3f84d5b5b54709179216d5d98979fb1bd1310ba698dfb5ac2ffd72dbd01adfb7b8e1afed6a267e96ba7c9045f12c7f9924a19947b3916cf70801f2e2858efc16636920d871574e69a458fea3f4933d7e0d95748f728eb658718bcd5882154aee7b54a41dc25a59b59c30d5392af26013c5d1b023286085f0ca417918b8db38ef8e79dcb0603a180e6c9e0e8bb01e8a3ed71577c1bd314b2778af2fda55605c60e65525f3aa55ab945748986263e8144055ca396a2aab10b6b4cc5c341141e8cea15486af7c72e993b3ee1411636fbc2a2ba9c55d741831f6ce5c3e169b87931eafd6ba336c24cf5c7a325381289586773b8f48986b4bb9afc4bfe81b6628219361d809ccfb21a991487cac605dec8032ef845d5de98575b1dc262302eb651b8823893e81d396acc50f6d6ff383f442392e0b4482a484200469c8f04a9e1f9b5e21c66842f6e96c9a670c9c61abd388f06a51a0d2d8542f68960fa728ab5133a36eef0b6c137a3be4ba3bf0507efb2a98a1f1651d39af017666ca593e82430e888cee8619456f9fb47d84a5c33b8b5ebee06f75d885c12073401a449f56c16aa64ed3aa62363f77061bfedf72429b023d37d0d724d00a1248db0fead3p0"
```

While I was preparing this post I found an unregistered
package [PiBBP.jl](https://github.com/kalvotom/PiBBP.jl) that implements the
Bailey–Borwein–Plouffe formula.  This is faster than my code above, mostly
thanks to a function
for
[modular exponentiation](https://en.wikipedia.org/wiki/Modular_exponentiation)
more efficient than that available in Julia standard library.

## Test results

Let’s check if the function is working correctly.  We can use the
[`parse`](https://docs.julialang.org/en/v1/base/numbers/#Base.parse) function to
convert the string to a decimal floating point number.  [IEEE
754](https://en.wikipedia.org/wiki/IEEE_754) double precision floating-point
numbers have a 53-bit mantissa, amounting to $$53 / \log_{2}(16) \approx 13$$
hexadecimal digits:

```julia
julia> pi_string(13)
"0x3.243f6a8885a30p0"

julia> parse(Float64, pi_string(13))
3.141592653589793

julia> Float64(π) == parse(Float64, pi_string(13))
true
```

[Generator expressions](https://docs.julialang.org/en/v1/manual/arrays/#Generator-Expressions-1) allow
us to obtain the decimal value of the number in a very simple way, without using
the hexadecimal string:

```julia
julia> 3 + sum(pi_digit(n)/16^n for n in 1:13)
3.141592653589793
```

We can use the
[arbitrary-precision](https://docs.julialang.org/en/v1/manual/integers-and-floating-point-numbers/#Arbitrary-Precision-Arithmetic-1)
`BigFloat` to check the correctness of the result for even more digits.  By
default, `BigFloat` numbers in Julia have a 256-bit mantissa:

```
julia> precision(BigFloat)
256
```

The result is correct for the first $$256 / \log_{2}(16) = 64$$ hexadecimal
digits:

```julia
julia> pi_string(64)
"0x3.243f6a8885a308d313198a2e03707344a4093822299f31d0082efa98ec4e6c89p0"

julia> BigFloat(π) == parse(BigFloat, pi_string(64))
true

julia> 3 + sum(pi_digit(n)/big(16)^n for n in 1:64)
3.141592653589793238462643383279502884197169399375105820974944592307816406286198
```

It’s possible to increase the precision of `BigFloat` numbers, to further test
the accuracy of the Bailey–Borwein–Plouffe formula:

```julia
julia> setprecision(BigFloat, 4000) do
           BigFloat(π) == parse(BigFloat, pi_string(1000))
       end
true
```

## Multi-threading

Since the Bailey–Borwein–Plouffe formula extracts the $$n$$-th digit of $$\pi$$
without computing the other ones, we can write a multi-threaded version of
`pi_string`, taking advantage of native support for
[multi-threading](https://docs.julialang.org/en/v1/manual/parallel-computing/#Multi-Threading-(Experimental)-1)
in Julia:

{% highlight julia linenos %}
function pi_string_threaded(N)
    digits = Vector{Int}(undef, N)
    Threads.@threads for n in eachindex(digits)
        digits[n] = pi_digit(n)
    end
    return "0x3." * join(string.(digits, base = 16)) * "p0"
end
{% endhighlight %}

For example, running Julia with 4 threads gives a 2× speed-up:

```julia
julia> Threads.nthreads()
4

julia> using BenchmarkTools

julia> pi_string(1000) == pi_string_threaded(1000)
true

julia> @benchmark pi_string(1000)
BenchmarkTools.Trial:
  memory estimate:  105.28 KiB
  allocs estimate:  2016
  --------------
  minimum time:     556.228 ms (0.00% GC)
  median time:      559.198 ms (0.00% GC)
  mean time:        559.579 ms (0.00% GC)
  maximum time:     564.502 ms (0.00% GC)
  --------------
  samples:          9
  evals/sample:     1

julia> @benchmark pi_string_threaded(1000)
BenchmarkTools.Trial:
  memory estimate:  113.25 KiB
  allocs estimate:  2018
  --------------
  minimum time:     270.577 ms (0.00% GC)
  median time:      271.075 ms (0.00% GC)
  mean time:        271.598 ms (0.00% GC)
  maximum time:     278.350 ms (0.00% GC)
  --------------
  samples:          19
  evals/sample:     1
```

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "american" -->
<!-- End: -->
