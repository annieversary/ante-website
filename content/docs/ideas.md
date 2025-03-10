+++
title = "Ideas"
date = "2021-11-02"
categories = ["docs"]
+++

---

This page is an incomplete list of features that are currently
being considered for ante but for one reason or another are not
included in the language already. These features are listed
in no particular order.

---
# Polymorphic Variants or Union Types

Originally, polymorphic variants were devised as a scheme to infer
union types in a hindley-milner type system. Their inclusion into
ante would simplify error handling, allowing error unions to automatically
combine with each other:

```ante
// Using 'A for a polymorphic variant constructor
get (vec: Vec a) (index: usz) -> Result a 'NotFound =
    if index < len vec
    then Ok $ vec#index
    else 'NotFound

open (filename: string) -> Result File 'NoSuchFile =
    match File.open filename
    | Some file -> Ok file
    | None -> 'NoSuchFile

// Both error tags are automatically combined in the inferred
// return type: Result string [> 'NotFound | 'NoSuchFile]
foo (vec: Vec string) =
    filename = get? vec 0
    file = open? filename
    Ok (read_all file)
```

They are, however unable to accurately infer
types in all situations. Here are some problematic examples
along with their error messages (in OCaml 4.12.0) or inferred types
and what types we would ideally infer.

```ocaml
let foo x =
    match x with
    | `A -> x
    | `B -> `C

