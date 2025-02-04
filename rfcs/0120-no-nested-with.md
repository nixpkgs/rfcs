---
feature: no-nested-with
start-date: 2021-01-11
author: Ellen Emilia Anna Zscheile (@fogti)
co-authors: @sternenseemann
shepherd-team: (names, to be nominated and accepted by RFC steering committee)
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

Disallow or discourage usage of multiple `with` expressions covering
the same expression / forbid nesting `with` expressions, even indirectly.
If infeasible in general (e.g. forbidding it in all nix expressions),
this can be limited to nixpkgs.

# Motivation
[motivation]: #motivation

It makes static analysis of nixpkgs easier, because as soon as `with`
expressions are nested, it becomes basically impossible to [statically
deduce where any free variable comes from] without implementing a
full-blown nix evaluator including lazy evaluation, which is difficult as
soon as `with` expressions and mutually recursive imports are involved
(e.g. as currently present in `nixpkgs/lib/systems/{inspect,parse}.nix`).

# Detailed design
[design]: #detailed-design

At least warn about any nested usage of `with` expressions,
at least when they get evaluated, possibly even when they get parsed.
After a grace period ("real time", probably a few months or years, not as in
"a timeout when interpreting" because that wouldn't make sense),
abort instead.

When the full approach (aborting instead of warning) is applied,
any reference to any free variable can be easily resolved
to the enclosing `with` expression "scope-include", and because
this `with` expression couldn't then be enclosed by another
one, even indirectly, no lookup ambiguity exists.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

```nix
pkgs: {
  # allowed
  a = with pkgs; patchelf;

  # disallowed
  b = with pkgs; with lib; patchelf;

  # also disallowed
  c = with pkgs; {
    meta = with lib; {
      license = with licenses; [ mit ];
    };
  };
}
```

# Drawbacks
[drawbacks]: #drawbacks

* Backward-incompatible, requires changing Nix code used "in the wild".

# Alternatives
[alternatives]: #alternatives

* Introduce a kind of `with-only` expression which allows bringing an attrset
  into scope while simultaneously hiding the outer scope, such that all
  inner free variables are either resolved via the given attrset, or
  result in an error.
  Similar to this is [RFC 110](https://github.com/NixOS/rfcs/pull/110),
  which introduces an `inherit-as-list` construct which solves the primary
  application of `with-only`, list specification in `meta` attributes of
  nixpkgs derivatons.

* Completely ban the usage of `with` in nixpkgs;
  seems unnecessarily excessive for this problem.

# Unresolved questions
[unresolved]: #unresolved-questions

Decide if this is enough.

e.g.
* Mutually recursive imports combined with `with` expressions also make static
  analysis harder, because they require lazyness at the level of scope lookups,
  which is difficult to implement corrently. e.g. [nix2js#2](https://github.com/YZITE/nix2js/issues/2)
* Check if any use case is too much negatively impacted by this.

# Future work
[future]: #future-work

* Static analyzers and transpilers for Nix
