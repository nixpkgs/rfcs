---
feature: inherit-as-list
start-date: 2021-10-17
author: Ryan Burns (@r-burns)
co-authors: (find a buddy later to help out with the RFC)
shepherd-team: (names, to be nominated and accepted by RFC steering committee)
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

This RFC proposes a new Nix syntax `[ inherit (<attrset>) <attrnames>; ]`,
which constructs a list from the values of an attrset.

The goal is to provide a similarly-terse but more principled alternative
to the often-used `with <attrset>; [ <attrnames> ]`.

# Motivation
[motivation]: #motivation

It is currently cumbersome to create a list from the values of an attrset.
If one has an attrset `attrs` and wishes to create a list containing some of
its values, one could naively write:

```nix
[ attrs.a attrs.b attrs.c ]
```

To avoid typing `attrs` many times, one will typically use `with` instead:

```nix
with attrs; [ a b c ]
```

However, the `with` expression has many well-known drawbacks, such as
unintuitive shadowing behavior [1][2], prevention of static scope checking [3][4],
and reduced evaluation performance [3].

* [1] https://github.com/NixOS/nix/issues/490
* [2] https://github.com/NixOS/nix/issues/1361
* [3] https://github.com/NixOS/nixpkgs/pull/101139
* [4] https://nix.dev/anti-patterns/language#with-attrset-expression

Nonetheless, Nix expression authors are subtly guided toward the `with` form
because it is (or at least appears) simpler than any existing alternatives.
Some alternatives are suggested in
https://nix.dev/anti-patterns/language#with-attrset-expression, but as these
are more verbose and complex than `with`, they are rarely used.

The goal of this RFC is to provide a similarly-terse alternative which avoids
these drawbacks.

# Detailed design
[design]: #detailed-design

The proposed syntax is:

```
[ inherit (attrs) a b c; ]
[ inherit a b c; ]
```

These expressions are (respectively) syntactic sugar for:

```nix
builtins.attrValues { inherit (attrs) a b c; }
builtins.attrValues { inherit a b c; }
```

The `inherit` keyword is intentionally reused so as to not add any new
keywords to the Nix grammar, and to evoke the similarities with the
existing attribute-based inherit syntax.

As the `inherit` keyword is currently a syntax error inside a list expression,
a Nix interpreter which supports this new language feature will be compatible
with existing Nix code.

This RFC is implemented here: https://github.com/nixos/nix/pull/5402

For MVP functionality, only minor additions to `src/libexpr/parser.y` are
needed. If accepted, the changes to the Nix interpreter can be backported
to current releases if desired. Relevant language documentation and
third-party parsers/linters would also need to be updated.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

This would be useful for many languages and frameworks in Nixpkgs which
extract packages from a package set argument.

For example, `python3.withPackages (ps: [ inherit (ps) ...; ])` will serve as a
more fine-grained alternative to `python3.withPackages (ps: with ps; [ ... ])`.
This would apply similarly to `vim.withPlugins`, `lua.withPackages`, etc.

Certain list-typed `meta` fields could also make use of this feature, e.g.:
```
meta.licenses = [ inherit (lib.licenses) bsd3 mit; ];
meta.maintainers = [ inherit (lib.maintainers) johndoe janedoe; ];
```

List-inherits can be used multiple times in a list-expression, and intermingled
with ordinary list elements without needing to concatenate:

```
buildInputs = [
  a b c /* ordinary list elements */
  inherit (pkgsFoo) d e f;
  g h i /* ordinary list elements */
  inherit (pkgsBar) j k l;
];
```

Sinc attributes inherit from the current scope with `{ inherit a b c; }`,
this RFC proposes a corresponding list-inherit syntax which also inherits
from the current scope. So if the order of the ordinary elements in the
previous list is not significant, one could rewrite the list as:

```
buildInputs = [
  inherit a b c;
  inherit (pkgsFoo) d e f;
  inherit g h i;
  inherit (pkgsBar) j k l;
];
```


# Drawbacks
[drawbacks]: #drawbacks

* This will add complexity to the Nix grammar and any third-party tools which
  operate on Nix expressions.
* Expressions reliant on the new syntax will be incompatible with
  Nix versions prior to the introduction of this feature.

# Alternatives
[alternatives]: #alternatives

* Skillful use of `with` to avoid its drawbacks
* Fine-grained attribute selection via existing (more verbose) language
  features, such as `builtins.attrValues { inherit (attrs) a b c; }`

# Unresolved questions
[unresolved]: #unresolved-questions

How would this feature be adopted, if accepted?

# Future work
[future]: #future-work

Determine best practices regarding when this language construct should be used