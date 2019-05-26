---
layout: post
title: "A case for multiple dispatch: complex division"
tags: [julia, draft, multiple dispatch, math]
---

Multiple dispatch is the ability to dispatch a function based on the number and
the run time types of its arguments.  The Julia programming language uses
[multiple dispatch](https://en.wikipedia.org/wiki/Multiple_dispatch) as its main
paradigm, and this has been a central design concept of the language from its
inception.

I already talked about multiple dispatch when I showed a simple implementation
of the [rock-paper-scissors game]({{ site.baseurl }}{% post_url 2017-11-03-rock-paper-scissors %}).
Multiple dispatch is useful not only to develop games, but also to write
mathematically correct code.

A key idea of multiple dispatch is that methods don’t belong to a single class,
but they are shared by all the arguments.  Quoting from [Julia
documentation](https://docs.julialang.org/en/v1/manual/methods/):

> does the addition operation in `x + y` belong to `x` any more than it does to
> `y`?  The implementation of a mathematical operator generally depends on the
> types of all of its arguments.  Even beyond mathematical operations, however,
> multiple dispatch ends up being a powerful and convenient paradigm for
> structuring and organizing programs.

An interesting example of how multiple dispatch allows one to more easily write
mathematically correct code is the division involving complex numbers.  In
particular, the corner case of division by 0.  But... is the 0 real or complex?
Indeed the answer is different depending on the nature of the 0.  When you
divide a complex finite number $$z$$ by (positive) real 0, you’re computing the
limit:

$$ \lim_{x \to 0^{+}} \dfrac{z}{x} $$

and it is clear the direction in which you take the limit: the real axis.  Thus,
for example, from the division $$(3 - 14i)/0$$ we should expect as a result
$$\infty - i\infty$$, assuming that the limit is taken from the positive part of
the real axis.  When the 0 is complex, instead, the direction of the limit in
the complex plane is not well-defined, and so the result of the division by a
complex zero should be undefined.  This behaviour is also mandated by the
[ISO/IEC 10967 standard](https://en.wikipedia.org/wiki/ISO/IEC_10967) (part 3,
section 5.2.5.5 "Fundamental complex floating point arithmetic").

## Comparison between languages

Let’s see how different programming languages behave with this operation.

### Python

```python
>>> import numpy as np
>>> np.complex128(1) / 0
__main__:1: RuntimeWarning: divide by zero encountered in cdouble_scalars
__main__:1: RuntimeWarning: invalid value encountered in cdouble_scalars
(inf+nanj) # This is correct
>>> np.float64(1) / np.complex128(0)
__main__:1: RuntimeWarning: divide by zero encountered in true_divide
__main__:1: RuntimeWarning: invalid value encountered in true_divide
(inf+nanj) # Should be (nan+nanj)
>>> np.complex128(1) / np.complex128(0)
(inf+nanj) # Should be (nan+nanj)
>>> np.float64(1) / np.complex128(1)
(1+0j)     # Should be (1-0j)
```

### R

```r
> complex(real = 1.0) / 0.0
[1] Inf+NaNi # This is correct
> 1.0 / complex(real = 0.0)
[1] Inf+NaNi # Should be NaN+NaNi
> complex(real = 1.0) / complex(real = 0.0)
[1] Inf+NaNi # Should be NaN+NaNi
> 1.0 / complex(real = 1.0)
[1] 1+0i     # Should be 1-0i
```

### GNU Octave

```matlab
octave:1> (1.0 + 0.0*i) / 0.0
warning: division by zero
ans =  Inf
octave:2> 1.0 / (0.0 + 0.0*i)
warning: division by zero
ans =  Inf
octave:3> (1.0 + 0.0*i) / (0.0 + 0.0*i)
warning: division by zero
ans =  Inf
octave:4> 1.0 / (1.0 + 0.0*i)
ans =  1
```

### C

```c
#include <complex.h>
#include <stdio.h>
int main(void)
{
  double complex c_zero = 0.0 + 0.0*I;
  double complex c_one  = 1.0 + 0.0*I;
  double complex w1 = c_one / 0.0;
  double complex w2 = 1.0 / c_zero;
  double complex w3 = c_one / c_zero;
  double complex w4 = 1.0 / c_one;
  printf("(%g,%g) # Should be (inf,nan)\n", creal(w1), cimag(w1));
  printf("(%g,%g) # Should be (nan,nan)\n", creal(w2), cimag(w2));
  printf("(%g,%g) # Should be (nan,nan)\n", creal(w3), cimag(w3));
  printf("(%g,%g)      # Should be (1,-0)\n", creal(w4), cimag(w4));
  return 0;
}
```

#### Compiled with GCC

```
$ gcc -std=c99 test.c&&./a.out
(inf,-nan) # Should be (inf,nan)
(inf,-nan) # Should be (nan,nan)
(inf,-nan) # Should be (nan,nan)
(1,0)      # Should be (1,-0)
```

#### Compiled with ICC

```
$ icc -std=c99 test.c&&./a.out
test.c(7): warning #39: division by zero
    double complex w1 = c_one / 0.0;
                                ^

(inf,-nan) # Should be (inf,nan)
(inf,-nan) # Should be (nan,nan)
(inf,-nan) # Should be (nan,nan)
(1,0)      # Should be (1,-0)
```

### C++

```cpp
#include <complex>
#include <iostream>

int main(void)
{
  std::complex<double> c_zero(0.0, 0.0);
  std::complex<double> c_one(1.0, 0.0);
  std::complex<double> w1 = c_one / 0.0;
  std::complex<double> w2 = 1.0 / c_zero;
  std::complex<double> w3 = c_one / c_zero;
  std::complex<double> w4 = 1.0 / c_one;
  std::cout << w1 << " # Should be (inf,nan)"   << std::endl;
  std::cout << w2 << " # Should be (nan,nan)"   << std::endl;
  std::cout << w3 << " # Should be (nan,nan)"   << std::endl;
  std::cout << w4 << "      # Should be (1,-0)" << std::endl;
  return 0;
}
```

#### Compiled with GCC

```
$ g++ test.cpp&&./a.out
(inf,-nan) # Should be (inf,nan)
(inf,-nan) # Should be (nan,nan)
(inf,-nan) # Should be (nan,nan)
(1,0)      # Should be (1,-0)
```

#### Compiled with ICC

```
$ icc test.cpp&&./a.out
(inf,-nan) # Should be (inf,nan)
(inf,-nan) # Should be (nan,nan)
(inf,-nan) # Should be (nan,nan)
(1,0)      # Should be (1,-0)
```

### Fortran90

```fortran
program main
  implicit none
  complex :: c_zero = (0.0, 0.0)
  complex :: c_one  = (1.0, 0.0)
  double precision :: f_zero = 0.0
  double precision :: f_one  = 1.0
  write (*, '(F0.0,"+i",F0.0,A)') c_one / f_zero,  " # Should be Inf+iNaN"
  write (*, '(F0.0,"+i",F0.0,A)') f_one / c_zero,  " # Correct"
  write (*, '(F0.0,"+i",F0.0,A)') c_one / c_zero,  " # Correct"
  write (*, '(F0.0,"+i",F0.0,A)') f_one / c_one, "   # Should be 1.-i0."
endprogram main
```

#### Compiled with Gfortran

```
$ gfortran test.f90&&./a.out
NaN+iNaN # Should be Inf+iNaN
NaN+iNaN # Should be NaN+iNaN
NaN+iNaN # Should be NaN+iNaN
1.+i0.   # Should be 1.-i0.
```

#### Compiled with ifort

```
$ ifort test.f90&&./a.out
Inf+iNaN # Should be Inf+iNaN
Inf+iNaN # Should be NaN+iNaN
Inf+iNaN # Should be NaN+iNaN
1.+i0.   # Should be 1.-i0.
```

### Julia

```julia
julia> complex(1) / 0
Inf + NaN*im # This is correct

julia> 1 / complex(0)
NaN + NaN*im # This is correct

julia> complex(1) / complex(0)
NaN + NaN*im # This is correct

julia> 1 / complex(1)
1.0 - 0.0im  # This is correct
```

## Summary

The difference between Julia and the other languages is that the division is
implemented in a different way depending on the type of the two operands.  By
using the
[`@which`](https://docs.julialang.org/en/v1/stdlib/InteractiveUtils/#InteractiveUtils.@which)
macro we can see which are the methods called in the examples above:
```julia
julia> @which complex(1) / 0
/(z::Complex, x::Real) in Base at complex.jl:324

julia> @which 1 / complex(0)
/(a::R, z::S) where {R<:Real, S<:Complex} in Base at complex.jl:323

julia> @which complex(1) / complex(0)
/(a::Complex{T}, b::Complex{T}) where T<:Real in Base at complex.jl:327
```
Thus each combination of real and complex arguments calls a different method.
This directly matches the different mathematical operations that should be
carried out.

The fact that in the other languages reviewed here we get the same result for
any combination of the types of the operands suggests that in those cases the
arguments are always silently converted to or treated as they were all complex,
thus losing the information about their original nature.

Multiple dispatch is nothing completely magic.  One could achieve similar
results in Python with a bunch of `if`s that check the types of the arguments,
but multiple dispatch keeps the different implementations neatly separated in
different methods, instead of putting the code for the different operations in a
single catch-em-all function.  Also in C++ one could define different methods
for different combinations of the type of the arguments, with the difference
that in that case the dispatch would be dynamic only on the first argument (the
class instance, the receiver of the method) and static on the others, whereas
with multiple dispatch the dispatch is based on the runtime type of *all* the
arguments.

*Note*: this post is largely inspired by the discussion in issue
[#22983](https://github.com/JuliaLang/julia/issues/22983) of the Julia
repository.

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "british" -->
<!-- End: -->
