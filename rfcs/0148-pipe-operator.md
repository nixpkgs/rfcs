---
feature: pipe-operator
start-date: 2023-05-23
author: @piegamesde
shepherd-team: @roberth @rhendric @illustris @adrian-gierakowski
shepherd-leader: @rhendric
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

Introduce a new "pipe" operator, `|>`, to the Nix language, defined as `f a` = `a |> f`.
Additionally, elevate `lib.pipe` to a built-in function.

As a reminder, `pipe a [ f g h ]` is defined as `h (g (f a))`.

# Motivation
[motivation]: #motivation

Creating advanced data processing like transforming a list is a thing commonly done in nixpkgs.
Yet the language has no support for function concatentation/composition,
which results in such constructs looking unwieldy and difficult to format well.
`lib.pipe` may be the most powerful library function with that regard,
but it is unknown and overlooked by many because it is not easily discoverable:
Despite its great usefulness, it is currently used in less than 30 files in Nixpkgs
(`rg '[\. ]pipe .* \['`).
Additionally, it is not accessible to Nix code outside of nixpkgs,
and due to Nix's lazy evaluation debugging type errors is really difficult.

Let's have a look at an arbitrarily chosen snippet of Nixpkgs code:

```nix
defaultPrefsFile = pkgs.writeText "nixos-default-prefs.js" (lib.concatStringsSep "\n" (lib.mapAttrsToList (key: value: ''
  // ${value.reason}
  pref("${key}", ${builtins.toJSON value.value});
'') defaultPrefs));
```

It is arguably pretty hard to read and reason about. Even when applying some more whitespace-generous formatting:

```nix
defaultPrefsFile = pkgs.writeText "nixos-default-prefs.js" (
  lib.concatStringsSep "\n" (
    lib.mapAttrsToList
    (
      key: value: ''
        // ${value.reason}
        pref("${key}", ${builtins.toJSON value.value});
      ''
    )
    defaultPrefs
  )
);
```

One can observe the following issues:

- If you want to follow the data flow, you must read it from bottom to top,
  from the inside to the outside (the input here is `defaultPrefs`).
- Adding a function call to the output would require wrapping the entire
  expression in parentheses and increasing its indentation.

Compare this to the equivalent call with `lib.pipe`:

```nix
defaultPrefsFile = pipe defaultPrefs [
  (lib.mapAttrsToList (
    key: value: ''
      // ${value.reason}
      pref("${key}", ${builtins.toJSON value.value});
    ''
  ))
  (lib.concatStringsSep "\n")
  (pkgs.writeText "nixos-default-prefs.js")
];
```

The code now clearly reads from top to bottom in the order the data is processed,
it is easy to add and remove processing steps at any point.

With a dedicated pipe operator, it would look like this:

```nix
defaultPrefsFile = defaultPrefs
  |> lib.mapAttrsToList (
    key: value: ''
      // ${value.reason}
      pref("${key}", ${builtins.toJSON value.value});
    ''
  )
  |> lib.concatStringsSep "\n"
  |> pkgs.writeText "nixos-default-prefs.js";
```

The artificial distinction between the first input and the functions via the list now is gone,
and so are the parentheses around the functions.
With the lower character overhead, using the operator becomes attractive in more situations,
whereas a `pipe` pays for its overhead only in more complex scenarios (usually three functions or more).
Having a dedicated operator also increases visibility and discoverability of the feature.

# Detailed design
[design]: #detailed-design

## `|>` operator

A new operator `|>` is introduced into the Nix language.
It is defined as function application with the order of arguments swapped: `f a` = `a |> f`.
It is left-associative and has a binding strength weaker than function application:
`a |> f |> g b |> h` = `h ((g b) (f a))`.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

## Tooling support

Like any language extension, this will require the available Nix tooling to be updated.
Updating parsers should be pretty easy, as the syntax changes to the language are fairly minimal.
Tooling that evaluates Nix code in some way or does static code analysis should be easy to support too,
since one may treat the operator as syntactic sugar for function application.
No fundamentally new semantics are introduced to the language.

## Nixpkgs interaction

As soon as the Nixpkgs minimum version contains `|>`, using it will be allowed and encouraged in the documentation.
There might be efforts to automatically convert existing `lib.pipe` usage or even discourage/deprecate using that,
see future work.

