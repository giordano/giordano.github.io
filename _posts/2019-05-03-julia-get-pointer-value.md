---
layout: post
title: Getting the value of a pointer in Julia
tags: [julia, draft, pointers, c]
---

Yesterday a question in the [Julia’s Slack
channel](https://julialang.slack.com/) (request an invite
[here](https://slackinvite.julialang.org/)!) prompted a lively discussion about
how to deal with pointers in Julia.  As the history of the channel is ephemeral,
I wrote this post to keep note of some interesting points.

In the C world, the operation of getting the value of a pointer is called
[dereferencing](https://en.wikipedia.org/wiki/Dereference_operator).  When
dealing with C libraries in Julia, we may need to get the value of a pointer
into a Julia object.  The Julia manual is quite detailed about [accessing data
in
pointers](https://docs.julialang.org/en/v1/manual/calling-c-and-fortran-code/index.html#Accessing-Data-through-a-Pointer-1)
and I warmly recommend reading that first.

Note that, usually, in Julia it’s not possible to "dereference" a pointer in the
C-sense, because most of the operations we’re going to see will *copy* the data
into Julia structures.

[`unsafe_load`](https://docs.julialang.org/en/v1/base/c/#Base.unsafe_load) is
the generic function to access the value of a pointer and copy it to a Julia
object.  However, depending on the specific data types involved, there may be
different and more efficient solutions.  For example, you can obtain the value
of the pointer to an array with
[`unsafe_wrap`](https://docs.julialang.org/en/v1/base/c/#Base.unsafe_wrap-Union{Tuple{N},%20Tuple{T},%20Tuple{Union{Type{Array},%20Type{Array{T,N}%20where%20N},%20Type{Array{T,N}}},Ptr{T},Tuple{Vararg{Int64,N}}}}%20where%20N%20where%20T),
while for strings there is
[`unsafe_string`](https://docs.julialang.org/en/v1/base/strings/#Base.unsafe_string).

## Example 1: get the current time

Let’s have a look at a simple time-related example.

```julia
julia> result = Ref{Int64}(0)
Base.RefValue{Int64}(0)

julia> ccall((:time, "libc.so.6"), Int64, (Ptr{Int64},), result)
1556838715

julia> result
Base.RefValue{Int64}(1556838715)
```

By calling the `time` function from the C standard library, we have saved in
`result` the current number of seconds since the Unix epoch.  We now want to
turn this number into an instance of the C struct
[`tm`](https://en.cppreference.com/w/c/chrono/tm).  To do this, we can use the
`localtime` function.  However, if we want to later use the `tm` struct in Julia
we need to define a Julia mutable structure with the same layout as the C one
and define the `show` method to have a fancy output:

```julia
julia> mutable struct Ctm
           sec::Cint
           min::Cint
           hour::Cint
           mday::Cint
           mon::Cint
           year::Cint
           wday::Cint
           yday::Cint
           isdst::Cint
       end

julia> function Base.show(io::IO, t::Ctm)
           print(io, t.year + 1900, "-", lpad(t.mon, 2, "0"), "-",
                 lpad(t.mday, 2, "0"), "T", lpad(t.hour, 2, "0"), ":",
                 lpad(t.min, 2, "0"), ":", lpad(t.sec, 2, "0"))
       end
```

```julia
julia> localtime = ccall((:localtime, "libc.so.6"), Ptr{Ctm}, (Ptr{Int64},), result)
Ptr{Ctm} @0x00007f730fe9d300

julia> unsafe_load(localtime)
2019-04-03T00:11:55

julia> dump(unsafe_load(localtime))
Ctm
  sec: Int32 55
  min: Int32 11
  hour: Int32 0
  mday: Int32 3
  mon: Int32 4
  year: Int32 119
  wday: Int32 5
  yday: Int32 122
  isdst: Int32 1
```

Of course, if you want to deal with dates you can use the
[`Dates`](https://docs.julialang.org/en/v1/stdlib/Dates/) standard library
instead of playing with system calls.  Have also a look at the other
time-related functions in the [`Base.Libc`
module](https://github.com/JuliaLang/julia/blob/v1.1.0/base/libc.jl) of Julia.

## Example 2: copy from a data type to another one

Now that we’ve seen how to use `unsafe_load`, we can define the following
function to "dereference" a generic pointer to a Julia object (with the caveat
that we’re copying it’s value to the new object!):

```julia
julia> dereference(ptr::Ptr) = unsafe_load(ptr)
dereference (generic function with 1 method)

julia> dereference(T::DataType, ptr::Ptr) = unsafe_load(Ptr{T}(ptr))
dereference (generic function with 2 methods)
```

This `dereference` function has two arguments: the type that will wrap the data,
and the input pointer.  The first argument is optional, and defaults to the type
of the data pointed by `ptr`.  The first method is effectively an alias of the
plain `unsafe_load`.  The second method is interesting because it allows us to
copy the value of a pointer of a certain data type to an object of another type
with the same memory layout.

For example, let’s define a simple Julia mutable structure called `Bar`, get
an instance of it, and its pointer:

```julia
julia> mutable struct Bar
           x::UInt64
       end

julia> b = Bar(rand(UInt64))
Bar(0x55139e3fd61e43a4)

julia> pointer_from_objref(b)
Ptr{Nothing} @0x00007f1f72eb7850

julia> ptr = Ptr{Bar}(pointer_from_objref(b))
Ptr{Bar} @0x00007f1f72eb7850
```

[`pointer_from_objref`](https://docs.julialang.org/en/v1/base/c/#Base.pointer_from_objref)
gave us a pointer to `Nothing` (that is, the C `void`), we have then casted it
to a pointer to `Bar` and assigned this pointer to `ptr`.  We now define another
data structure with the same memory layout as `Bar` and use
`dereference` to "dereference" `ptr` as an instance of the new structure:

```julia
julia> mutable struct Foo
           a::UInt16
           b::UInt16
           c::UInt32
       end

julia> f = dereference(Foo, ptr)
Foo(0x43a4, 0xd61e, 0x55139e3f)
```

Note that

```julia
julia> UInt64(f.a) | UInt64(f.b) << 16 | UInt64(f.c) << 32
0x55139e3fd61e43a4

julia> b.x
0x55139e3fd61e43a4
```

as expected.

Remember that the term "dereference" is a stretch, because data is copied from
the pointer.  In fact:

```julia
julia> dereference(Foo, ptr) |> pointer_from_objref
Ptr{Nothing} @0x00007f1f72a62440

julia> dereference(Foo, ptr) |> pointer_from_objref
Ptr{Nothing} @0x00007f1f72fb3840

julia> dereference(Bar, ptr) |> pointer_from_objref
Ptr{Nothing} @0x00007f1f730054e0

julia> dereference(Bar, ptr) |> pointer_from_objref
Ptr{Nothing} @0x00007f1f73007340
```

## Conclusions

What we’ve seen here is nothing new for programmers already familiar with the C
language, but at the same time shows how easy and natural can be to communicate
from Julia with C programs.

## Special bonus

If you feel really bold and creative, you can even overload the `*` operator to
do *very* weird things like:

```julia
julia> Base.:*(ptr::Ptr) = dereference(ptr)

julia> Base.:*(T::DataType, ptr::Ptr) = dereference(T, ptr)

julia> *(ptr)
Bar(0x55139e3fd61e43a4)

julia> Foo *ptr
Foo(0x43a4, 0xd61e, 0x55139e3f)

julia> Bar *ptr
Bar(0x55139e3fd61e43a4)
```

However, the `*` operator shouldn’t be really used to mock C syntax is an
awkward way (we’re not actually dereferencing!), so don’t tell people that I
told you to use this!

<!-- Local Variables: --> 
<!-- ispell-local-dictionary: "british" --> 
<!-- End: -->
