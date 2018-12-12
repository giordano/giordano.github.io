---
layout: post
title: Cosmology.jl now integrates Unitful.jl
tags: [draft, julia, juliaastro, cosmology, unitful, measurements]
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
[`Unitful.jl`](https://github.com/ajkeller34/Unitful.jl).  This means that the
functions defined in this package now return a number with proper physical
units.  For example:

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
together structures from different and independent packages.  In fact, besides
having numbers with physical units, we can use numbers with uncertainties (from
[`Measurements.jl`](https://github.com/JuliaPhysics/Measurements.jl)) as
parameters of cosmological models or for the redshift.  This enable use to
propagate the uncertainties through the operations performed by `Cosmology.jl`.
This feature comes for free, there is no code in `Cosmology.jl` nor in
`Measurements.jl` to make all this work.

As an example, we can define a cosmological model using the parameters
[determined](https://arxiv.org/abs/1502.01589) by the [Planck
collaboration](https://en.wikipedia.org/wiki/Planck_(spacecraft)) in 2015:

```
julia> using Cosmology, Measurements

julia> planck2015 = cosmology(h=0.6774±0.0046, OmegaM=0.3089±0.0062, Tcmb=2.718±0.021, Neff=3.04±0.33)
Cosmology.FlatLCDM{Measurement{Float64}}(0.6774 ± 0.0046, 0.6910099044166828 ± 0.006200002032729847, 0.3089 ± 0.0062, 9.009558331733912e-5 ± 5.020543222494152e-6)
```

Now we calculate the comoving volume at redshift 3.6:

```julia
julia> cv = comoving_volume(planck2015, 3.6)
1461.0828166268652 ± 40.378303844048226 Gpc^3
```

`Measurements.jl` provides a utility,
[`Measurements.uncertainty_components`](https://juliaphysics.github.io/Measurements.jl/stable/usage.html#Uncertainty-Contribution-1),
to determine the contribution to the total uncertainty of a quantity:

```julia
julia> Measurements.uncertainty_components(cv.val)
Dict{Tuple{Float64,Float64,UInt64},Float64} with 4 entries:
  (3.04, 0.33, 0x0000000000000005)     => 0.0494192
  (0.3089, 0.0062, 0x0000000000000003) => 27.3009
  (0.6774, 0.0046, 0x0000000000000002) => 29.7501
  (2.718, 0.021, 0x0000000000000004)   => 0.0344486
```

This means that the major contributions to the uncertainty of `cv` comes from
the Hubble parameter (`0.6774±0.0046`) and the matter density (`0.3089±0.0062`).

We can also compute, in this cosmological model, the age of the Universe today
and at a certain redshift in the past:

```julia
julia> age(planck2015, 0) # Age of the Universe today
13.79748128449975 ± 0.12164948254546123 Gyr

julia> age(planck2015, 42) # Age of the Universe at redshift z = 1.42
4.481579852131797 ± 0.05168067053818648 Gyr
```

`lookback_time` gives the difference between age at redshift 0 and age at
redshift z:

```julia
julia> lookback_time(planck2015, 1.42)
9.315901432385681 ± 0.07270835053850357 Gyr
```

Note that uncertainties are always propagated taking care of the correlation
between quantities.  Indeed, we can check that the sum of the lookback time at
redshift z and the age of the Universe at the same redshift is equal to the age
of the Universe today, up to numerical errors of the integrals involved in the
calculations:

```julia
julia> lookback_time(planck2015, 1.42) + age(planck2015, 1.42)
13.797481284517477 ± 0.12164948254345108 Gyr
```

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "american" -->
<!-- End: -->
