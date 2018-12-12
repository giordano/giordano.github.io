---
layout: post
title: Cosmology.jl now integrates Unitful.jl
tags: [julia, juliaastro, cosmology, unitful, measurements]
---

[`Cosmology.jl`](https://github.com/JuliaAstro/Cosmology.jl) is the
[cosmology](https://en.wikipedia.org/wiki/Physical_cosmology) calculator for
Julia.  After you define a cosmological model, you can use it to perform some
operations like computation of angular diameter distance or comoving radial
distance at given redshift values, or calculate the age of the Universe in a
certain cosmological model at a specific redshift.

The package doesn’t have a full documentation, but the information in the
`README.md` file should be sufficient to get you started, if you’re already
familiar with the topic.

`Cosmology.jl` is a registered package, so you can install it in the Julia REPL
with the package manager:

```julia
pkg> add Cosmology
```

## New release

A few days ago version
[`v0.5.0`](https://github.com/JuliaAstro/Cosmology.jl/releases/tag/v0.5.0) has
been released.  The main new feature is the long-awaited integration with
[`Unitful.jl`](https://github.com/ajkeller34/Unitful.jl) and
[`UnitfulAstro.jl`](https://github.com/JuliaAstro/UnitfulAstro.jl).  This means
that the functions defined in this package now return a number with proper
physical units.  For example:

```julia
julia> using Cosmology

julia> c = cosmology(h=0.7, OmegaM=0.3, OmegaR=0, w0=-0.9, wa=0.1)
Cosmology.FlatWCDM{Float64}(0.7, 0.0, 0.7, 0.3, 0.0, -0.9, 0.1)

julia> comoving_radial_dist(c, 0.6)
2162.83342244173 Mpc
```

Previously, instead, the units were attached to the name of the function,
without possibility to easily change units.  These methods are now deprecated:

```julia
julia> comoving_radial_dist_mpc(c, 0.6)
┌ Warning: `comoving_radial_dist_mpc(c::AbstractCosmology, z; kws...)` is deprecated, use `ustrip(comoving_radial_dist(c::AbstractCosmology, z; kws...))` instead.
│   caller = top-level scope at none:0
└ @ Core none:0
2162.83342244173
```

In addition, now you can select a different unit for the output by simply
passing the desired output unit as first argument:

```
julia> using Unitful

julia> comoving_radial_dist(u"ly", c, 0.6)
7.054219146683016e9 ly
```

## Integration with Measurements.jl

`Cosmology.jl` is one of the several examples of how easy is in Julia to combine
together structures from different and independent packages.  Thanks to Julia’s
type system:

* the data structure used in this package to represent a cosmological model and
  the units data structure in `Unitful.jl` don’t know anything about each other
  (in `Cosmology.jl` the numbers are just multiplied by the appropriate unit, in
  the most natural way possible),
* units in `Unitful` and the `Measurement` structure in
  [`Measurements.jl`](https://github.com/JuliaPhysics/Measurements.jl) don’t
  know anything about each other,
* by now you can already tell that `Measurement` and the data structure for
  cosmological models don’t know anything about each other,

yet, we can use numbers with uncertainties as parameters of cosmological models
or for the redshift.  This enable use to propagate the uncertainties through the
operations performed by `Cosmology.jl` getting the results with the appropriate
physical units.

As an example, we can define a cosmological model using the parameters
[determined](https://arxiv.org/abs/1502.01589) by the [Planck
collaboration](https://en.wikipedia.org/wiki/Planck_(spacecraft)) in 2015:

```
julia> using Cosmology, Measurements

julia> planck2015 = cosmology(h=0.6774±0.0046, OmegaM=0.3089±0.0062, Tcmb=2.718±0.021, Neff=3.04±0.33)
Cosmology.FlatLCDM{Measurement{Float64}}(0.6774 ± 0.0046, 0.6910099044166828 ± 0.006200002032729847, 0.3089 ± 0.0062, 9.009558331733912e-5 ± 5.020543222494152e-6)
```

Now we calculate the comoving volume at redshift z = 3.62±0.04:

```julia
julia> z = 3.62 ± 0.04
3.62 ± 0.04

julia> cv = comoving_volume(planck2015, z)
1471.004984671155 ± 45.25029615715926 Gpc^3
```

`Measurements.jl` provides a utility,
[`Measurements.uncertainty_components`](https://juliaphysics.github.io/Measurements.jl/stable/usage.html#Uncertainty-Contribution-1),
to determine the contribution to the total uncertainty of a quantity:

```julia
julia> Measurements.uncertainty_components(cv.val)
Dict{Tuple{Float64,Float64,UInt64},Float64} with 5 entries:
  (3.04, 0.33, 0x0000000000000005)     => 0.0499315
  (0.3089, 0.0062, 0x0000000000000003) => 27.5208
  (3.62, 0.04, 0x0000000000000006)     => 19.8259
  (0.6774, 0.0046, 0x0000000000000002) => 29.952
  (2.718, 0.021, 0x0000000000000004)   => 0.0348057
```

This means that the major contributions to the uncertainty of `cv` comes from
the Hubble parameter (`0.6774±0.0046`) and the matter density (`0.3089±0.0062`).

We can also compute, in this cosmological model, the age of the Universe today
and at redshift z:

```julia
julia> age(planck2015, 0) # Age of the Universe today
13.79748128449975 ± 0.12164948254546123 Gyr

julia> age(planck2015, 42) # Age of the Universe at redshift z = 1.42
1.7337397190605572 ± 0.03053416280033588 Gyr
```

`lookback_time` gives the difference between age at redshift 0 and age at
redshift z:

```julia
julia> lookback_time(planck2015, 1.42)
12.063741565462488 ± 0.10426342852851997 Gyr
```

Note that uncertainties are always propagated taking care of the correlation
between quantities.  Indeed, we can check that the sum of the lookback time at
redshift z and the age of the Universe at the same redshift is equal to the age
of the Universe today, up to numerical errors of the integrals involved in the
calculations:

```julia
julia> lookback_time(planck2015, 1.42) + age(planck2015, 1.42)
13.797481284523045 ± 0.12164948254269693 Gyr
```

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "american" -->
<!-- End: -->