(*
Error: This expression has type [> `C ]
       but an expression was expected of type [< `A | `B ]
       The second variant type does not allow tag(s) `C
*)
```

Because we returned our input `x` which can only be either `A` or `B`,
polymorphic variants restrict our output types to be the same even though
we know in the branch `x` is returned in, it can only be `A`. When `C`
is returned from the second branch it then errors that it expected
(at most) `A` or `B` instead of inferring the preferred result of
```ocaml
foo : [< `A | `B ] -> [> `A | `C ]
```

Another problematic example is:

```ocaml
let bar = function
    | `A -> `B
    | y -> y
```
which infers
```ocaml
bar : ([> `A | `B ] as 'a) -> 'a
```
even though `A` can never be returned from the function, and the parameter
should be able to be any variant type instead of just `A` or `B`.
Ideally, it should infer something resembling
```ocaml
bar : [< `A | a] -> [> `B | a]
```

While useful, polymorphic variants aren't perfect. Generalized union types
may be more useful in they do not fall to these problems, but have the
caveat of requiring more advanced type machinary to infer them (namely
subtyping and intersection typing).

--- 
# Traits as Types

Allowing traits to be used in a type position such that they match with any type
that implements them could help ease some of the notational burden of traits. Prior
art here includes `impl Trait` and `dyn Trait` in rust, existential types in haskell,
any interface type in OO langs, and others.

An ideal implementation would take advantage of ante being slightly higher level than
rust to get rid of the distinction between `impl` and `dyn` trait to just choose the right
one where possible. This should allow for both:

```ante
print (x: Show) -> unit =
    printne "${show x}\n"
```

and

```ante
print_all (xs: Vec Show) -> unit =
    printne '['
    fields = map xs show |> join ", "
    iter fields printne
    printne ']'
```

Where the semantics of `print` likely translates to `fn print(x: impl Show)` in rust, and
the semantics of `print_all` likely translate to `fn print_all(xs: Vec<dyn Show>)`. An
alternative would be to have both functions be polymorphic over whether the underlying type
is known or whether the trait is dynamic.

## Multiple variable traits

In traits with multiple type parameters or traits with type parameters which are not used
directly in a function's parameters it is unclear which type a trait object would represent
at runtime. For example, given the traits

```ante
trait Pair a b with
    pack : a -> b -> string

trait Parse a with
    parse : string -> a

example1 (x: Pair) = pack ???

example2 (y: Parse) = parse ?
```

What should a trait object for `Pair` represent? Arbitrarily picking one type seems out
of the question. To be able to call `pack` we'd need to supply both parameters somehow.
A valid response may be just to limit trait objects to single parameter traits with
"object-safe" functions as rust does, but this may be more limiting than is needed. For
example, if we change our syntax such that the existential type variable must be explicitly
specified, then `Pair` becomes usable as a trait object as long as we specify a type to pack with:

```ante
example1 (x: Pair _ i32) = pack x 7

example1b (x: Pair string _) = pack "hello" x
```

This would incur some notational burden but is otherwise explicit and strictly more
flexible than the previous approach. `_` is likely not a good keyword to be used here
however since it is already used for explicit currying in ante, and this may be
applicable to type constructors some day.

## Exists syntax

It is worth briefly exploring a more explicit and flexible syntax via an `exists`
keyword to introduce an existential type variable (rather than the default `forall` quantified
type variables ante has). It sidesteps most of the issues with previous syntaxes for trait
objects by separating the exists clause from where the type is used later in the signature:

```ante
example1 (x: e?) -> string with Pair e? i32 = ...
```

Although flexible, this does not solve the original problem of improving the ergonomics of
using traits in function signatures. Instead, it makes it worse.

## Traits and effects

Since traits in ante can be thought of as a restricted form of effects which must resume in
a tail position and have an automatic impl search, a natural question that arises is "if there
are trait objects, are there effect objects too?"

At the time of writing, I'm leaning towards "no" as an answer for two reasons.
1. Trait objects exist to ease notational burden of traits or to provide dynamic dispatch.
   - Effects do not have the same notational burden since they do not need to have a type
   implementing the effect passed in through the function's parameters. There would thus
   be no benefit for most effects like `State s` because these are already only found in
   the effects clause of a type signature. Using traits in this way would be useless since
   without an accompanying `(iter: it)` parameter, a trait like `Iterator it elem` would
   not be usable within a function to begin with.
   - For similar reasons, effects do not need to be dynamically dispatched since they have
   no type that can represent them and handlers should be statically known.
2. Erasing effects in any way increases the difficulty of optimizing effects which would be
   a hard sell when algebraic effects must already be carefully optimized out to not incur
   great performance loss.

--- 
# Allocator Optimizations

The default allocator malloc in addition to it's faster friends jemalloc and mimalloc are
designed in such a way to make them general purpose: they must be thread-safe and they cannot
assume any lifetime constraints of their data. A very common manual optimization in languages
like C or C++ is then to switch out to a faster allocator for some data. For example, a game
may elect to use a bump-pointer allocator for any temporary data that is only needed to process
the current frame. Another optimization could be to swap out these global allocators for faster
thread-local allocators that require no atomic or locking instructions for synchronization.

## Automatic usage of thread-local allocators

In Ante, the goal should be to perform these optimizations automatically. The compiler should
be able to analyze the transfer of data such that if it is only used in a single thread, a faster thread-local
allocator is used over the global allocator. Otherwise, we'd fall back to a global thread-safe allocator like mimalloc.
This optimization would be similar to Koka's Perceus system in which atomic reference count instructions
are optimized into non-atomic reference count instructions if the referenced data is used in a single
threaded manner.

This will likely end up being an important optimization since thread local allocators can be 
substantially faster than global allocators. A key property of this optimization however is that it 
would only change the default allocator. If users wish to manually optimize their program and decide
which allocators to use where they will still be able to as before.

## Allocate all memory up-front on the stack

If the compiler can put a bound on the amount of memory a thread may allocate for either thread-local 
or shared data, we would be able to allocate all of a thread's memory up front. Thread-local data could 
simply be allocated on the thread's stack and shared data on the parent thread's stack or global allocator.
This optimization is likely less practical for longer running threads where no memory bound
is likely to be found.

## Grouping allocations with lifetime inference

A good way to optimize allocations is to simply make fewer of them. Ante should be able to leverage
lifetime analysis to find places where we can combine allocations and deallocations of multiple variables.
A trivial example would be:

```ante
foo () =
    l1 = [1, 2, 3] : List i32
    l2 = [4, 5, 6] ++ l1
    ...
    ()
```

A naive implementation may allocate all these list nodes separately but since they are created adjacent
to each other and neither are needed outside the current function we should be able to optimize the 6
list node allocations into 1 call to the allocator. In this specific example, the allocator may also
simply be the stack itself since we do not need a dynamic lifetime for these nodes in this function.

Grouping allocations in this way however leads to questions on how aggressively we should group adjacent
items. What if they are separated by other definitons? Or arbitrary function calls? In general widening
the lifetime of one allocation to match another's so it may be grouped increases the amount of memory
the resulting program would use by allocating earlier and freeing later. It is unclear what heuristics
should be used - if any - to make this decision on when to group allocations separated by other statements.

---
# Always Incremental Compilation

For languages with long compile times like rust (and presumably ante due to refinement types, lifetime inference,
and monomorphisation) incremental compilation largely solves the problem of speeding up compile times when developers
are iterating on a problem. It does not solve the problem for users however when they go to download a 
program or library which now must be recompiled from scratch. Why is this the case? Even if the user is on 
some new architecture that the compiler must optimize for, this does not mean we should have to re-do all of 
lexing, parsing, name resolution, type checking, etc for programs that we already know to compile.

Ante's solution to this will be experimenting with distributing the incremental compilation metadata along with
the code. When you download a library from a trusted source (centralized package repository or a company-specific
local repository) you can compile the project incrementally rather than compiling from scratch. Downloading a new
library and adding a line to your program making use of it should take no longer to compile than adding a line to 
your program without adding a new library.

## Formatting

The language Unison represents a codebase as a sqlite database rather than traditional text. It achieves
incremental compilation by only inserting verified code that passed type checking and all
prior passes into this database. It would be possible for ante to store incremental compilation metadata in 
such a database as well. Advantages of this scheme would be leveraging a pre-existing tool and packing metadata
together into a single somewhat standard binary format.

## Limitations

This approach of distributing incremental compilation metadata has a few limitations that are worth noting.

### Space and Downloads

Locally, little to no extra space should be required from this feature as the extra incremental information downloaded
from a library would have been created anyway when users go to compile the library. It is possible to use extra space 
if the compiler ever stores extra information that would be unneeded for some users. For example, caching different llvm
IR representations that are dependent on the host's architecture. Solving this may either mean compressing the data so that
as little space as possible is duplicated, or it may mean not saving data that is very dependent on the host's system such 
as llvm IR.

Downloading this data rather than creating it on compilation would mean an increase in download times. Compared to Unison,
ante users would be downloading both the textual code and the incremental version rather than just the later. It is unclear
how much of a problem it would be in practice. One potential solution would be to ensure the incremental data of a library
or program contains all the information needed to compile it. Then downloading the release of a library from a package manager
would only entail downloading the metadata and not the source code itself. One potential issue with this solution is IDE integration.
A hypothetical ante-language-server could be able to read type signatures or documentation from this, but if the user wished
to explore the source code of the library they would likely only be able to see a pretty-printed AST recreated from the metadata.
