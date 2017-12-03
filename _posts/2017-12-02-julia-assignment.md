---
layout: post
title: Returned value of assignment in Julia
tags: julia
---

Today I realized by chance that in
the [Julia programming language](https://julialang.org/), like in C/C++,
assignment returns the assigned value.

To be more precise, assignment returns the right-hand side of the assignment
expression.  This doesn’t appear to be documented in the current (as of December
2017) stable version of the manual, but
a
[FAQ](https://docs.julialang.org/en/latest/manual/faq/#What-is-the-return-value-of-an-assignment?-1) has
been added to its development version.  Be aware of this subtlety, because you
can get unexpected results if you rely on the type of the returned value of an
assignment:

```julia
julia> let a
         a = 3.0
       end
3.0

julia> let a
         a::Int = 3.0
       end
3.0
```

In both cases the `let` blocks return `3.0`, even though in the former case `a`
is really `3.0` and in the latter case it’s been marked to be an `Int`.

## Assignment in the REPL

The fact that assignment returns the assigned value is a very handy feature.
First of all, this is probably the reason why in the Julia’s REPL the assigned
value is printed after an assignment, so that you don’t have to type again the
name of the variable to view its value:

```julia
julia> x = sin(2.58) ^ 2 + cos(2.58) ^ 2
1.0
```

Instead in Python, for example, assignment returns `None`.  Thus, in the
interactive mode of the interpreter you have to type again the name of the
variable to check its value:

```python
>>> from math import *
>>> x = sin(2.58) ** 2 + cos(2.58) ** 2
>>> x
1.0
```

## Using assignment as condition

Another useful consequence of this feature is that you can use assignment as
condition in, e.g., `if` and `while`, making them more concise.  For example,
consider this brute-force method to compute
the [Euler’s number](https://en.wikipedia.org/wiki/E_(mathematical_constant))
stopping when the desired precision is reached:

```julia
euler = 0.0
i = 0
while true
    tmp = inv(factorial(i))
    euler += tmp
    tmp < eps(euler) && break
    i += 1
end
```

By using this feature the `while` can be simplified to:

```julia
euler = 0.0
i = 0
while (tmp = inv(factorial(i))) >= eps(euler)
    euler += tmp
    i += 1
end
```

### Can this feature be a problem in Julia?

A common objection against having this feature in Python is that it is
error-prone.  Quoting from
a [Python tutorial](https://docs.python.org/3/tutorial/datastructures.html):

> Note that in Python, unlike C, assignment cannot occur inside expressions. C
> programmers may grumble about this, but it avoids a common class of problems
> encountered in C programs: typing `=` in an expression when `==` was intended.

This is indeed an issue because in C and Python conditions can be pretty much
anything that can be converted to plain-bit 0 and 1, including, e.g.,
characters.

Consider the following C program:

```c
#include <stdio.h>

int main()
{
  char a = '\0';

  if (a = 0)
    printf("True\n");
  else
    printf("False\n");

  if (a == 0)
    printf("True\n");
  else
    printf("False\n");

  return 0;
}
```

which prints

```
False
True
```

Mistyping the condition (`=` instead of `==` or vice-versa) in this language can
lead to unexpected results.

Instead, this is mostly a non-issue in Julia because conditions can only be
instances of `Bool` type.  Therefore, problems may arise only when dealing with
`Bool` objects.  One way to make an error in Julia is the following:

```julia
julia> a = false
false

julia> if (a = false)
           println("True")
       else
           println("False")
       end
False

julia> if a == false
           println("True")
       else
           println("False")
       end
True
```

but it is kind of useless to test equality with a `Bool` as a condition, since
you can use the `Bool` itself.  The only other possible error you can make in
Julia I came up with is the following:

```julia
julia> a = false
false

julia> b = false
false

julia> if (a = b)
           println("True")
       else
           println("False")
       end
False

julia> if a == b
           println("True")
       else
           println("False")
       end
True
```

For this case I don’t have an easy replacement that could prevent you from an
error, but this is also an unlikely situation.  The Julia style guide suggests
to
[not parenthesize conditions](https://docs.julialang.org/en/stable/manual/style-guide/#Don%27t-parenthesize-conditions-1).
Following this recommendation is a good way to catch the error in this case
because the assignment requires parens:

```julia
julia> if a = b
           println("True")
       else
           println("False")
       end

ERROR: syntax: unexpected "="
```

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "american" -->
<!-- End: -->
