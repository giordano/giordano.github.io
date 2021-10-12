---
layout: post
title: "Mythbusting Julia speed"
tags: [draft, julia, performance]
---

Question: _**I read a lot on the Internet about this programming language called Julia being
very fast, but I'm very skeptic.  What do you say?**_

Answer: It's good to be skeptic, but you can try yourself!

Q: _**Some time ago I actually tried and I found it to be very slow.  See, I wrote this
small script, but it took almost a couple of seconds to do very simple things...**_

A: Right.  There are some things to keep in mind:

1. Julia has a non-negligible start-up time.  It isn't that bad, it's generally on the order
   of 0.4 seconds or so, but if your workflow consists on running several times very small
   scripts, you're going to pay this cost _every_ time you start Julia.  If this is a
   problem, you can either change your workflow to run a single long-running script or just
   don't use Julia.  If you want to repeat very small tasks multiple times driving them from
   the shell maybe Julia isn't what you need to use and that's OK.  However keep in mind
   that Julia is a perfectly fine [language for
   scripting](https://julialang.org/blog/2012/03/shelling-out-sucks/), so you could also
   consider using Julia to drive your scripts, which would also allow you not to start Julia
   multiple times.
2. Julia is a [Just-In-Time](https://en.wikipedia.org/wiki/Just-in-time_compilation) (JIT)
   compiler, or maybe a Just-Ahead-Of-Time one.  This means that in each Julia session, the
   first time you run a function (or a method of a function, to be more precise) it needs to
   be compiled, unless its compiled version was saved in the
   [sysimage](https://en.wikipedia.org/wiki/System_image).  Again, compilation time isn't
   that bad, usually in the order of ~0.1 seconds for most simple functions, but more
   complicated functions can take longer.  The problem however is that this time adds up,
   and it adds to the start-up time above.
3. It's very easy to make mistakes that very... I mean _**very**_ negatively affect
   performance of Julia code.  The most common gotcha is using global variables.  Global
   variables are usually considered bad practice in many compiled languages, in Julia
   non-constant variables completely destroy performance.  Avoid them and reorganise the
   code to use functions, or make the global variable constant, if that's really a constant
   that has to be accessed by multiple functions.  See [Julia's performance
   tips](https://docs.julialang.org/en/v1/manual/performance-tips/) for more gotchas.  While
   it seems daunting to remember all of them, it takes little practice to become familiar.
   Also, you can gradually learn them, you don't need to get everything right from the
   beginning, you'll learn them by making mistakes ;-)

To summarise, Julia gives its best in long-running sessions so that start-up and JIT
compilation costs are paid as few as possible.  There are some tools to make it easier to
restart Julia less frequently, or decrease the JIT cost:

* [`Revise.jl`](https://github.com/timholy/Revise.jl) automatically recompiles methods that
  are modified in the source code of packages or scripts.  When developing packages, this
  tool lets you seamlessly hot-reload the code and not restart Julia all the time.  Thanks
  to Julia's introspection and reflection capabilities, this works granularly and recompiles
  only the needed methods.  Note that when you do restart Julia, the package you edited will
  be entirely recompiled anyways.  One more reason to avoid restarting Julia multiple times
  :-)
* If you're an Emacs user, the need to restart a program as little as possible may sound
  familiar.  In that case a solution is to start an Emacs server and connect new clients to
  it.  It isn't a surprise there is a similar tool also for Julia:
  [`DaemonMode.jl`](https://github.com/dmolina/DaemonMode.jl).  This package is very useful
  if you insinst on a workflow based on starting Julia multiple times to run small scripts:
  with the server-clients model, you pay the cost of startup time only once.
* With [`PackageCompiler.jl`](https://github.com/JuliaLang/PackageCompiler.jl) you can
  create custom sysimages.  A common use-case is to fully compile a stable package that
  you're planning to use very frequently: JIT cost will be drastically reduced, the drawback
  is an increased size of the sysimage on disk.

Q: _**I've seen the [micro-benchmarks](https://julialang.org/benchmarks/) on Julia's
website.  They seem too good to be real!  And the set of benchmarks is very arbitrary to
make Julia look good**_

A: Let's make it clear.  Benchmarking is hard, languages benchmarks even more so: language
specifications can allow better or worse optimisations, but to get real numbers to compare
you usually use specific implementations (for C/C++ think of GCC vs LLVM vs Intel compilers
vs you-name-it; there are many Python implementations with different focus on speed), which
can have very different performance on different hardware.  At that point, what are you
actually benchmarking?  Language specification?  Particular implementation?  Vendor
optimisations?  Hardware performance?

There is no reason to think Julia's micro-benchmarks are somewhat fake: the [source code is
available](https://github.com/JuliaLang/Microbenchmarks), anyone can check, improve, and run
it.  Julia is in the same ballpark as other compiled languages, like C, LuaJIT, Rust, Go and
Fortran, there is nothing special about it.  Just like all other compiled languages, Julia
timings don't include compilation time, to show the pure runtime performance of compiled
code.  I agree the set of these benchmarks is arbitrary, any set would be, it's impossible
to cover all possible applications!  Actually, these benchmarks were chosen early in Julia's
development to guide optimisations to do in the language.  In a sense, Julia was optimised
to make these benchmarks look good.

Q: _**Well, plotting is so slow in Julia!**_

A: That's a fair point.  Honestly, plotting doesn't _have to_ be slow in Julia.  Some simple
or low-level plotting packages are actually fairly fast.  However, one of the most popular
packages for plotting, `Plots.jl` tends to feel slow to someone used to plotting in R or
Python.  This happens because `Plots.jl` has an interesting interface with which you can use
the same code to produce plots using different backends, but this means that some code paths
are lazily compiled on-the-fly when a new backend is enabled, including the default one.
This problem used to be very annoying, 2-3 years ago the first time you'd do a plot in a
Julia session (loading the `Plots.jl` package and running the `plot` command) it'd take up
to ~20 seconds for the plot to appear, leading to the so-called "time-to-first-plot"
problem.  The situation today is much improved, with time-to-first-plot for `Plots.jl` in
the order of ~5 seconds on recent computers.  Definitely not as fast as you'd like, but much
more bearable than before, with the hope to see more small improvements along the way in the
future.  As already said, the issue can be made less tedious by restarting Julia as little
as possible, or using a custom sysimage which includes `Plots.jl`.

Q: _**I've heard claims of people getting some ridiculously large speed-ups by moving to
Julia, like 200x, is that true? Is that even possible?**_

A: To be fair, probably the original code was not so performant, but speeding it up would
have been complicated for the developer.  And maybe with Julia it was easier to use hardware
accelerators like GPUs.  Does that make the claim misleading?  Maybe so, because that's not
a 1-to-1 comparison, but again, 1-to-1 comparisons are hard to make in a way that is fair
for all languages.  Does that make the claim false?  The fact is that Julia allowed them to
more easily enable optimisations that'd have been harder to achieve otherwise with the same
amount of effort.  Isn't that the point of a language that aims to make you more productive?

Q: _**Then should I expect a similar speed up by moving to Julia?**_

A: Not really!  A more accurate answer would be that it depends.  If your C/C++/Fortran code
is already heavily optimised, maybe even with some assembly to squeeze out the last bits of
performance, or if you have a Python+NumPy code which follows best performance practices
(vectorised functions, no `for` loops, etc...) don't expect any sensible speedup by
translating it to Julia, if any speedup at all.  In this case moving to Julia may not be
even worth the effort, just to set the right expectations.  Actually, a naive literal
translation is likely to have fairly bad performance because what's idiomatic in a language
may not be so in a different language with different properties (for example, they may have
different memory layouts).  Julia isn't a magic wand, it's a compiler which generates
machine code, you can likely achieve the same machine code with many other languages.  The
difference usually lies in the amount of effort you need to put on it with the same
experience in each language, starting a project from scratch: my _totally biased_ claim is
that tends to favour Julia :-) On the other hand, if the problem you want to solve isn't
amenable to be vectorised, note there are no performance penalties in using `for` loops in
Julia, so in this case moving from a `for`-allergic language or package to Julia can give
you a nice boost.

If you're willing to move your code to Julia, the improvements you should expect are not
necessarily about speed: you will work in a portable language, and get access to a wide,
composable ecosystem of packages for numerical computing, with possibility to switch to use
hardware accelerators (like GPUs) with minimal effort.

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "british" -->
<!-- fill-column: 92 -->
<!-- End: -->
