---
feature: list-index-syntax
start-date: 2024-07-14
author: rhendric
co-authors:
shepherd-team: infinisil inclyc AndersonTorres
shepherd-leader: infinisil
related-issues: https://github.com/NixOS/nix/issues/10949, https://github.com/NixOS/rfcs/pull/137
---

# Summary
[summary]: #summary

This proposal extends the attrpath syntax to include `'[' INT ']'` components that refer to elements in lists.
This would enable expressions such as the following:

```nix
x.[0]       # = builtins.elemAt x 0
x.[1] or y  # = if builtins.isList x && builtins.length x > 1 then builtins.elemAt x 1 else y
x ? [3].y   # = builtins.isList x && builtins.length x > 3 && builtins.elemAt x 3 ? y
```

# Motivation
[motivation]: #motivation

I'm in a REPL.
I'm exploring parts of Nixpkgs.
I type a partial expression:

```
nix-repl> someExpr.foo
{ bar = { ... }; ignoreThis = { ... }; moreStuff = { ... }; }
```

I hit up-arrow and keep typing to drill deeper:

```
nix-repl> someExpr.foo.bar
{ baz = true; qux = [ ... ]; quux = { ... }; }

nix-repl> someExpr.foo.bar.qux
[ { ... } ]

nix-repl> someExpr.foo.bar.qux.0
error: attempt to call something which is not a function but a list

       at «string»:1:1:

            1| someExpr.foo.bar.qux.0
             | ^
```

Of course that doesn't work.
I don't actually type that.
What I actually do is hit up-arrow, then hit Home, then type `builtins.elemAt`, then hit End, then type ` 0`.

```
nix-repl> builtins.elemAt someExpr.foo.bar.qux 0
{ greatMoreStuff = { ... }; }
```

Now what do I get to do?
That's right, hit up-arrow, then hit Home, then type `(`, then hit End, then type `).greatMoreStuff`.

---

