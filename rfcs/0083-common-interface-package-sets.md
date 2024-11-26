---
feature: common-interface-package-sets
start-date: 2020-12-19
author: Frederik Rietdijk (@FRidh)
co-authors:
related-issues:
---

# Summary
[summary]: #summary

The Nixpkgs package set consists of a large amount of packages of which a
significant amount of grouped into sub package sets. This RFC recommends a
common interface for these package sets.

# Motivation
[motivation]: #motivation

The Nixpkgs package set consists of a large amount of packages of which a
significant amount of grouped into sub package sets. Sets exist for various
languages, frameworks and plugins.

They are typically grouped together for one of the following two reasons:
- clarity, e.g. because they're written in the same language or framework;
- necessity, e.g. for compatibility reasons.

Over time different methods for defining package sets were created. Currently
multiple methods are in use in Nixpkgs and pull requests are opened to modify
sub package sets interfaces from one kind to another. Not only is this confusing
for users but it also causes trouble; in some cases overriding of derivations
inside a set is not possible or cross-compilation is broken.

This RFC thus aims to unify the package sets to a common interface for the
following reasons:
- simplify usage of package sets and reduce confusion surrounding them;
- single approach for dealing with overrides;
- handle variants of a package set;
- ensure cross-compilation works.

Often one also wants to build an environment with an interpreter or main program
and some additional packages or plugins. This RFC will therefore also recommend
a function each package set should offer for doing so, when appliceable that is.

## Related issues

TODO: refer to these issues in correct place.

- Common override interface derivations https://github.com/NixOS/rfcs/pull/67
- Make PHP packages overrideable https://github.com/NixOS/nixpkgs/pull/107044
- Change in emacs interface https://github.com/NixOS/nixpkgs/pull/107152
- Package set for Octave https://github.com/NixOS/nixpkgs/issues/65398#issuecomment-743926570
- Python package set is not overrideable
  https://github.com/NixOS/nixpkgs/issues/44426
- Support `overrideScope'` in Python package set
  https://github.com/NixOS/nixpkgs/pull/105374
- Common `overrideArgs` for sub package sets
  https://github.com/NixOS/nixpkgs/pull/46842. May be resolved using
  `overrideAuto` in https://github.com/NixOS/rfcs/pull/67.

# Detailed design
[design]: #detailed-design

We will now look in more detail at what a common interface should offer.

## Attribute name of the package set: `fooPackages` versus `foo.pkgs`

Two different interfaces are common in Nixpkgs when referring to package sets:
- `fooPackages`
- `foo.pkgs`

TODO which one to pick? Consider also overriding of the interpreter or main program and how that should propagate. Consider also the function for generating variants, where you need to have a name under which your interpreter or main program is available in the subpackage set.
## Variants
Often multiple variants of a package set need to be created. E.g., in case of
emacs or Python there are different versions of the program and each of them
should have their own package set. For this reason it is important that one can
easily create a new variant

```nix
fooPackagesFor = foo: import ./foo-packages.nix { ... };
fooPackages_3_6 = fooPackagesFor foo_3_6;
```

## Set overriding
It should be possible to override packages in a sub package set and have the
other packages in the set take that override into account. To that end, a scope
is created

```nix
lib.makeScope pkgs.newScope (self: with self; {
  ...
}
```

that can be overridden using `overrideScope'`

```nix
fooPackages_3_6.overrideScope' overrides;
```

where `overrides` is of the form

```nix
(final: previous: { ... })
```

In case one uses overlays with Nixpkgs, one could now make the overrides
composible using

```nix
fooPackages_3_6 = fooPackages_3_6.overrideScope' overrides;
```

## Package overriding

Now that it is possible to override a set, a common interface to overriding the
packages inside the set is needed as well. This is treated in [RFC
67](https://github.com/NixOS/rfcs/pull/67).

## Cross-compilation

For cross-compilation it is important that `callPackage` in the sub package set
has access to the spliced versions of the sub package set. Until recently, there
were no spliced sub package sets in Nixpkgs, but support was added for Python
utilizing the `makeScopeWithSplicing` function. There is [room for
improvement](https://github.com/NixOS/nixpkgs/pull/105374).

An important aspect for making this work is that, when constructing the package
set, it needs to know its own top-level attribute.

...


To support nested package sets, the full attribute path is needed, including
dots.

```nix
`fooPackages_3_6.subFooPackages_2_5`
```

## Single function for creating a package set

To ease the creation of a package set, a single function is proposed for
creating a set given a main program and an attribute name.


```nix
makePackageSet = ...
```


A package set can then be created

```
fooPackagesFor = foo: callPackages

fooPackages =

```

## Composing an environment

Thus far we considered building a package set and overriding it. Somewhat
orthogonal yet still commonly needed, is a method to compose an environment of
packages inside the package set.

Starting with Haskell and later adopted by other package sets is a
`withPackages(ps: [...])` function that allows you to compose an environment with chosen
packages from the set.

This RFC recommends each package set should have such a function, in case it is
appliceable. An example of where it would not make sense to include such a
function is in case of a [Qt package
set](https://github.com/NixOS/nixpkgs/pull/102168) because you would not be
interested in having a Qt with libraries during runtime and nothing else.

Haskell introduced the function as part of its package set, that is, one uses
`haskellPackages.ghcWithPackages (ps: [...])`. Some time later Python added such
function, but as part of the interpreter `python3.withPackages(ps: [...])`.
Since then, many other sets also added `withPackages` as part of the
interpreter or main program.

Should the `withPackages` function be part of the main program or the package
set? If we override the package set using `overrideScope'`, then the updated
package set is visible to attributes of the package set, that is

```
(fooPackages.overrideScope' overrides).withPackages
```

and

```
(fooPackages.overrideScope' overrides).foo.withPackages
```

will consider the updates.

### `fooPackages` versus `foo.pkgs` again...

TODO

Unfortunately, having `withPackages` as part of the main program makes it
somewhat more difficult to use it when overrides are needed as well. One would have to write

```nix
(foo_3_6.pkgs.overrideScope' overrides).foo.withPackages(ps: [...])
```
Indeed, the `withPackages` of the main program inside the sub set needs to be used. Using

TODO the actual recommendation

# Examples and Interactions

## Create a package set

The following example shows how to create a package set

...

## Override a package set

The following example shows how to override a package set

```nix
fooPackages.overrideScope' (final: previous: {
  ...
})
```

## Compose an environment

The following example shows how to compose an environment

TODO

```
fooPackages.foo.withPackages(ps: [...])
```

or

```
fooPackages.withPackages(ps: [...])
```

## Deprecating old interfaces

It it is recommended that package sets adopt the new interface and deprecate
their current one. For compatibility reasons it may not be possible to use the
recommended functions to construct the new interface, requiring custom
solutions.

# Drawbacks
[drawbacks]: #drawbacks

By not recommending a common interface, package sets may continue to have
different interfaces and change even in opposite directions.

# Alternatives
[alternatives]: #alternatives

# Future work
[future]: #future-work

- Document the recommended interface in the Nixpkgs manual.
- Adopt the recommended interface in existing package sets in Nixpkgs.
- Encourage tools that create package sets, e.g. from lock files, to also adopt
  this interface.
