---
layout: post
title: Julia do-block vs Python with statement
tags: [julia, python, control flow, draft]
---

The `do`-block in the Julia programming language and the `with` statement in
Python can appear to be similar because of some common uses (most notably,
opening a stream and automatically closing it at the end), but are fundamentally
different.  Here we will review their purposes.

## `with` statement in Python

The [`with`
statement](https://docs.python.org/3/reference/compound_stmts.html#with) has
been introduced in Python with [PEP
343](https://www.python.org/dev/peps/pep-0343/) to simplify the control flow of
[`try`/`finally`
statements](https://docs.python.org/3/reference/compound_stmts.html#try).

A good explanation of this feature is given at
<http://effbot.org/zone/python-with-statement.htm>.  In general, you can use
`try`/`finally` to handle unmanaged resources like stream:

```python
def controlled_execution(callback):
    set things up
    try:
        callback(thing)
    finally:
        tear things down

def my_function(thing):
    do something with thing

controlled_execution(my_function)
```

For an object of a class, it is possible to reorganize the above control flow by
using the `with` statement:

```python
class controlled_execution:
    def __enter__(self):
        set things up
        return thing
    def __exit__(self, type, value, traceback):
        tear things down

with controlled_execution() as thing:
    do something with thing
```

The `__enter__` method prepares the object for later use and returns `thing`,
which can be then consumed in the body of `with`.  The `__exit__` method will be
called in any case at the end of the body of `with` and can be used to close the
previously opened resources and handle any exception raised within the `with`
body, if necessary.

### Opening a file with `with`

A common application of the `with` statement is the [`open`
function](https://docs.python.org/3/library/functions.html#open):

```python
with open(filename, mode='w') as f:
    do something with f
```

Just to practically see how the `__enter__` and `__exit__` methods can be
defined, we implement a very simple class, `MyFile`, to replicate the behaviour
of the builtin `open` function when opening a file:

```python
class MyFile:
    def __init__(self, file):
        self.file = file
    def __enter__(self):
        return self.file
    def __exit__(self, type, value, traceback):
        return self.file.close()
    def read(self):
        return self.file.read()

def MyOpen(name, mode='r'):
    return MyFile(open(name, mode))

with MyOpen('file.txt') as f:
    f.read()
```

## `do`-block in Julia

The
[`do`-block](https://docs.julialang.org/en/v1/manual/functions/#Do-Block-Syntax-for-Function-Arguments-1)
in Julia aims to address a different problem: write in a different way the
passing a function as argument to another function.

In a simple way, any function `func` accepting a function as first argument:

```julia
function func(f::Function, x, y, z...)
    # do something with arguments x, y, z..., and call f at some point
end
```

can be called as

```julia
func(x, y, z...) do args
    # put here the body of function f(args) that will
    # will be passed as first argument to func
end
```

For example, the
[`filter`](https://docs.julialang.org/en/v1/base/collections/#Base.filter)
enables filtering a collection by removing those elements for which the provided
function returns `false`.  For example, we can select the odd numbers between 1
and 100 divisible by 3 with

```julia
julia> filter(x -> isinteger(x/3) && isodd(x), 1:100)
17-element Array{Int64,1}:
  3
  9
 15
 21
 27
 33
 39
 45
 51
 57
 63
 69
 75
 81
 87
 93
 99
```

Here we passed an [anonymous
function](https://docs.julialang.org/en/v1/manual/functions/#man-anonymous-functions-1)
as first argument to `filter` in order to select the desired elements of the
collection passed as second argument.  We can rewrite this by using the
`do`-block in the following way:

```julia
julia> filter(1:100) do x
           isinteger(x/3) && isodd(x)
       end
17-element Array{Int64,1}:
  3
  9
 15
 21
 27
 33
 39
 45
 51
 57
 63
 69
 75
 81
 87
 93
 99
```

The code in the body of the `do`-block is used to create an anonymous function,
whose arguments are specified after `do`, that is passed as first argument to
the function before the `do`.

### Opening a file with `do`

A common use of the `do`-block is to open a file and automatically close it when
you’re done, just like the `with` statement in Python.  For example:

```julia
open("outfile", "w") do io
    write(io, data)
end
```

In Julia’s Base code, this is achieved by extending the [`open`
method](https://docs.julialang.org/en/v1/base/io-network/#Base.open) in the
following way:

```julia
function open(f::Function, args...; kwargs...)
    io = open(args...; kwargs...)
    try
        f(io)
    finally
        close(io)
    end
end
```

This method, which accepts a function as first argument, is what makes it
possible to use the `do`-block in the example above: after effectively opening
the file, the body of the `do`-block is executed in the [`try`
statement](https://docs.julialang.org/en/v1/manual/control-flow/#The-try/catch-statement-1),
and in the end the [`finally`
clause](https://docs.julialang.org/en/v1/manual/control-flow/#finally-Clauses-1)
takes care of safely closing the file, even in case of errors while executing
the function `f`.

## Summary

The `with` statement in Python is a syntactic sugar for `try`/`finally`.  This
construct is bound to classes, this means that the `__enter__`/`__exit__`
methods are bound to a specific class and they can be defined only once, so you
can't have multiple `__enter__`/`__exit__` handlers for the same class, or
independently from a class.  In order for the `with` statement to work, both
`__enter__` and `__exit__` must be defined in any case, however if you don't
really need the `try`/`finally` construct you can still take advantage of the
handy `with` syntax, for example by making the `__exit__` method dummy.

The `do`-block in Julia is not bound to any type, the only condition to use it
is to have a function taking a function as first argument.  The function may
contain a control flow construct, like `try`/`finally`, but this is not
mandatory.  You can also have different methods for a function to be used with
the `do`-block, depending on the types and/or the number of the other arguments.
