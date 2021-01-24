---
layout: post
title: "Measuring the prevalence of documentation, testing and continuous integration in Julia packages"
tags: [julia, testing, documentation, ci, registry]
---

[Documentation](https://en.wikipedia.org/wiki/Software_documentation),
[testing](https://en.wikipedia.org/wiki/Software_testing), and [continuous
integration](https://en.wikipedia.org/wiki/Continuous_integration) (CI) are some
of the key ingredients to make software maintinable and reliable in the long
term.  During the first part of my (short) career in academia, when my daily
software tools were C, Fortran and IDL, I barely knew these practices (probably
didn't know continuous integration at all), and never applied them in practice.
Some time around 5 years ago I started my journey with the Julia programming
language, and I learned to value and use these principles.

I have the general feeling that Julia makes documentation and testing very
simple and lowers the entry barriers for newcomers.  Today, the Julia ecosystem
offers many tools about this:

* [`Documenter.jl`](https://github.com/JuliaDocs/Documenter.jl): a package to
  generate HTML and PDF versions of the documentation
* the [`Test`](https://docs.julialang.org/en/v1/stdlib/Test/) standard library
  provides very basic tools for testing, but it's also extremely simple to use:
  you don't have a good reason for not testing your code
* in addition to `Test`, there are third-party packages which offer more
  advanced testing frameworks.  Some of them are:
  * [`Jive.jl`](https://github.com/wookay/Jive.jl)
  * [`ReTest.jl`](https://github.com/JuliaTesting/ReTest.jl)
  * [`SafeTestsets.jl`](https://github.com/YingboMa/SafeTestsets.jl)
  * [`TestSetExtensions.jl`](https://github.com/ssfrr/TestSetExtensions.jl)
  * [`XUnit.jl`](https://github.com/RelationalAI-oss/XUnit.jl)
* packages like [`PkgTemplates.jl`](https://github.com/invenia/PkgTemplates.jl)
  and [`PkgSkeleton.jl`](https://github.com/tpapp/PkgSkeleton.jl) lets you
  easily generate a new package with minimal setup for documentation, testing,
  and continuous integration.

If you're looking for tips about testing workflow in Julia, see [these best
testing practices with
Julia](https://erik-engheim.medium.com/julia-v1-5-testing-best-practices-3ca8780e6336)
by Erik Engheim.

## Analysing Julia packages in the General registry

Is my feeling that Julia makes documentation and testing easy actually true?
Prompted by the recent [analysis of the prevalence of continuous integration in
JOSS](http://blog.jamiejquinn.com/analysing-ci-in-joss) made by my colleague
Jamie Quinn, I decided to look how Julia packages in the [General
registry](https://github.com/JuliaRegistries/General) fare with regard to
documentation, testing and continuous integration.  I quickly wrote a Julia
package, [`AnalyzeRegistry.jl`](https://github.com/giordano/AnalyzeRegistry.jl)
for this task: it clones all packages in the registry (only the last commit of
the default branch, to save time and bandwidth) and looks for some specific
files to decide whether a package uses documentation, testing and, continuous
integration.

The usage of the package is described in its
[`README.md`](https://github.com/giordano/AnalyzeRegistry.jl/blob/fe93b158f551cdb5849b48c03a8a656959749ed0/README.md)
(yes, I haven't set up proper documentation, yet!).  I ran the analysis with 8
threads, it took less than 30 minutes to analyse the entire General registry (I
was mainly limited by my Internet connection, using some threads less wouldn't
have changed much):
```
julia> using AnalyzeRegistry

julia> @time report = analyze(find_packages());
1567.008404 seconds (7.41 M allocations: 857.636 MiB, 0.01% gc time, 0.00% compilation time)
```

`analyze` returns a vector of a data structure, `Package`, which describes a
package:

* name
* URL of the git repository
* can the repository be cloned?  Some repositories have been deleted or made
  private, so it can be accessed from the public anymore
* does it have documentation?  This is the hardest criterion: I looked for the
  file `docs/make.jl`, or `doc/make.jl`, which is used to generate the
  documentation with `Documenter.jl`, but many packages may do something else,
  see more in the comments below
* does it have the `test/runtests.jl` file?  This is what the `Test` standard
  library uses to launch the tests
* does it use GitHub Actions?
* does it use Travis CI?
* does it use AppVeyor?
* does it use Cirrus CI?
* does it use Circle CI?
* does it use Drone CI?
* does it use Buildkite?
* does it use Azure Pipelines?
* does it use GitLab Pipeline?

## Report of the analysis on 2021-01-23

Before summarising my findings, I saved the [results of the
analysis](https://github.com/giordano/AnalyzeRegistry.jl/releases/download/report-2021-01-23/report-2021-01-23.jld2)
in a [JLD2](https://github.com/JuliaIO/JLD2.jl) archive, so that anyone can look
into them.

Now, some statistics:

* total number of packages: 4312 (note: I excluded [JLL
  packages](https://docs.binarybuilder.org/stable/jll/), which are automatically
  generated, and not very useful to measure whether humans follow best
  programming practices);
* packages hosted on GitHub: 4296
* packages hosted on GitLab: 16
* packages that could be cloned: 4287 (99.4% of the total).  The rest of
  percentages will refer to this number, since I could look for documentation
  and testing only if I could actually clone the repository of a package;
* packages using `Documenter.jl` to publish documentation: 1887 (44%)
* packages with testing: 4139 (96.5%)
* packages using any of the CI services below: 4105 (95.8%)
* packages using GitHub Actions: 3240 (75.6%)
* packages using Travis CI: 2512 (58.6%)
* packages using AppVeyor: 783 (18.3%)
* packages using Cirrus CI: 60 (1.4%)
* packages using Circle CI: 13 (0.3%)
* packages using Drone CI: 43 (1%)
* packages using Buildkite: 29 (0.7%)
* packages using Azure Pipelines: 7 (0.2%)
* packages using GitLab Pipeline: 85 (1.9%)

I ran the analysis on 2020-01-23, on [this revision of the General
registry](https://github.com/JuliaRegistries/General/commit/824e39a809b2932f029866f5169930c0306c70fc).

## Comments

* The [General registry](https://github.com/JuliaRegistries/General) _is not_ a
  curated list of Julia packages and at the moment there is no requirement to
  have documentation, nor testing, nor continuous integration to be accepted.
  The question here is: do Julia users apply these principles even if they
  aren't mandatory?
* The results are biased by the criteria I've chosen to determine whether a
  package "has documentation", or "has tests" and should be taken _cum grano
  salis_: these criteria aren't bullet-proof, also an empty `test/runtests.jl`
  would count as "has tests", but see below.  If you change the criteria a bit
  you may find different numbers, but I hope I got the orders of magnitude
  right;
* the goal of this analysis was _not_ to measure the goodness of documentation
  or testing, but rather to see whether users apply these principles at all.
  It's also very difficult to accurately measure them.  [Line code
  coverage](https://en.wikipedia.org/wiki/Code_coverage) is often used to
  measure degree of testing of the code, but I don't think it's says the whole
  story: it's often easy to have 100% _lines_ coverage, but have a poor _paths_
  coverage (if you have many conditionals, loops, etc... you can hit the same
  lines from different directions), also, in very elaborated code-bases it may
  be good enough to have high coverage of the core of the software, while some
  other extra features may be less important to test, or may be too complicated
  to test because require sophisticated external setups;
* **about 44% of packages are set up to use `Documenter.jl` to publish
  documentation**.  While this fraction doesn't look particularly high, consider
  that many packages don't keep the documentation within the repository but it
  may be stored somewhere else (very notable examples are
  [`DifferentialEquations.jl`](https://github.com/SciML/DifferentialEquations.jl)
  and [`Plots.jl`](https://github.com/JuliaPlots/Plots.jl)), or they are so
  simple that the entire documentation is written down in the `README` file (for
  example [`Tar.jl`](https://github.com/JuliaIO/Tar.jl)) or the wiki.
  * I looked at a random sample of 43 packages (1% of total) "without
	documentation", of this
	* 4 didn't have any documentation, docstrings, or examples of uses
      whatsoever (but 2 of them still had meaningful tests!)
	* 1 didn't have documentation nor examples, but only docstrings
	* 1 only had an `examples` directory, with poorly commented samples of code
	* all the others had documentation in the `README`, in the wiki, or anyway
      published on a different website.

  To summarise, 6 packages out of 43 (14%) had a _very_ poor or completely
  missing meaningful documentation for users.  Assuming this was a
  representative sample, my conclusion is that about 8% of packages in the
  registry have a lacking documentation, while the **rest of the packages (92%)
  have some documentation to get users started**, and a bit less than half of
  the total are set up to use `Documenter.jl` to publish documentation;
* an overwhelming majority of packages, 96.5%, have tests!
  * packages like the above mentioned `PkgTemplates.jl` and `PkgSkeleton.jl`
	creates for you a `test/runtests.jl` file when you generate a new package,
	but this script doesn't actually contain any test at all, or just a dummy
	assertion like `@test 1 == 1`.  I looked at a random sample of 43 packages
	"with tests", to check whether they actually run some real tests, related to
	package: only two of them (4.7%) had a dummy `test/runtests.jl` file.  As
	already explained above, here I'm not judging the goodness of the tests or
	whether they cover enough code of the package, but most of the packages I've
	seen had very extensive test suites.  If the sample I analysed was
	representative of all Julia packages in the General registry, we can say
	that **about 92% of Julia packages do test their code**.  Interestingly
	enough, this is about the same fraction of packages with some documentation;
* **almost all packages with testing also set up continuous integration
  (95.8%)**, even though a small fraction is probably running dummy tests.  If
  compared to the about 80% of papers in JOSS found by Jamie not to have any
  obvious CI setup, the uptake of CI among Julia packages is remarkable (the
  audience of authors of JOSS paper and Julia packages has a large overlap, so I
  believe it makes sense to look at them together);
* GitHub Actions is the most used CI service (75.6%), followed by Travis
  (58.6%).  Note that measuring usage of GitHub Actions for CI is tricky,
  because, differently from all the other services, there isn't a single
  standard name for the configuration script, and GitHub Actions is also often
  used for other tasks than pure CI.  For example, I didn't consider files for
  [CompatHelper](https://github.com/JuliaRegistries/CompatHelper.jl/) or
  [TagBot](https://github.com/JuliaRegistries/TagBot).  It would have been
  interesting to look at these statistics before last November, when [the new
  Travis pricing
  model](https://blog.travis-ci.com/2020-11-02-travis-ci-new-billing) pushed
  many open-source users away from their services.

My take-away message is that despite the General registry not being curated and
not enforcing specific styles or best practices, **a large fraction of Julia
packages does embrace documentation, testing, and continuous integration**,
showing that the ecosystem provides easy-to-use tools for these practices.

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "british" -->
<!-- End: -->