When writing Nix code, it is relatively uncommon to want to index into a list, and `builtins.elemAt` suffices.
When exploring data in a REPL, however, indexing into lists is more common, and the above example illustrates how `elemAt` is incompatible with the attrpath selector syntax that is the primary means of data exploration.
Many other programming languages allow syntaxes such as
`foo.bar.qux[0].moreStuff` (C, many others)
or `foo.bar.qux(0).moreStuff` (Octave, Scala)
or `foo.bar.qux.[0].moreStuff` (F#, OCaml)
or `foo.bar.qux.0.moreStuff` (Coco, LiveScript, the `--attr` option of `nix-instantiate` and other `nix` commands).
Of these, only `.[]` does not conflict with existing syntax (see [Alternatives] for more details).

# Detailed design
[design]: #detailed-design

The `attrpath` grammar nonterminal is currently defined as

```
attrpath
  : attrpath '.' attr
  | attrpath '.' string_attr
  | attr
  | string_attr
  ;
```

where `attr` is a simple identifier and `string_attr` is either a string literal or a bare `${}` interpolation.
This nonterminal is used in three contexts in the grammar:
* In a selector (e.g. `expr.attrpath` or `expr.attrpath or default`)
* As the right-hand side of the `?` operator (e.g. `expr ? attrpath`)
* As the left-hand side of a binding (e.g. `let attrpath = expr; in body` or `{ attrpath = expr; }`)

This proposal adds two productions to `attrpath`:

```
  | attrpath '.' '[' INT ']'
  | '[' INT ']'
```

(Supporting non-literal expressions is scoped out of this proposal; see [Future work][future].)

In a selector or `?` operator, these new productions have the following semantics:
* `expr.prefix.[n].suffix` is the result of evaluating `(builtins.elemAt (expr.prefix) n).suffix`
* `expr ? prefix.[n].suffix` is true if and only if all of the following are true:
  * `expr ? prefix` (or `prefix` is empty)
  * `expr.prefix` evaluates to a list with length at least `n + 1`
  * `expr.prefix.[n] ? suffix` (or `suffix` is empty)
* `expr.prefix.[n].suffix or expr2` is the result of evaluating `if expr ? prefix.[n].suffix then expr.prefix.[n].suffix else expr2`

It is a syntax error if either new production is used in the left-hand side of a binding.

An implementation of this design is available as patches for Nix at <https://gitlab.com/rhendric/nix-list-index-syntax/>; see instructions there for use.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

```
nix-repl> [ 1 4 9 ].[2]
9

nix-repl> pkgs.maptool.meta.maintainers.[0].github
"rhendric"

nix-repl> [ 1 4 9 ].[3]
error: list index 3 is out of bounds

       at «string»:1:1:

            1| [ 1 4 9 ].[3]
             | ^

nix-repl> [ 1 4 9 ] ? [2]
true

nix-repl> [ 1 4 9 ] ? [3]
false

nix-repl> [ 1 4 9 ] ? [0].[0]
false

nix-repl> [ [ 1 4 9 ] ] ? [0].[0]
true
```

The following are all syntax errors:

```
nix-repl> let n = 0; in [ 4 ].[n]
error: syntax error, unexpected ID, expecting INT

       at «string»:1:22:

            1| let n = 0; in [ 4 ].[n
             |                      ^

nix-repl> { a.[0] = true; }
error: syntax error, index '0' not allowed here

       at «string»:1:3:

            1| { a.[0] = true;
             |   ^

nix-repl> let a.[0] = true; in a                   
error: syntax error, index '0' not allowed here

       at «string»:1:5:

            1| let a.[0] = true;
             |     ^
```

# Drawbacks
[drawbacks]: #drawbacks

The prototype [patch] implementing this feature adds a net 69 lines of code to Nix, excluding tests.

[patch]: https://gitlab.com/rhendric/nix-list-index-syntax/-/blob/main-dist/patches/bracketed/from-90e630a5.patch

As noted in passing in the motivation section, the proposed syntax differs from the syntax already used by Nix tools on the command line for `--attr`.
`nix-instantiate` has no difficulty with:

```
$ nix-instantiate --eval '<nixpkgs>' -A maptool.meta.maintainers.0.github
"rhendric"
```

This divergence might lead to confusion.

Evolving the syntax of Nix always imposes a cost on third-party tools that process Nix syntax, including syntax highlighters, linters, formatters, static analyzers, and language servers.
This syntax is not a dramatic extension of the language but would require support from all of the above for them to maintain full functionality.

Finally, there is an opportunity cost to claiming new syntax.
One could imagine speculative features that might want to use this syntax, such as a list or string slicing syntax, or a ‘list swizzle’ operator that desugars `expr.[ 2 0 1 ]` to `[ (elemAt expr 2) (elemAt expr 0) (elemAt expr 1) ]`.
It is, in my opinion, unlikely that list and string manipulation (assuming that any feature in competition for this syntax would involve lists or strings somehow) would be so common in Nix to make this a compelling objection.

# Alternatives
[alternatives]: #alternatives

#### `expr.${0}`

The simplest alternative, aside from doing nothing, would be to reuse the `.${expr}` syntax that Nix already supports in attrpaths.
In this alternative, when evaluating `expr.${idx}`, the interpreter would determine whether `idx` evaluates to an integer or a string.
`expr` would be required to be a list if the former and an attribute set if the latter.

While this is parsimonious with respect to syntax, it creates more complexity for static analyzers.
It is currently statically known that the `expr` in `expr.${idx}` must be a set and that `idx` must be a string.
Representing a new constraint that _either_ `expr` is a set and `idx` is a string _or_ `expr` is a list and `idx` is an integer might be difficult for such tools and could result in a degraded user experience.

#### `expr[1]`

The most common syntax, by far, for indexing into a list or array in other programming languages is `expr[idx]`.
Nix is currently whitespace-insensitive with respect to attrpaths, so such an expression is indistinguishable from `expr [idx]`, an application of the function `expr` to the argument `[idx]`.
It would be possible to adapt the Nix lexer to interpret the `[` character in `expr[idx]` as the opening of an indexer or a list depending on whether there is whitespace before it.
Among the disadvantages of this approach would be that it would make specifying the grammar of Nix more complicated, and that it may prevent long attrpaths from being broken naturally across lines.
Perhaps most fatally, despite the odds of someone writing `expr[idx]` and intending a function call being virtually nil, the conservative principles of the Nix team forbid altering the meaning of even such an unlikely bit of syntax without some sort of larger language versioning or deprecation story, which has eluded us for some time.

All of the above objections apply to `expr(1)` as well, with the additional drawback that in a wide array of C-like languages, this syntax represents exactly what it currently (if coincidentally) represents in Nix.

#### `expr.$[2]`

The syntax `expr.$[idx]` was offered as a possibility in [NixOS/nix#10949](https://github.com/NixOS/nix/issues/10949).
It resembles the `${}` syntax already used in attrpaths, with the change in delimiters suggesting a shift from attribute sets to lists.
However, it requires an additional character to type and its technical qualities are identical to those of the proposed syntax without the `$` character.
There is at least some prior art for `.[]` in OCaml and F#; there is none that I know of for `.$[]`.

#### `expr.3`

It is no accident that a simple dotted index `expr.3` was the syntax chosen for attribute paths in `nix-instantiate` and friends.
It is the most straightforward expression of intent imaginable, if it is known for certain that an attribute path is what is intended.
Implementing the same syntax in the Nix language would be harmonious.
The drawback is that, as with the `expr[1]` case, whitespace insensitivity means that `expr.3` is indistinguishable from `expr .3`, the application of a function to a float literal.
As before, abandoning whitespace sensitivity is possible, if distasteful.

Another approach would be to abandon float literals that don't start with a digit.
There are currently no such literals in Nixpkgs.
Though such literals have historically been supported by many C-like languages, some languages (Haskell, Ruby, Rust, Swift) and the [Google C++ style guide](https://google.github.io/styleguide/cppguide.html#Floating_Literals) reject them.
Several standards for science and engineering, such as the United States' [NIST Guide to the SI][NIST] and [National Renewable Energy Laboratory's Communication Standards][NREL], do the same.
Forbidding them would also require a capacity for language versioning or deprecation, but the end result would not require adding whitespace sensitivity to the grammar.

[NIST]: https://www.nist.gov/pml/special-publication-811/nist-guide-si-chapter-10-more-printing-and-using-symbols-and-numbers#1052
[NREL]: https://www.nrel.gov/comm-standards/editorial/zero.html

If the practical barriers to introducing backwards incompatibilities into Nix were not a concern, this would be far and away my preferred choice.
An implementation of this option is also available at <https://gitlab.com/rhendric/nix-list-index-syntax/>, and I'm using it as my main Nix package.
(This patch is marginally more complex than the other because the lexer needs to be persuaded not to see a float in `expr.1.2`, but this is _not_ a grammar conflict because `expr . 1.2` doesn't parse as anything.)

#### `expr!4` (or other character)

To get the parsimony of `expr.3` but without the fuss of dealing with the float conflict, we could choose another symbolic character that isn't already an infix operator.
Ideally this character would be somewhat mnemonic.
Pros and cons of various choices briefly covered below, in highly subjective best-to-worst order.

* `!` (+ Haskelly, resembles `.` but, like, different)
* `@` (+ evokes `elemAt`; − apparently we want this to be reserved for things that create bindings)
* `\` (okay I guess?)
* `&` (− association with bitwise-and, but possibly in Nix that's a sufficiently remote concept to be irrelevant)
* `|` (− ditto for bitwise-or)
* `$` (− strong Haskell association with a different concept)
* `^` (− could be used for exponentiation if `**` isn't)
* `%` (− could be used for modulo)
* `` ` `` (− confusing)

Relative to the proposed syntax, any feasible option here seems like it would be more alienating to new users, to a degree not worth the benefit of saving two characters.

# Prior art
[prior-art]: #prior-art

As mentioned above, F# also supports a `.[]` syntax for indexing into arrays, though since F# 6.0 the C-like notation is recommended over this.
The following quote from the [changelog entry][F#6expridx] may be relevant, as a warning to us:

> Up to and including F# 5, F# has used `expr.[idx]` as indexing syntax. Allowing the use of `expr[idx]` is based on repeated feedback from those learning F# or seeing F# for the first time that the use of dot-notation indexing comes across as an unnecessary divergence from standard industry practice.

[F#6expridx]: https://learn.microsoft.com/en-us/dotnet/fsharp/whats-new/fsharp-6#simpler-indexing-syntax-with-expridx

# Unresolved questions
[unresolved]: #unresolved-questions

None at this time.

# Future work
[future]: #future-work

This proposal is motivated primarily, if not exclusively, by the REPL use case, in which the desired index is known.
While there is minimal technical challenge to allowing arbitrary expressions inside the square brackets, doing so would consume a larger slice of the syntax design space.
Just as there is a difference between `.foo` and `.${foo}`, it might be desirable to have a different syntax for this new case—`.[2]` and `.$[1 + 1]`, perhaps.
Discussing these issues and determining whether the benefits relative to using `elemAt` are worth the drawbacks is left as future work.
