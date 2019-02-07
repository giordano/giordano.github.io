---
layout: post
title: Julia do-block vs Python with statement
tags: [julia, python, control flow, draft]
---

The `do`-block in the Julia programming language and the `with` statement in
Python can appear to be similar because of some common uses, but are
fundamentally different.  Here we will review their purposes.

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

The `__enter__` method prepare the object for later use and returns `thing`,
which can be then consumed in the body of `with`.  The `__exit__` method will be
called in any case at the end of the body of `with` and can be used to close the
previously opened resources and handle any exception raised within the `with`
body, if necessary.

A common application of the `with` statement is the [`open`
function](https://docs.python.org/3/library/functions.html#open):

```python
with open(filename, mode='w') as f:
    do something with f
```

Just to practically see how the `__enter__` and `__exit__` methods can be
defined, we implement a very simple class, `MyFile`, to replicate the behaviour
of the builtin `open` function:

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

with MyOpen('foo.py') as f:
    f.read()
```

To summarise, the `with` statement in Python is a syntactic sugar for
`try`/`finally`.  The `__enter__`/`__exit__` methods are bound to a specific
class and they can be defined only once, so you can't have multiple
`__enter__`/`__exit__` handlers for the same class, or independently from a
class.  Of course, if you don't really need to control the flow like
`try`/`finally` you can still make use of the handy `with` syntax, for example
by making the `__exit__` method dummy.