### Existing lib functions

Nixpkgs `lib` contains a couple of functions that are concatenated versions of other lib functions,
for example `concatMapStringsSep` being a fuse of `map` and `concatStringsSep`.
This is not unusual in many programming languages,
nevertheless the existence of easy to use piping functionality would reduce the need for some of them.

Of course removing existing lib functions is not an option, but in the future,
newly added functions should meet stronger criteria than being purely convenience helpers replacing two function calls with one.

To keep with that example, is the function called `concatMapStringsSep` or `concatMapStringSep`?
In which order do you provide the mapper or the separator first?
Using `map (…) |> concatStringsSep` requires to memorize less information.
Some example with different alternatives:

```nix
lib.concatMapStringsSep "\n" (test: writeTest "success" test.name "${test}/bin/${test.name}") (lib.attrValues bin)

lib.concatStringsSep "\n" (map (test: writeTest "success" test.name "${test}/bin/${test.name}") (lib.attrValues bin))

lib.attrValues bin |> map (test: writeTest "success" test.name "${test}/bin/${test.name}") |> lib.concatStringsSep "\n"

lib.concatStringsSep "\n" <| map (test: writeTest "success" test.name "${test}/bin/${test.name}") <| lib.attrValues bin
```

# Prior art

Nickel has `|>` too, with the same name and semantics.

F# has `|>`, called "pipe-forward" operator, with the same semantics.
Additionally, it also has "pipe-backward" `<|` and `>>`/`<<` for forwards and backwards function composition.
`<|` is equivalent to function application, however its lower binding order allows removing parentheses:
`g (f a)` = `g <| f a`. All these operators have the same precedence and are left-associative.
F#'s `<|` being left-associative strongly reduces its power of usage,
this can be considered a mistake/compromise/collateral in the language design.
All other discussed variants of `<|` in other languages are right-associative.

Elm has the same operators as F#.

Haskell has the (backwards) function composition operator `.` in its prelude: `(g . f) a` = `g (f a)`.
It also has "reverse application" `&`, which is roughly equivalent to `|>`,
and `$`, which is function application again but right-associative and very weakly binding.
`.` binds stronger than both.

