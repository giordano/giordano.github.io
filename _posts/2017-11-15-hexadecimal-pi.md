---
layout: post
title: Hexadecimal representation of pi in Julia
tags: draft julia math pi
---

```julia
# Return the fractional part of x, modulo 1, always positive
fpart(x) = mod(x, one(x))

function Σ(n, j)
    # Compute the finite sum
    s = 0.0
    for k in 0:n
        s = fpart(s + powermod(16, n - k, 8k + j) / (8k + j))
    end
    # Compute the infinite sum
    num = 1 / 16
    denom = 8(n + 1) + j
    frac = num / denom
    while frac > eps(Float64)
        s     += frac
        num   /= 16
        denom += 8
        frac   = num / denom
    end
    return fpart(s)
end

pi_digit(n) = floor(Int, 16 * fpart(4Σ(n, 1) - 2Σ(n, 4) - Σ(n, 5) - Σ(n, 6)))

pi_string(n) = "0x3." * join(hex.(pi_digit.(0:(n-1)))) * "p0"
```

```julia
julia> pi_string(1000)
"0x3.243f6a8885a308d313198a2e03707344a4093822299f31d0082efa98ec4e6c89452821e638d01377be5466cf34e90c6cc0ac29b7c97c50dd3f84d5b5b54709179216d5d98979fb1bd1310ba698dfb5ac2ffd72dbd01adfb7b8e1afed6a267e96ba7c9045f12c7f9924a19947b3916cf70801f2e2858efc16636920d871574e69a458fea3f4933d7e0d95748f728eb658718bcd5882154aee7b54a41dc25a59b59c30d5392af26013c5d1b023286085f0ca417918b8db38ef8e79dcb0603a180e6c9e0e8bb01e8a3ed71577c1bd314b2778af2fda55605c60e65525f3aa55ab945748986263e8144055ca396a2aab10b6b4cc5c341141e8cea15486af7c72e993b3ee1411636fbc2a2ba9c55d741831f6ce5c3e169b87931eafd6ba336c24cf5c7a325381289586773b8f48986b4bb9afc4bfe81b6628219361d809ccfb21a991487cac605dec8032ef845d5de98575b1dc262302eb651b8823893e81d396acc50f6d6ff383f442392e0b4482a484200469c8f04a9e1f9b5e21c66842f6e96c9a670c9c61abd388f06a51a0d2d8542f68960fa728ab5133a36eef0b6c137a3be4ba3bf0507efb2a98a1f1651d39af017666ca593e82430e888cee8619456f9fb47d84a5c33b8b5ebee06f75d885c12073401a449f56c16aa64ed3aa62363f77061bfedf72429b023d37d0d724d00a1248db0fead3p0"

julia> parse(BigFloat, pi_string(64)) - pi
0.000000000000000000000000000000000000000000000000000000000000000000000000000000
```

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "american" -->
<!-- End: -->
