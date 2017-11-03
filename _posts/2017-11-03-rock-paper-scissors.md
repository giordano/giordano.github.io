---
layout: post
title: Rock–paper–scissors game in less than 10 lines of code
tags: [julia, game, multiple dispatch]
---

[Rock–paper–scissors](https://en.wikipedia.org/wiki/Rock%E2%80%93paper%E2%80%93scissors) is
a popular hand game.  However, some nerds may prefer playing this game on their
computer rather than actually moving their hands.

[![Rock–paper–scissors rules](https://upload.wikimedia.org/wikipedia/commons/thumb/6/67/Rock-paper-scissors.svg/803px-Rock-paper-scissors.svg.png)](https://en.wikipedia.org/wiki/File:Rock-paper-scissors.svg)

_Image credit: [Enzoklop, Wikimedia Commons, CC-BY-SA 3.0](https://en.wikipedia.org/wiki/File:Rock-paper-scissors.svg)_

We can write this game in less than 10 lines of code in
the [Julia programming language](https://julialang.org/).  This implementation
will offer the opportunity to have a closer look to one of Julia’s main
features:
[multiple dispatch](https://docs.julialang.org/en/stable/manual/methods/#Methods-1).

## The game

Here we go:

{% highlight julia linenos %}
abstract type Shape end
struct Rock     <: Shape end
struct Paper    <: Shape end
struct Scissors <: Shape end
play(::Type{Paper}, ::Type{Rock})     = "Paper wins"
play(::Type{Paper}, ::Type{Scissors}) = "Scissors wins"
play(::Type{Rock},  ::Type{Scissors}) = "Rock wins"
play(::Type{T},     ::Type{T}) where {T<: Shape} = "Tie, try again"
play(a::Type{<:Shape}, b::Type{<:Shape}) = play(b, a) # Commutativity
{% endhighlight %}

That’s all.  Nine lines of code, as promised.  This is considerably shorter,
simpler, and easier to understand than any other implementation in all languages
over at [Rosetta Code](https://rosettacode.org/wiki/Rock-paper-scissors).

### Explanation

Let’s dissect the code.

```julia
abstract type Shape end
```

defines `Shape` as
an
[abstract type](https://docs.julialang.org/en/stable/manual/types/#Abstract-Types-1).
This will be the parent of the concrete types `Rock`, `Paper` and `Scissors`
that represent the characters of the game.  To be fair, it’s not necessary to
create the `Shape` abstract type (so they would be eight lines in total!), but
this allows us to define methods for the `play` function only with `Shape`
subtypes as arguments, so that one can extended that function to other games
without clashing with our definitions.

```julia
struct Rock     <: Shape end
struct Paper    <: Shape end
struct Scissors <: Shape end
```

Here the concrete shapes are defined
as
[composite types](https://docs.julialang.org/en/stable/manual/types/#Composite-Types-1),
subtypes of `Shape` (indicated by the `<:` sign).  They don’t actually contain
anything, but that’s OK, we just want to define the elements of the game as
types in order to take advantage of Julia’s type system.

```julia
play(::Type{Paper}, ::Type{Rock})     = "Paper wins"
play(::Type{Paper}, ::Type{Scissors}) = "Scissors wins"
play(::Type{Rock},  ::Type{Scissors}) = "Rock wins"
```

These are the basic rules of the game.  We’ve defined
three
[methods](https://docs.julialang.org/en/stable/manual/methods/#Defining-Methods-1) for
the `play` function, which return a string indicating the winning shape.  The
two arguments of these methods are two shapes, for instance `Rock` and
`Scissors`.  If you look carefully to the definitions, we omitted the names of
the arguments (they should come right before the `::` in the list of elements),
because they’re not used in the body of the function and we don’t need them.
Instead, what’s important here is the type of both arguments.

```julia
play(::Type{T}, ::Type{T}) where {T<: Shape} = "Tie, try again"
```

With this single line we’ve defined the tie for all shapes.  The arguments of
this method are two equal shapes, they have the same type `T` subtype of
`Shape`, whatever `T` is.  `T` doesn’t even need to be defined at this point,
because in Julia, dispatch is dynamic on all arguments (in in C++/Java you can
achieve dynamic dispatch on first argument, but it’ll be static for the others).

So far we’ve seen the rules, for example, for the arguments `Paper` and `Rock`,
in this order, but there is no rule for the same arguments in the reversed
order.  Here comes the magic:

```julia
play(a::Type{<:Shape}, b::Type{<:Shape}) = play(b, a)
```

Recall that [multiple dispatch](https://en.wikipedia.org/wiki/Multiple_dispatch)
is the ability to define function behavior across many combinations of argument
types.  What’s crucial here is that the type of all arguments matters, not just
the first one as in object-oriented programming languages.  If none of the
previous methods applies (`Paper`-`Rock`, `Paper`-`Scissors`, `Rock`-`Scissors`
and all the combinations of arguments with equal types), this method will be
used, which simply swaps the two arguments so that one of the above methods can
be called.

Besides letting us save four definitions, multiple dispatch makes the program
very efficient.  The appropriate method is quickly selected based on the type of
all the arguments.  Without multiple dispatch we’d have to write explicitly a
sequence of at least seven `if ... elseif ... end`,
but [branching](https://en.wikipedia.org/wiki/Branch_(computer_science)) comes
at a performance cost.  Of course this isn’t a big deal in a simple game like
this, but think about your CPU-intensive application.

## Play the game

Now that we’ve implemented the game we can play it in the
Julia
[REPL](https://docs.julialang.org/en/stable/manual/interacting-with-julia/#Interacting-With-Julia-1):

```julia
julia> abstract type Shape end

julia> struct Rock     <: Shape end

julia> struct Paper    <: Shape end

julia> struct Scissors <: Shape end

julia> play(::Type{Paper}, ::Type{Rock})     = "Paper wins"
play (generic function with 1 method)

julia> play(::Type{Paper}, ::Type{Scissors}) = "Scissors wins"
play (generic function with 2 methods)

julia> play(::Type{Rock},  ::Type{Scissors}) = "Rock wins"
play (generic function with 3 methods)

julia> play(::Type{T},     ::Type{T}) where {T<: Shape} = "Tie, try again"
play (generic function with 4 methods)

julia> play(a::Type{<:Shape}, b::Type{<:Shape}) = play(b, a) # Commutativity
play (generic function with 5 methods)

julia> play(Paper, Scissors)
"Scissors wins"

julia> play(Rock, Rock)
"Tie, try again"

julia> play(Rock, Paper)
"Paper wins"

julia> @which play(Rock, Paper)
play(a::Type{#s1} where #s1<:Shape, b::Type{#s2} where #s2<:Shape) in Main at REPL[9]:1
```

There was no explicit method for the combination of arguments `Rock`-`Paper`,
but the commutative rule has been used here, as confirmed by the
`@which`
[macro](https://docs.julialang.org/en/stable/manual/metaprogramming/#man-macros-1).

### Play randomly

We can also add some randomness to the game.
The
[`rand`](https://docs.julialang.org/en/stable/stdlib/numbers/#Base.Random.rand)
function can pick up a random element from a given collection.  Luckily,
the
[`subtypes`](https://docs.julialang.org/en/stable/stdlib/base/#Base.subtypes)
function returns the array with all the subtypes of the given abstract type:

```julia
julia> subtypes(Shape)
3-element Array{Union{DataType, UnionAll},1}:
 Paper
 Rock
 Scissors

julia> rand(subtypes(Shape))
Rock

julia> rand(subtypes(Shape))
Rock

julia> rand(subtypes(Shape))
Paper

julia> rand(subtypes(Shape))
Scissors

julia> play(rand(subtypes(Shape)), rand(subtypes(Shape)))
"Scissors wins"

julia> play(rand(subtypes(Shape)), rand(subtypes(Shape)))
"Paper wins"

julia> play(rand(subtypes(Shape)), rand(subtypes(Shape)))
"Paper wins"

julia> play(rand(subtypes(Shape)), rand(subtypes(Shape)))
"Tie, try again"

julia> play(rand(subtypes(Shape)), rand(subtypes(Shape)))
"Paper wins"
```

## Extend to four shapes

[Here](https://math.stackexchange.com/q/410558/80474) it is proposed an
extension of the game to a fourth shape, the well, with the following rules:

* the well wins against rock and scissors, because both fall into it,
* the well loses against paper, because the paper covers it.

This is a [non-zero-sum game](https://en.wikipedia.org/wiki/Zero-sum_game), but
this isn’t a concern for us.

We can extend the above Julia implementation to include the well.

{% highlight julia linenos %}
struct Well <: Shape end
play(::Type{Well}, ::Type{Rock})     = "Well wins"
play(::Type{Well}, ::Type{Scissors}) = "Well wins"
play(::Type{Well}, ::Type{Paper})    = "Paper wins"
{% endhighlight %}

We accomplished the extension with just four additional lines, one to define the
new type and three for the new game rules.  We don’t need to redefine the tie or
the commutativity methods, thanks to Julia’s dynamic type system.

Here you can see that multiple dispatch let us extend the game very easily,
saving us four more definitions.  Out of the total $$4 \times 4 = 16$$ necessary
definitions we had just $$8$$ definitions, without giving up commutativity.

Let’s see it in action in the REPL:

```julia
julia> struct Well <: Shape end

julia> play(::Type{Well}, ::Type{Rock})     = "Well wins"
play (generic function with 6 methods)

julia> play(::Type{Well}, ::Type{Scissors}) = "Well wins"
play (generic function with 7 methods)

julia> play(::Type{Well}, ::Type{Paper})    = "Paper wins"
play (generic function with 8 methods)

julia> play(Paper, Well)
"Paper wins"

julia> play(Well, Rock)
"Well wins"

julia> play(Well, Well)
"Tie, try again"
```

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "american" -->
<!-- End: -->
