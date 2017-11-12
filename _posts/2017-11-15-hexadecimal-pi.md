---
layout: post
title: Computing the hexadecimal value of pi
tags: draft julia math pi
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
it allows one to directly extract the $$n$$-th digit of the hexadecimal value of
$$\pi$$ without computing the preceding ones.

[![Digits of pi](https://cloud.githubusercontent.com/assets/52289/23835253/0277a45e-075c-11e7-9431-14b8613e15ce.png)](https://cloud.githubusercontent.com/assets/52289/23835253/0277a45e-075c-11e7-9431-14b8613e15ce.png)

_Image courtesy of [Cormullion](https://github.com/cormullion),
code
[here](https://gist.github.com/cormullion/e979d819e478da73280faaeb67490888)._

The Wikipedia article about the Bailey–Borwein–Plouffe formula explains that the
$$n$$-th digit $$d_{n}$$ is given by

$$ d_{n} = 16 \left[ 4 \Sigma(n, 1) - 2 \Sigma(n, 4) - \Sigma(n, 5) - \Sigma(n,
6) \right] $$

where

$$ \Sigma(n, j) = \sum_{k = 0}^{n} \frac{16^{n-k} \bmod (8k+j)}{8k+j} + \sum_{k
= n + 1}^{\infty} \frac{16^{n-k}}{8k+j} = \sum_{k = 0}^{n} \frac{16^{n-k} \bmod
(8k+j)}{8k+j} + \sum_{k = 1}^{\infty} \frac{16^{-k}}{8(n + k)+j} $$

Only the fractional part of expression in square brackets on the right side of
$$d_{n}$$ expression is relevant, thus, in order to avoid rounding errors, when
we compute each term of the finite sum above we can take only the fractional
part.  This allows us to always use ordinary double precision floating-point
arithmetic, without resorting to arbitrary-precision numbers.  In addition note
that the terms of the infinite sum get quickly very small, so we can stop the
summation when they become negligible.

Here is a [Julia](https://julialang.org/) implementation of the algorithm to
extract the $$n$$-th digit of $$\pi$$:

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
    frac = num / denom
    while frac > eps(s)
        s     += frac
        num   /= 16
        denom += 8
        frac   = num / denom
    end
    return fpart(s)
end

pi_digit(n) = floor(Int, 16 * fpart(4Σ(n, 1) - 2Σ(n, 4) - Σ(n, 5) - Σ(n, 6)))

pi_string(n) = "0x3." * join(hex.(pi_digit.(0:(n-1)))) * "p0"
{% endhighlight %}

The `pi_string` function returns the first $$n$$ hexadecimal digits of $$\pi$$
as a valid hexadecimal floating-point literal:

```julia
julia> pi_string(1000)
"0x3.243f6a8885a308d313198a2e03707344a4093822299f31d0082efa98ec4e6c89452821e638d01377be5466cf34e90c6cc0ac29b7c97c50dd3f84d5b5b54709179216d5d98979fb1bd1310ba698dfb5ac2ffd72dbd01adfb7b8e1afed6a267e96ba7c9045f12c7f9924a19947b3916cf70801f2e2858efc16636920d871574e69a458fea3f4933d7e0d95748f728eb658718bcd5882154aee7b54a41dc25a59b59c30d5392af26013c5d1b023286085f0ca417918b8db38ef8e79dcb0603a180e6c9e0e8bb01e8a3ed71577c1bd314b2778af2fda55605c60e65525f3aa55ab945748986263e8144055ca396a2aab10b6b4cc5c341141e8cea15486af7c72e993b3ee1411636fbc2a2ba9c55d741831f6ce5c3e169b87931eafd6ba336c24cf5c7a325381289586773b8f48986b4bb9afc4bfe81b6628219361d809ccfb21a991487cac605dec8032ef845d5de98575b1dc262302eb651b8823893e81d396acc50f6d6ff383f442392e0b4482a484200469c8f04a9e1f9b5e21c66842f6e96c9a670c9c61abd388f06a51a0d2d8542f68960fa728ab5133a36eef0b6c137a3be4ba3bf0507efb2a98a1f1651d39af017666ca593e82430e888cee8619456f9fb47d84a5c33b8b5ebee06f75d885c12073401a449f56c16aa64ed3aa62363f77061bfedf72429b023d37d0d724d00a1248db0fead3p0"
```

Let’s check if the function is working correctly.  We can use
the
[`parse`](https://docs.julialang.org/en/stable/stdlib/numbers/#Base.parse-Tuple{Type,Any,Any}) function
to covert the string to a decimal floating point
number.  [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754) double precision
floating-point numbers have a 53-bit mantissa, amounting to about $$53 /
\log_{2}(16) \approx 13$$ hexadecimal digits:

```julia
julia> pi_string(13)
"0x3.243f6a8885a30p0"

julia> parse(Float64, pi_string(13))
3.141592653589793

julia> parse(Float64, pi_string(13)) - π
0.0
```

We can go use the
arbitrary-precision
[`BigFloat`](https://docs.julialang.org/en/stable/manual/integers-and-floating-point-numbers/#Arbitrary-Precision-Arithmetic-1) to
check the correctness of the result for even more digits.  By default,
`BigFloat` numbers in Julia have a 256-bit mantissa:

```
julia> precision(BigFloat)
256
```

The result is correct for the first $$256 / \log_{2}(16) = 64$$ hexadecimal
digits:

```julia
julia> pi_string(64)
"0x3.243f6a8885a308d313198a2e03707344a4093822299f31d0082efa98ec4e6c89p0"

julia> parse(BigFloat, pi_string(64)) - π
0.000000000000000000000000000000000000000000000000000000000000000000000000000000
```

It’s possible to increase the precision of `BigFloat` numbers, to further test
the accuracy of the Bailey–Borwein–Plouffe formula:

```julia
julia> setprecision(BigFloat, 4000) do
           parse(BigFloat, pi_string(1000)) - π
       end
0.00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "american" -->
<!-- End: -->
