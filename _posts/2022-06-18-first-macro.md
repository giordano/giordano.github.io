---
layout: post
title: "My first macro in Julia"
tags: [julia, draft, macro, metaprogramming]
---

This post isn't about the first macro I wrote in the [Julia programming
language](https://julialang.org/), but it can be about _your_ first macro.

## Frequently Asked Questions

_Question_: _**What are macros in Julia?**_

_Answer_: Macros are sort of functions which take as input _unevaluated_ expressions
([`Expr`](https://docs.julialang.org/en/v1/base/base/#Core.Expr)) and return as output
another expression, whose code is then regularly evaluated at runtime.  This post isn't a
substitute for reading the [section about macros in the Julia
documentation](https://docs.julialang.org/en/v1/manual/metaprogramming/#man-macros), it's
more complementary to it, a light tutorial for newcomers, but I warmly suggest reading the
manual to learn more about them.

Calls to macros are different from calls to regular functions because you need to prepend
the name of the macro with the at-sign `@` sign: `@view A[4, 1:5]` is a call to the
[`@view`](https://docs.julialang.org/en/v1/base/arrays/#Base.@view) macro, `view(A, 4, 1:5)`
is a call to the [`view`](https://docs.julialang.org/en/v1/base/arrays/#Base.view) function.
There are a couple of important reasons why macro calls are visually distinct from function
calls (which sometimes upsets Lisp purists):

* the arguments of function calls are always evaluated before entering the body of the
  function: in `f(g(2, 5.6))`, we know that the function `g(2, 5.6)` will always be
  evaluated before calling `f` on its result.  Instead, the expression which is given as
  input to a macro can in principle be discarded and never taken into account and the macro
  can be something completely different: in `@f g(2, 5.6)`, the expression `g(2, 5.6)` is
  taken unevaluated and rewritten into something else.  The function `g` may actually never
  be called, or it may be called with different arguments, or whatever the macro `@f` has
  been designed to rewrite the given expression;
* nested macro calls are evaluated left-to-right, while nested function calls are evaluated
  right-to-left: in the expression `f(g())`, the function `g()` is first called and then fed
  into `f`, instead in `@f @g x` the macro `@f` will first rewrite the unevaluated
  expression `@g x`, and then, if it is still there after the expression-rewrite operated by
  `@f` (remember the previous point?), the macro `@g` is expanded.

Writing a macro can also be an unpleasant experience the first times you try it, because a
macro operates on a new level (expressions, which you want to turn into other expressions)
and you need to get familiar with new concepts, like
[hygiene](https://en.wikipedia.org/wiki/Hygienic_macro) (more on this below), which can be
tricky to get initially right.  However it may help remembering that the `Expr` macros
operate on are regular Julia objects, which you can access and modify like any other Julia
structure.  So you can think of a macro as a regular function which operates on `Expr`
objects, to return a new `Expr`.  This isn't a simplification: many macros are actually
defined to only call a regular function which does all the expression rewriting business.

_Q_: _**Should I write a macro?**_

_A_: If you're asking yourself this question, the answer is likely "no".  Steven G. Johnson
gave an interesting [keynote speech at JuliaCon
2019](https://www.youtube.com/watch?v=mSgXWpvQEHE) about metaprogramming (not just in
Julia), explaining when to use it, and more importantly when _not_ to use it.

Also, macros don't compose very well: remember that any macro can rewrite an expression in a
completely arbitrary way, so nesting macros can sometimes have unexpected results, if the
outermost macro doesn't anticipate the possibility the expressions it operates on may
contain another macro which expects a specific expression.  In practice, this is less of a
problem than it may sound, but it can definitely happens if you overuse many complicated
macros.  This is one more reason why you should not write a macro in your code unless it's
really necessary to substantially simplify the code.

_Q_: _**So, what's the deal with macros?**_

_A_: Macros are useful to programmatically generate code which would be tedious, or very
complicated, to type manually.  Keep in mind that the goal of a macro is to eventually get a
new expression, which is run when the macro is called, so the generated code will be
executed regularly, no shortcut about that, which is sometimes a misconception.  There is
_very_ little that plain (i.e. not using macros) Julia code cannot do that macros can.

_Q_: _**Why I should keep reading this post?**_

_A_: Writing a simple macro can still be a useful exercise to learn how it works, and
understand when they can be useful.

## Our first macro: `@stable`

Globals in Julia are usually a [major performance
hit](https://docs.julialang.org/en/v1/manual/performance-tips/), when their type is not
constant, because the compiler have no idea what's their actual type.  When you don't need
to reassign a global variable, you can mark it as constant with the
[`const`](https://docs.julialang.org/en/v1/base/base/#const) keyword, which greatly improves
performance of accessing a global variable, because the compiler will know its type and can
reason about it.

Julia v1.8 introduces a new way to have non-horribly-slow global variables: you can annotate
the type of a global variable, to say that its type won't change:

```julia
julia> x::Float64 = 3.14
3.14

julia> x = -4.2
-4.2

julia> x = 10
10

julia> x
10.0

julia> x = nothing
ERROR: MethodError: Cannot `convert` an object of type Nothing to an object of type Float64
Closest candidates are:
  convert(::Type{T}, ::T) where T<:Number at number.jl:6
  convert(::Type{T}, ::Number) where T<:Number at number.jl:7
  convert(::Type{T}, ::Base.TwicePrecision) where T<:Number at twiceprecision.jl:273
  ...
```

This is different from a `const` variable, because you can reassign a type-annotated global
variable to another value with the same type (or
[`convert`](https://docs.julialang.org/en/v1/base/base/#Base.convert)ible to the annotated
type), while reassigning a `const` variable isn't possible (nor recommended).  But the type
would still be constant, which helps the compiler optimising code which accesses this global
variable.  Note that `x = 10` returns `10` (an `Int`) but the actual value of `x` is `10.0`
(a `Float64`) because [assignment returns the right-hand side]({{ site.baseurl }}{% post_url
2017-12-02-julia-assignment %}) but the value `10` is `convert`ed to `Float64` before
assigning it to `x`.

_Wait, there is no macro so far!_  Right, we'll get there soon.  The problem with
type-annotated globals is that in the expression `x::Float64 = 3.14` it's easy to predict
the type we want to attach to `x`, but if you want to make `x = f()` a type-annotated global
variable and the type of `f()` is much more involved than `Float64`, perhaps a [type with
few parameters](https://docs.julialang.org/en/v1/manual/types/#Parametric-Types), then doing
the type annotation can be annoying.  Mind, not impossible, just tedious.  So, that's where
a macro could come in handy!

The idea is to have a macro, which we'll call `@stable`, which operates like this:

```julia
@stable x = 2
```

will automatically run something like

```julia
x::typeof(2) = 2
```

so that we can automatically infer the type of `x` from the expression on the right-hand
side, without having to type it ourselves.  A useful tool when dealing with macros is
[`Meta.@dump`](https://docs.julialang.org/en/v1/base/io-network/#Base.Meta.@dump) (another
macro, yay!).

```julia
julia> Meta.@dump x = 2
Expr
  head: Symbol =
  args: Array{Any}((2,))
    1: Symbol x
    2: Int64 2
```

This tells us how the expression `x = 2` is parsed into an `Expr`, when it's fed into a
macro.  So, this means that our `@stable x = 2` macro will see an `Expr` whose `ex.args[1]`
field is the name of the variable we want to create and `ex.args[2]` is the value we want to
assign to it, which means the expression we want to generate will be something like
`ex.args[1]::typeof(ex.args[2]) = ex.args[2]`, but remember that you need to
[interpolate](https://docs.julialang.org/en/v1/manual/metaprogramming/#man-expression-interpolation)
variables inside a [quoted
expression](https://docs.julialang.org/en/v1/manual/metaprogramming/#Quoting):

```julia
julia> macro stable(ex::Expr)
           return :( $(ex.args[1])::typeof($(ex.args[2])) = $(ex.args[2]) )
       end
@stable (macro with 1 method)

julia> @stable x = 2
2
```

_Well, that was easy, it worked at the first try!  Now we can use our brand-new type-stable
`x`!  Let's do it!_

```julia
julia> x
ERROR: UndefVarError: x not defined
```

_Waaat!  What happened to our `x`, we just defined it above!  Didn't we?_  Well, let's use
yet another macro,
[`@macroexpand`](https://docs.julialang.org/en/v1/base/base/#Base.@macroexpand), to see
what's going on:

```julia
julia> @macroexpand @stable x = 2
:(var"#2#x"::Main.typeof(2) = 2)
```

_Uhm, that looks weird, we were expecting the expression to be `x::typeof(2)`, what's that
`var"#2#x"`?_  Let's see:

```julia
julia> var"#2#x"
ERROR: UndefVarError: #2#x not defined
```

_Another undefined variable, I'm more and more confused._  What if that `2` in there is a
global counter?  Maybe we need to try with `1`:

```julia
julia> var"#1#x"
2

julia> var"#1#x" = 5
5

julia> var"#1#x"
5

julia> var"#1#x" = nothing
ERROR: MethodError: Cannot `convert` an object of type Nothing to an object of type Int64
Closest candidates are:
  convert(::Type{T}, ::T) where T<:Number at number.jl:6
  convert(::Type{T}, ::Number) where T<:Number at number.jl:7
  convert(::Type{T}, ::Base.TwicePrecision) where T<:Number at twiceprecision.jl:273
  ...
```

_Hey, here is our variable, and it's working as we expected!  But this isn't as convenient
as calling the variable `x` as we wanted.  I don't like that.  what's happening?_  Alright,
we're now running into hygiene, which we mentioned above: this isn't about washing your
hands, but about the fact macros need to make sure the variables in the returned expression
don't accidentally clash with variables in the scope they expand to.  This is achieved by
using the [`gensym`](https://docs.julialang.org/en/v1/base/base/#Base.gensym) function to
automatically generate unique identifiers (in the current module) to avoid clashes with
local variables.

What happened above is that our macro generated a variable with a `gensym`-ed name, instead
of the name we used in the expression, because macros in Julia are hygienic by default.  To
opt out of this mechanism, we can use the
[`esc`](https://docs.julialang.org/en/v1/base/base/#Base.esc) function.  A rule of thumb is
that you should apply `esc` on input arguments if they contain variables or identifiers from
the scope of the calling site that you need use as they are, but for more details do read
the [section about hygiene in the Julia
manual](https://docs.julialang.org/en/v1/manual/metaprogramming/#Hygiene).  Note also that
the pattern `var"#N#x"`, with increasing `N` at every macro call, in the `gensym`-ed
variable name is an implementation detail which may change in future versions of Julia,
don't rely on it.

Now we should know how to fix the `@stable` macro:

```julia
julia> macro stable(ex::Expr)
           return :( $(esc(ex.args[1]))::typeof($(esc(ex.args[2]))) = $(esc(ex.args[2])) )
       end
@stable (macro with 1 method)

julia> @stable x = 2
2

julia> x
2

julia> x = 4.0
4.0

julia> x
4

julia> x = "hello world"
ERROR: MethodError: Cannot `convert` an object of type String to an object of type Int64
Closest candidates are:
  convert(::Type{T}, ::T) where T<:Number at number.jl:6
  convert(::Type{T}, ::Number) where T<:Number at number.jl:7
  convert(::Type{T}, ::Base.TwicePrecision) where T<:Number at twiceprecision.jl:273
  ...

julia> @macroexpand @stable x = 2
:(x::Main.typeof(2) = 2)
```

_Cool, this is all working as expected!  Are we done now?_  Yes, we're heading into the
right direction, but no, we aren't quite done yet.  Let's consider a more sophisticated
example, where the right-hand side is a function call and not a simple literal number, which
is why we started all of this.  For example, let's define a new type-stable variable with a
[`rand()`](https://docs.julialang.org/en/v1/stdlib/Random/#Base.rand) value, and let's print
it with one more macro, [`@show`](https://docs.julialang.org/en/v1/base/base/#Base.@show),
just to be sure:

```julia
julia> @stable y = @show(rand())
rand() = 0.19171602949009747
rand() = 0.5007039099074341
0.5007039099074341

julia> y
0.5007039099074341
```

_Ugh, that doesn't look good.  We're calling `rand()` twice and getting two different
values?_  Let's ask again our friend `@macroexpand` what's going on (no need to use `@show`
this time):

```julia
julia> @macroexpand @stable y = rand()
:(y::Main.typeof(rand()) = rand())
```

_Oh, I think I see it now: the way we defined the macro, the same expression, `rand()`, is
used twice: once inside `typeof`, and then on the right-hand side of the assignment, but
this means we're actually calling that function twice, even though the expression is the
same._  Correct!  And this isn't good for at least two reasons:

* the expression on the right-hand side of the assignment can be expensive to run, and
  calling it twice wouldn't be a good outcome: we wanted to create a macro to simply things,
  not to spend twice as much time;
* the expression on the right-hand side of the assignment can have side effects, which is
  precisely the case of the `rand()` function: every time you call `rand()` you're advancing
  the mutable state of the random number generator, but if you call it twice instead of
  once, you're doing something unexpected.  By simply looking at the code `@stable y =
  rand()`, someone would expect that `rand()` is called exactly once, it'd be bad if users
  of your macro would experience undesired side effects, which can make for hard-to-debug
  issues.

In order to avoid double evaluation of the expression, we can assign it to another temporary
variable, and then use its value in the assignment expression:

```julia
julia> macro stable(ex::Expr)
           quote
               tmp = $(esc(ex.args[2]))
               $(esc(ex.args[1]))::typeof(tmp) = tmp
           end
       end
@stable (macro with 1 method)

julia> @stable y = @show(rand())
rand() = 0.5954734423582769
0.5954734423582769
```

_This time `rand()` was called only once!  That's good, isn't it?_ It is indeed, but I think
we can still improve the macro a little bit.  For example, let's look at the list of all
names defined in the current module with
[`names`](https://docs.julialang.org/en/v1/base/base/#Base.names).  Can you spot anything
strange?

```julia
julia> names(@__MODULE__; all=true)
13-element Vector{Symbol}:
 Symbol("##meta#58")
 Symbol("#1#2")
 Symbol("#1#x")
 Symbol("#5#tmp")
 Symbol("#@stable")
 Symbol("@stable")
 :Base
 :Core
 :InteractiveUtils
 :Main
 :ans
 :x
 :y
```

_I have a baad feeling about that `Symbol("#5#tmp")`.  Are we leaking the temporary variable
in the returned expression?_  Correct!  Admittedly, this isn't a too big of a deal, the
variable is `gensym`-ed and so it won't clash with any other local variables thanks to
hygiene, many people would just ignore this minor issue, but I believe it'd still be nice to
avoid leaking it in the first place, if possible.  We can do that by sticking the
[`local`](https://docs.julialang.org/en/v1/base/base/#local) keyword in front of the
temporary variable:

```julia
julia> macro stable(ex::Expr)
           quote
               local tmp = $(esc(ex.args[2]))
               $(esc(ex.args[1]))::typeof(tmp) = tmp
           end
       end
@stable (macro with 1 method)

julia> @stable y = rand()
0.7029553059625194

julia> @stable y = rand()
0.04552255224129409

julia> names(@__MODULE__; all=true)
13-element Vector{Symbol}:
 Symbol("##meta#58")
 Symbol("#1#2")
 Symbol("#1#x")
 Symbol("#5#tmp")
 Symbol("#@stable")
 Symbol("@stable")
 :Base
 :Core
 :InteractiveUtils
 :Main
 :ans
 :x
 :y
```

_Yay, no other leaked temporary variables!  Are we done now?_  Not yet, we can still make it
more robust.  At the moment we're assuming that the expression fed into our `@stable` macro
is an assignment, but what if it isn't the case?

```julia
julia> @stable x * 12
4

julia> x
4
```

_Uhm, it doesn't look like anything happened, `x` is still `4`, maybe we can ignore also
this case and move on._  Not so fast:

```julia
julia> 1 * 2
ERROR: MethodError: objects of type Int64 are not callable
Maybe you forgot to use an operator such as *, ^, %, / etc. ?
```

_Aaargh!  What does that even mean?!?_ Let's ask our dear friends `Meta.@dump` and
`@macroexpand`:

```julia
julia> Meta.@dump x * 2
Expr
  head: Symbol call
  args: Array{Any}((3,))
    1: Symbol *
    2: Symbol x
    3: Int64 2

julia> @macroexpand @stable x * 12
quote
    var"#5#tmp" = x
    (*)::Main.typeof(var"#5#tmp") = var"#5#tmp"
end

julia> *
4
```

_Let me see if I follow: with `@stable x * 12` we're assigning `x` (which is now
`ex.args[2]`) to the temporary variable, and then the assignment is basically `* = 4`,
because `ex.args[1]` is now `*`.  Ooops._  Brilliant!  In particular we're shadowing `*` in
the current scope (for example the `Main` module in the REPL, if you're following along in
the REPL) with the number `4`, the expression `1 * 2` is actually equivalent to `*(1, 2)`,
and since `*` is `4`

```julia
julia> 4(1, 2)
ERROR: MethodError: objects of type Int64 are not callable
Maybe you forgot to use an operator such as *, ^, %, / etc. ?
```

_Gotcha!  So we should validate the input?_  Indeed, we should make sure the expression
passed to the macro is what we expect, that is an assignment.  We've already seen before
that this means `ex.head` should be the symbol `=`.  We should also make sure the left-hand
side is only a variable name, we don't want to mess up with indexing expressions like `A[1]
= 2`:

```julia
julia> Meta.@dump A[1] = 2
Expr
  head: Symbol =
  args: Array{Any}((2,))
    1: Expr
      head: Symbol ref
      args: Array{Any}((2,))
        1: Symbol A
        2: Int64 1
    2: Int64 2
```

_Right, so `ex.head` should be only `=` and `ex.args[1]` should only be another symbol.  In
the other cases we should throw a useful error message._  You're getting the hang of it!

```julia
julia> macro stable(ex::Expr)
           (ex.head === :(=) && ex.args[1] isa Symbol) || throw(ArgumentError("@stable: `$(ex)` is not an assigment expression."))
           quote
               tmp = $(esc(ex.args[2]))
               $(esc(ex.args[1]))::typeof(tmp) = tmp
           end
       end
@stable (macro with 1 method)

julia> @stable x * 12
ERROR: LoadError: ArgumentError: @stable: `x * 12` is not an assigment expression.

julia> @stable A[1] = 2
ERROR: LoadError: ArgumentError: @stable: `A[1] = 2` is not an assigment expression.
```

_Awesome, I think I'm now happy with my first macro!  Love it!_ Yes, now it works pretty
well and it has also good handling of errors!  Nice job!

## Conclusions

I hope this post was instructive to learn how to write a very basic macro in Julia.  In the
end, the macro we wrote is quite short and not very complicated, but we ran into many
pitfalls along the way: hygiene, thinking about corner cases of expressions, avoiding
repeated undesired evaluations and introducing extra variables in the scope of the macro's
call-site.  This also shows the purpose of macros: rewriting expressions into other ones, to
simplify writing more complicated expressions or programmatically write more code.

This post is inspired by a macro which I wrote some months ago for the [Seven Lines of
Julia](https://discourse.julialang.org/t/seven-lines-of-julia-examples-sought/50416/157)
thread on JuliaLang Discourse.

_This was cool!  Where can I learn more about macros?_  Good to hear!  I hope now you aren't
going to abuse macros though!  But if you do want to learn something more about macros, in
addition to the official documentation, some useful resources are:

* [A Brief Introduction to Julia Metaprogramming, Macros, and Generated
  Functions](https://nextjournal.com/a/KpqWNKDvNLnkBrgiasA35?change-id=CQRuZrWB1XaT71H92x8Y2Q)
* [Introduction to macros | Week 3 | 18.S191 MIT Fall
  2020](https://www.youtube.com/watch?v=e6LGMeoQhfs)
* [Julia Macro hygiene made easy! - Tom Kwong](https://www.youtube.com/watch?v=JePBb9-ychE)
* [Julia macros for
  beginners](https://jkrumbiegel.com/pages/2021-06-07-macros-for-beginners/)

<!-- Local Variables: -->
<!-- ispell-local-dictionary: "british" -->
<!-- fill-column: 92 -->
<!-- End: -->
