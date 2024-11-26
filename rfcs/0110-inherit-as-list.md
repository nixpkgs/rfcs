---
feature: inherit-as-list
start-date: 2021-10-17
author: Ryan Burns (@r-burns)
co-authors: (find a buddy later to help out with the RFC)
shepherd-team: @synthetica9, @infinisil, @kevincox, @bobvanderlinden
shepherd-leader: @kevincox 
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

This RFC proposes a new Nix syntax `<attrset>.[ <attrnames> ]`,
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
* [4] https://nix.dev/recipes/best-practices#with-scopes

Nonetheless, Nix expression authors are subtly guided toward the `with` form
because it is (or at least appears) simpler than any existing alternatives.
Some alternatives are suggested in
https://nix.dev/recipes/best-practices#with-scopes, but as these
are more verbose and complex than `with`, they are rarely used.

The goal of this RFC is to provide a similarly-terse alternative which avoids
these drawbacks.

# Detailed design
[design]: #detailed-design

The proposed syntax is:

```
attrs.[ a b c ]
```

This expression is syntactic sugar for:

```nix
[ attrs.a attrs.b attrs.c ]
```

As the token `.` immediately preceding `[` is currently a syntax error,
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

For example, `python3.withPackages (ps: ps.[ ... ])` will serve as a
more fine-grained alternative to `python3.withPackages (ps: with ps; [ ... ])`.
This would apply similarly to `vim.withPlugins`, `lua.withPackages`, etc.

Certain list-typed `meta` fields could also make use of this feature, e.g.:
```
meta.licenses = lib.licenses.[ bsd3 mit ];
meta.maintainers = lib.maintainers.[ johndoe janedoe ];
```

Note that only simple textual attrnames are allowed in the square brackets.
For example, `pkgs.[ (openssl.overrideAttrs { patches = [ ... ]; }) ]`
is currently a syntax error, as is `pkgs.[ "${some_expression}" ]`,
`a.[ b.[ c d ] ]`, and `a.[ [ b c ] [ d e ] ]`.
Future RFCs may add additional support for useful idioms such as
`pkgs.[ python310 python310Packages.pytorch ]` on a case-by-case basis,
but that is not planned for this RFC.

For a comparison of other forms of syntax considered but not proposed
in this RFC, refer to the Alternatives section.

# Drawbacks
[drawbacks]: #drawbacks

* This will add complexity to the Nix grammar and any third-party tools which
  operate on Nix expressions.
* Expressions reliant on the new syntax will be incompatible with
  Nix versions prior to the introduction of this feature.

# Alternatives
[alternatives]: #alternatives

A number of alternatives have been considered, which can be roughly divided
into syntactic (introducing new syntax which requires changes to the Nix language
to parse) and non-syntactic.

## Comparison of syntactic alternatives:

Priority is given to syntax which would be "backwards compatible" with
existing Nix code, meaning that any existing code would be evaluated the
same under an interpreter supporting the new syntax. Conversely,
evaluating new code under an old interpreter which does not support the
new syntax would cause a syntax error.


| Syntax | Notes | Cons |
|---|---|---|
| `inherit (attrs) [ a b c ]` | Initial draft | ["confusing that for attribute sets, the inherit keyword is on the inside but here it is on the outside" -jtojnar](https://github.com/NixOS/rfcs/pull/110#discussion_r730527443) |
|  `[ inherit (attrs) a b c; ]` | 2nd draft, [proposed by jtojnar](https://github.com/NixOS/rfcs/pull/110#discussion_r730527443) | ["not very happy with reusing inherit here. If I read inherit I read 'right, an attribute-set'." -Ma27](https://github.com/NixOS/rfcs/pull/110#issuecomment-947517675) <br /> ["There currently is no separator in lists, and this would add that." -Synthetica9](https://github.com/NixOS/rfcs/pull/110#issuecomment-959114390) |
| `attrs.[a b c]` | 3rd (current) draft, [proposed by Synthetica9](https://github.com/NixOS/rfcs/pull/110#issuecomment-959114390) | Has ["ambiguities [...] which would have to be worked out" -Synthetica9](https://github.com/NixOS/rfcs/pull/110#issuecomment-971760508) <br /> ["what `[a b c]` means depends on if it is after `attrs.`" -kevincox](https://github.com/NixOS/rfcs/pull/110#discussion_r933500003) <br /> Needs care to limit scope of this rule, currently limited to single-identifier case (discussions [here](https://github.com/NixOS/rfcs/pull/110#discussion_r1001737515) and [here](https://github.com/NixOS/rfcs/pull/110#discussion_r1013099694)) |
| `[ with attrs \| a b c ]` | [Proposed by nrdxp](https://github.com/NixOS/rfcs/pull/110#discussion_r933570815) | ["Looks more foreign", "may also cause confusion with other languages" -kevincox](https://github.com/NixOS/rfcs/pull/110#discussion_r934516433) <br /> ["introduces a small inconsistency" vs attribute-set `inherit` -rehno-lindeque](https://github.com/NixOS/rfcs/pull/110#discussion_r973033159) |
| Exclusive `with`, `let with` | [Proposed by infinisil](https://github.com/NixOS/rfcs/pull/110#issuecomment-1319338335) and [later refined](https://github.com/NixOS/rfcs/pull/110#issuecomment-1319338335) | Requires some dynamic behavior to fully replace `with`, including nested usage [-oxalica](https://github.com/NixOS/rfcs/pull/110#issuecomment-1334537982), [-infinisil](https://github.com/NixOS/rfcs/pull/110#issuecomment-1334593541) <br /> May change semantics of existing Nix code |

Some common threads here are the desire to introduce a syntax form which
is simpler and more ergonomic than existing `with` or alternatives,
naturally guiding users toward a "safer" form. We also desire consistency,
reusing keywords or syntax patterns but only where it would be
harmonious with the existing Nix language.

## Comparison of non-syntactic alternatives:

Other alternatives have been proposed where the motivation for `with`
deprecation is acknowledged, but would be resolved without introducing
new syntax to the Nix language.

| Alternative | Notes | Cons |
|---|---|---|
| `builtins.attrValues { inherit (attrs) a b c; }` | Considered in initial draft | Verbose, cumbersome to compose, not order-preserving |
| `select [ "a" "b" "c" ] attrs` | [Proposed by ocharles](https://github.com/NixOS/rfcs/pull/110#issuecomment-952704340) | ["highlights wrong (strings are not data but literal variable names)" -7c6f434c](https://github.com/NixOS/rfcs/pull/110#issuecomment-952817547) <br /> ["`with` is slightly more ergonomic", "the proposed change is arguably same [...] but without semantical gotchas" -7c6f434c](https://github.com/NixOS/rfcs/pull/110#issuecomment-952931398) |
| Deprecation of list-types in NixOS modules and build inputs | [Proposed by infinisil](https://github.com/NixOS/rfcs/pull/110#issuecomment-959757180) | ["order of `buildInputs` is significant so unordered sets cannot be used" -jtojnar](https://github.com/NixOS/rfcs/pull/110#issuecomment-959799730) |

# Unresolved questions
[unresolved]: #unresolved-questions

How would this feature be adopted, if accepted?

# Future work
[future]: #future-work

Determine best practices regarding when this language construct should be used