`|>` is definable as an infix function in several other programming languages,
and in even more languages as macro or higher-order function (including Nix, that's `lib.pipe`).
Notable, the Haskell package `flow` provides some common operators like `|>` and `<|`,
with the usual associativity and same binding strength (unlike Haskell's `$` and `&` discussed above).

Languages that allow for custom operators with custom associativity and precedence like Haskell and Scala
(but unlike F#) usually forbid mixing same-strengh operators with different associativity without using parentheses
as a syntax/compile error.

# Alternatives
[alternatives]: #alternatives

For each change this RFC proposes, there is always the trivial alternative of not doing it. See #drawbacks.

We could use the occasion and introduce more operators like those mentioned above.

## Function composition operators

Function composition is mostly interesting for the so-called "point-free" programming style,
where partially applied compositions of functions are preferred over the introduction of lambda terms.
However, Nix is not well suited for that programming style for various reasons,
nor would that point-free style have nearly as many applications in typical Nixpkgs code.

Take for example this library function, written in a point-free style by using `flip pipe` as function concatenation operator:

```nix
concatMapAttrs = f: flip pipe [ (mapAttrs f) attrValues (foldl' mergeAttrs { }) ];
```

When reading this code, one has to manually do the headwork of inferring the types to understand what this function does.
In Haskell, its powerful type system and type inference would quickly spot any mistakes made.
But in Nix, this can lead to very confusing runtime errors instead
(even ignoring the additional stack trace noise of using `flip pipe`).
Compare this to the fully specifified version of the same function:

```nix
concatMapAttrs = f: v: pipe v [ (mapAttrs f) attrValues (foldl' mergeAttrs { }) ];
```

Would you have guessed correctly from the first code example whether it's `f: v:` or `v: f:`?

## Pipe-forward vs pipe-backward

We could use `<|` instead of `|>` instead:

```nix
defaultPrefsFile = 
  pkgs.writeText "nixos-default-prefs.js" <|
  lib.concatStringsSep "\n" <|
  lib.mapAttrsToList (
    key: value: ''
      // ${value.reason}
      pref("${key}", ${builtins.toJSON value.value});
    ''
  ) <| # the '<|' here is optional/redundant
  defaultPrefs
  ;
```

`<|` also opens up to other scenarios in which `|>` might be less well suited
(examples inspired by https://github.com/NixOS/nix/issues/1845):

```nix
lib.makeOverridable <|
{ foo, bar }:

builtins.trace "my debug stuff" <|
# some more code here
```

While only one of them would probably be sufficient for most use cases, we could also have both `|>` and `<|`.
Given that we want to call them `|>` and `<|`, users should assume both having equal binding strength.
Therefore mixing them without parentheses should be forbidden like in other languages,
having `<|` weaker than `|>` like Haskell's `$` and `&` would be a bad idea.

## Change the `pipe` function signature

There are many equivalent ways to declare this function, instead of just using the current design.
For example, one could flip its arguments so that it becomes function composition on list of functions.
Not only can this function trivially replace pipe; it can also be readily used where a function is expected, such as in `map`.

The current design of `pipe` has the advantage that its asymmetry points at its operating direction, which is quite valuable.

## `apply` keyword

As suggested in https://github.com/NixOS/rfcs/pull/148#discussion_r1206966546,
one could introduce a keyword (tentatively called `apply`) for piping,
which syntactically similar to `with` and `assert` statements:

```nix
apply f;
apply g;
x

# The same as
f (g x)
```

The biggest disadvantage with it is backwards compatibility of adding a new keyword into the language,
which would require solving language versioning first (see RFC #137).

This approach would be roughly equivalent to introducing a `<|` operator.
See the above for a discussion on the overall design space of that.

## `builtins.pipe`

`lib.pipe`'s functionality could be implemented as a built-in function.

The main motivation for this is that it allows to give better error messages
like line numbers when some part of the pipeline fails:
Currently `lib.pipe` internally uses a fold over the list,
therefore any type mismatches will give a trace which points into `lib.fold`,
leaving the user without the information at which stage of the pipeline it failed.
(This is less of a problem when used in packages, but significant enough that currently,
`lib.pipe` unfortunately should not be used in the implementation of any library functions.)
This could probably be fixed within Nixpkgs alone,
however not without incurring a significant performance penalty for using "reflection".
A built-in operator would be able to provide this more detailed error information basically for free.

Additionally, it allows easy usage outside of Nixpkgs and increases discoverability.

While Nixpkgs is bound to minimum Nix versions and thus `|>` won't be available until
several years after its initial implementation,
it can directly benefit from `builtins.pipe` and its better error diagnostic by overriding `lib.pipe`.
Elevating a Nixpkgs library function to a builtin has been done several times before,
for example `bitAnd`, `splitVersion` and `concatStringsSep`.

The main drawback is that once `|>` is available, there is little use for `builtins.pipe` anymore,
so the main purpose of that would be as a stop-gap for Nixpkgs
until the minimum Nix version is sufficiently high to allow using `|>`.

# Drawbacks
[drawbacks]: #drawbacks

- Introducing `|>` has the drawback of adding complexity to the language, and it will break older tooling.

# Unresolved questions
[unresolved]: #unresolved-questions

- What is the precise binding strength of the operator?
- Who is going to implement this in Nix?
- How difficult will the implementation be?
- Will this affect evaluation performance in some way?
  - There is reason to expect that replacing `lib.pipe` with a builtin will reduce its overhead,
    and that the builtin should have little to no overhead compared to regular function application.

In order to decide which operators to add to the language (see Alternatives),
a larger survey across the Nixpkgs code will be conducted.
This will give us quantitative information to better make any decisions involving tradeoffs.

# Future work
[future]: #future-work

Once introduced and usable in Nixpkgs, existing code may benefit from being migrated to using these features.
Automatically transforming nested function calls into pipelines is unlikely,
as doing so is not guaranteed to always be a subjective improvement to the code.
It might be possible to write a lint which detects opportunities for piping, for example in nixpkgs-hammering.
On the other hand, the migration from `pipe` to `|>` should be a straightforward transformation on the syntax tree.
