---
layout: post
title: AstroImages.jl: package for visualization of astronomical images
tags: [draft, julia, astronomy, visualization, plotting]
---

In the last days I worked on a very little
package: [`AstroImages.jl`](https://github.com/JuliaAstro/AstroImages.jl).
Written in the [Julia programming language]() and part of
the [JuliaAstro organization](https://github.com/JuliaAstro), this package
provides simple tools to visualize astronomical images saved
in [FITS files](https://en.wikipedia.org/wiki/FITS).

`AstroImages.jl` is meant as an interface to popular Julia packages
like [`Images.jl`](https://github.com/JuliaImages/Images.jl)
and [`Plots.jl`](https://github.com/JuliaPlots/Plots.jl), and
uses [`FITSIO.jl`](https://github.com/JuliaAstro/FITSIO.jl) to read FITS files.

Currently the package supports only single-frame images, in gray scale.

`AstroImages.jl` is licensed under the MIT "Expat" License.

## Installation

The package is available for Julia 0.6 but not yet registered, thus if you would
like to try it out use the following command in the Julia REPL:

```julia
julia> Pkg.clone("https://github.com/JuliaAstro/AstroImages.jl")
```

## Usage

After installing the package, you can start using it with

```julia
julia> using AstroImages
```

For the time being the package provides only two functions:

* `load` (extending the `load` function
  from [`FileIO.jl`](https://github.com/JuliaIO/FileIO.jl)), to load an
  extension of a FITS file
* `AstroImage`, a type constructor to create a new astronomical image object.

### Reading extensions from FITS file

You can load and read the the first extension of a FITS file with `load`:

```julia
julia> load("file.fits")
1300×1200 Array{UInt16,2}:
[...]
```

The second, optional, argument of this `load` method is the number of the
extension to read.  Read the third extension of the file with:

```julia
julia> load("file.fits", 3)
1300×1200 Array{UInt16,2}:
[...]
```

### `AstroImage` type

The package provides a new type, `AstroImage` to integrate FITS images with
Julia packages for plotting and image processing. The `AstroImage` function has
the same syntax as `load`. This command:

```julia
julia> img = AstroImage("file.fits")
AstroImages.AstroImage{UInt16,ColorTypes.Gray}[...]
```

will read the first extension from the `file.fits`.

If you are working in a Jupyter notebook, an AstroImage object is automatically
rendered as a PNG image:

[![AstroImage in Jupyter](/img/astroimages1.png)](/img/astroimages1.png)

### Plotting an `AstroImage`

An `AstroImage` object can be plotted with `Plots.jl` package. Just use

```julia
julia> using Plots

julia> plot(img)
```

and the image will be displayed as a heatmap using your favorite backend.

[![AstroImage and Plots](/img/astroimages2.png)](/img/astroimages2.png)

## Feedback welcome

If you test the package and would like to suggest improvements, feel free to
file an issue on GitHub or, even better, send a pull request to add new features
yourself!

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "american" -->
<!-- End: -->
