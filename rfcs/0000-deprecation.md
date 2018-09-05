---
feature: deprecation
start-date: 2018-08-25
author: Silvan Mosberger (@infinisil)
co-authors: (find a buddy later to help our with the RFC)
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

We propose to add a deprecation guideline and a set of accompanying functions and tools to deprecate functionality in nixpkgs. Instead of removing an attribute which would result in an `attribute missing` error for users, one can deprecate a function, which will then first throw a warning and later throw an error. This is intended to be used for anything accessible from `import <nixpkgs> {}`, including the standard library, packages, functions, aliases, package sets, etc.

# Motivation
[motivation]: #motivation

Currently nixpkgs doesn't have any standard way to deprecate functionality. When something is too bothersome to keep around, the attribute often just gets dropped (490ca6aa8ae89d0639e1e148774c3cd426fc699a, 28b6f74c3f4402156c6f3730d2ddad8cffcc3445), leaving users with an `attribute missing` error, which is neither helpful nor necessary, because Nix has functionality for warning the user of deprecation or throwing useful error messages. This approach has been used before, but on a case-by-case basis (6a458c169b86725b6d0b21918bee4526a9289c2f). (Todo: link to 1aaf2be2a022f7cc2dcced364b8725da544f6ff7 and find more commits that remove aliases/packages/function args).

Why are we doing this? What use cases does it support? What is the expected
outcome?

# Detailed design
[design]: #detailed-design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with the ecosystem to understand, and implement.  This should get
into specifics and corner-cases, and include examples of how the feature is
used.

## Discussion

### Scope

Backwards compatibility should only concern the intended usage of nixpkgs:

1. Anything accessible through the top-level attribute set `pkgs = import <nixpkgs> {}`
  a. Packages (`pkgs.hello`, `pkgs.gcc`, ...) including their resulting attribute sets
  b. The standard library (`pkgs.lib`)
  c. Functions (`pkgs.mkDerivation`, `pkgs.fetchzip`, ...)
  d. Other package sets (`pkgs.haskellPackages`, `pkgs.pythonPackages`) and their packages and functions
2. The standard library through `import <nixpkgs/lib>`
3. NixOS system evaluation `import <nixpkgs/nixos> {}`
  a. All NixOS options from modules in `<nixpkgs/nixos/modules/module-list.nix>`
  b. NixOS profiles in `<nixpkgs/nixos/modules/profiles>` (TODO: Couldn't these be implemented as a NixOS option?)

Point 3a is already dealt with through `mkRemovedOptionModule`, `mkRenamedOptionModule`, etc. and is therefore not in scope for this RFC (TODO: These however don't work in submodules -> options in submodules can't use them). Point 3b is very low-traffic and therefore neither in scope (see https://github.com/NixOS/nixpkgs/pull/32776). Point 2 is pretty much the same as 1b. We'll also not distinguish between standard library functions and other functions in `pkgs`.

We'll therefore limit ourselves to everything accessible through `import <nixpkgs> {}`. The two main categories and their compatibility properties are:

- Attributes: An existing attribute continues to exist.
- Functions: All accepted function arguments continue to be accepted.

Everything else you might assume of nixpkgs isn't meant to be kept compatible, such as how many elements are in `pkgs.hello.buildInputs` or that `pkgs.hello.outPath` stays the same, therefore we don't include them here. Function arguments are secondary to this RFC, as most functions are rarely changed (Todo: Maybe extend this RFC to function arguments or rename it to "attribute deprecation"). Therefore attributes and their existence will be the focus.

Examples:
- `pkgs.pythonPackages.pytest` continues existing
- `pkgs.fetchurl` continues existing
- A potential exception: `pkgs.fzf.bin` may stop existing in the future and you should rely on `pkgs.lib.getBin` for getting the binary output instead.

### Types of deprecation

There can be multiple ways of dealing with deprecation, each having different properties, which we will discuss here.

Desired deprecation properties

- User knows deprecation reason
- Old code can be removed
- External code continues to work
- The may be undone/changed without any UX inconsistencies. This means users shouldn't get a warning about deprecation when this might get undone later, e.g. when it gets decided that deprecation isn't wanted after all, or that a compatibility package can be added. Once a warning of deprecation has been issued, it should be deprecated.

Types of deprecation

Deprecation happens in a sequence of phases, which will correspond to the release cycle of nixpkgs.

- Warn(delayed): Don't warn at first, then warn, then throw. Meaning: "We don't know if we really want to deprecate it as of now"
- Warn: First warn, then throw. Meaning: "We're sure to deprecate it, but we'll keep it usable for now"
- Throw: Throw. Meaning: "It is deprecated and not supported anymore"
- Removal: Remove code instantly. Meaning: "It has been deprecated and we want to get our codebase rid of it now" or "This attribute wasn't even meant to be used in the first place"

Which types have which properties

| Property                                                     | Removal | Throw | Warn | Warn(delayed) |
| ---                                                          | ---     | ---   | ---  | ---           |
| User knows deprecation reason                                | No      | Yes   | Yes  | Yes           |
| Old code can be instantly removed                            | Yes     | Yes   | No   | No            |
| Users expressions continue to evaluate                       | No      | No    | Yes  | Yes           |
| The change can be undone/replaced without UX inconsistencies | No      | No    | No   | Yes           |

#### State transitions

The properties of the four deprecation types along with the properties they ensure lead us to a set of allowed transitions between them. We will use the following primitives to represent deprecation. Here `r` represents the current release as an integer.

- `val` signifies a non-deprecated value `val`. In Nix this corresponds to `val` itself
- `warn d n val` signifies deprecation since release `d` and unsupported since release `d + n`. In Nix this corresponds to a function like this:
    ```nix
    if r < d then val # The deprecation time is the future, don't emit any message right now
    else if r >= d + n then throw "Deprecated since ${d}, removed in ${d + n}" # The current time is after the time of the planned throw
    else builtins.trace "Deprecated since ${d}, will be removed in ${d + n}" val` # The deprecation time is not in the future and before the planned throw
    ```
- `throw d n` signifies a deprecation since release `d` and unsupported since release `d + n`, the same as `warn` without a `val`. In Nix this corresponds to `throw "Deprecated since ${d}, removed in ${d + n}`.
- `removed`, the attribute is removed. In Nix this corresponds to an `attribute missing` error.

Our deprecation types map to these primitives like this:

- Warn(delayed) is `warn d n val` with `r < d`, aka deprecate in the future
- Warn is `warn d n val` with `r == d`, aka deprecate now
- Throw is `throw d n`
- Removal is `removed`

This leads us to the following allowed state transitions between our primitives:

- `val` -> `warn d n val`, if `r <= d`, aka can't deprecate in the past
- `warn d n val` -> `val`, if `r < d`, aka no warning has been issued yet, we can safely remove the deprecation again without causing any inconsistencies
- `val` -> `throw d n`, if `d == r`, aka can't deprecate in past nor future, because we won't have the value anymore
- `warn d n val` -> `throw d n`, if `r >= d -> r >= d + n`, aka if a warning has been issued, we need to wait until the warning time has passed to transition to a throw. This is the same as just `r >= d + n`
- `warn d n val` -> `removed`, if `r >= d + n + long`, aka it should be a while before we can remove throw messages. `long` stands for the number of releases the `throw` should have been issued for.
- `throw d n` -> removed, if `r >= d + n + long`, aka it should be a while before we can remove throw messages. `long` stands for the number of releases the `throw` should have been issued for.

## Implementation

The implementation of such a deprecation function will purely based on the planned release version of the next release (e.g. 18.09) and release versions of past releases. An argument could then be made that deprecation dates should also just be a release version, however it appears to be more convenient and less error prone if we allow all year/month combinations. Deprecations can then be added simply by inserting the current or planned deprecation year and month, instead of having to figure out what the next (or later than next) release version is going to be. This will work correctly as long as the `<nixpkgs/.version>` file always contains a version number that's not in the past. This can indeed occur if the date of effective release stretches itself past the month of the release version, which isn't a big problem however, even if unnoticed (the deprecation will just be delayed by a release).

To make implementation more efficient and easier, a pair of release version year/month will be converted into an integer corresponding to the number of months since year 2000. This means the release version `"18.09"` will be handled as `18 * 12 + 9 = 225`. In the implementation variables that contain such a value will be named `...YearMonth`.

### Release tracking

To implement deprecation functions that can vary their behavior depending on the number of releases that have passed since initial deprecation, we need to track past releases. A file `<nixpkgs/lib/releases.nix>` should be introduced with contents of the form
```nix
map ({ release, ... }@attrs: attrs // {
  yearMonth = let
    year = lib.toInt (lib.versions.major release);
    month = lib.toInt (lib.versions.minor release);
    in year * 12 + month;
}) [
  {
    name = "Impala";
    release = "18.03";
    year = 2018;
    month = 4;
    day = 4;
  }
  {
    name = "Hummingbird";
    release = "17.09";
    year = 2017;
    month = 9;
    day = ??;
  }
  {
    name = "Gorilla";
    release = "17.03";
    year = 2017;
    month = 3;
    day = 31;
  }
  # ...
]
```

Additional fields can be added if the need arises. This file should be updated at date of release. It may also be updated already at branch-off time (TODO: Think about this some more).

### Deprecation function

A function `lib.deprecate` will be introduced. When an attribute should be deprecated, it can be used like follows:
```nix
{
  foo = lib.deprecate {
    year = 2018;
    month = 8;
    warnFor = 2;
    reason = "foo has been deprecated, use bar instead";
    value = "foo";
  };
}
```

Optionally, for convenience, a shorthand `lib.deprecate'` could be provided as well:
```nix
{
  bar = lib.deprecate' 2018 8 "bar is a burden to the codebase, refrain from using it" "bar";
}
```

Which will correspond to
```nix
{
  bar = lib.deprecate {
    year = 2018;
    month = 8;
    warnFor = 1;
    reason = "bar is burden to the codebase, refrain from using it";
    value = "bar";
  };
}
```

The basic implementation and meaning of arguments is a follows (Note: I didn't test this yet).
```nix
{
  releaseYearMonth = let
    year = lib.toInt (lib.versions.major lib.trivial.release);
    month = lib.toInt (lib.versions.minor lib.trivial.release);
    in year * 12 + month;
    
  allReleases = [ releaseYearMonth ] ++ map (r: r.yearMonth) releases;
  
  takeWhile = pred: list: ...;
    
  deprecate =
    { year # Year of deprecation as an integer
    , month # Month of deprecation as an integer
    , warnFor ? 1 # Number of releases to warn before throwing
    , reason # Reason of deprecation
    , value ? null # The value to evaluate to, may be null if the current release throws
    }@args:
    if warnFor < 0 then abort "the warnFor argument to lib.deprecate needs to be non-negative" else
    let
      deprecationYearMonth = (year - 2000) * 12 + month;
      depReleases = takeWhile (ym: ym > deprecationYearMonth) allReleases;
      location = let pos = unsafeGetAttrPos "reason" args; in
        pos.file + ":" + toString pos.line;
        
      isDeprecated = depReleases != [];
      deprecationRelease = if isDeprecated then last depReleases else
        releaseYearMonth + ((deprecationYearMonth - releaseYearMonth) / 6) * 6);
      isRemoved = depReleases > warnFor;
      removedRelease = if isRemoved then elemAt depReleases (length depReleases - warnFor) else
        releaseYearMonth + ((deprecationYearMonth - releaseYearMonth) / 6 + warnFor) * 6);
      prettyRelease = yearMonth: "${toString (div yearMonth 12)}.${toString (mod yearMonth 12)}";
      deprecationString = if isDeprecated then "Deprecated since ${prettyRelease deprecationRelease}" else
        "Will be deprecated in ${prettyRelease deprecationRelease}";
      removedString = if isRemoved then ", removed since ${prettyRelease removedRelease}" else
        ", will be removed in ${prettyRelease removedRelease}";
      message = "${deprecationString}${removedString} at ${location}. ${reason}";
    in
      # TODO: Improve message for warnFor = 0
      if ! isDeprecated then assert value != null; value # TODO: Colors and stuff?
      else if isRemoved then throw message
      else assert value != null; builtins.trace message value;
}
```

### Configuration

Sometimes one might wish to either silence certain warnings or turn on warnings for planned deprecations. To do this, the nixpkgs `config` attribute will be used:
```nix
import <nixpkgs> {
  config.deprecation = {
    warnPlannedDeprecations = true;
    warnOverrides = {
      foo.bar = false; # Turn off warning for foo.bar
      qux.florp = true; # Turn on warning for qux.florp if it's a planned warning for now
    };
  };
}
```

TODO: Could there warning overrides be used for more than just deprecations?

Implementation details: Add a `config` attribute to `lib` with the value `{}` and use it for the deprecation function. This value can be changed by using `lib.extends (self: super: { config = <newvalue>; } }`. Define `pkgs.lib` as `lib.extends (self: super: { config = config.deprecation; }`.

## Deprecation process

When a decision has been made to deprecate an attribute, one has to decide:
- Date of deprecation. Let's call it `d` here, meaning "deprecate after `d` time has passed from now on". This corresponds to the `year` and `month` arguments to the deprecation function.
- The number of releases it should still be evaluable for albeit with a warning, let's call it `w` here, meaning it should warn for `w` releases. This corresponds to the `warnFor` argument of the deprecation function.

A sane default is `d = 0, w = 1`, to deprecate immediately and warn for one release. In general, it depends on the situation. Some more interesting examples might include:

- Pinning an old release of a package `hello_0_1` could have a planned deprecation of 2 years after introduction (`d = 2 years`) such that it can be removed eventually.
- A package being unable to be built anymore because it would require big changes might be deprecated immediately with no warning phase (`d = 0, w = 0`), such that the value doesn't have to be kept around anymore.
- A change that will influence a lot of code and might be hard to migrate from could have an instant deprecation but a warning phase of 4 releases (`d = 0, w = 4`), such that people have a lot of time to migrate away from it.
- Aliases might be deprecated only after two years so people enabling all warnings can be alarmed of the eventual removal. The warning phase could then be about 4 releases long because aliases don't cost much anyways (`d = 2 years, w = 4`).
- A package that has a known end-of-support date such as Java 7 in July 2022 can have this reflected in the deprecation time.

This isn't main focus of this RFC, however we will address a controversial issues regarding this:

### Aliases

Aliases have a long history of being introduced due to either the old name being unfit, the attribute being moved to a package set (or moved from one) or naming convention changes (`_` vs `-`). The question here is: Is it even worth deprecating the old aliases? We look at advantages and disadvantages:

Advantages of removing aliases:
1. Nix evaluation will be faster and use less memory for everybody.
   Doing a quick analysis using `export NIX_SHOW_STATS=1` on nixpkgs de825a4eaacd and Nix 2.0.4 with either all or (almost) no aliases with [this diff](https://gist.github.com/7f647d15b27825c3cc4dc2079acddb39), these are the results for different commands executed in the nixpkgs root. Time has been measured with an initial unmeasured run and an average over multiple runs after that, standard derivation in parens, in seconds. Heap is deterministic.
    
    | Command                                                | time elapsed (aliases -> no aliases)       | total Boehm heap allocations (aliases -> no aliases) |
    | ---                                                    | ---                                        | ---                                                  |
    | `nix-instantiate --eval -A path`                       | (15 runs) 0.043 (0.0041) -> 0.042 (0.0042) | 4.12MB -> 3.89MB                                     |
    | `nix-instantiate -A openssl`                           | (15 runs) 0.095 (0.0060) -> 0.090 (0.0049) | 18.5MB -> 17.7MB                                     |
    | `nix-instantiate nixos/release.nix -A iso_graphical`   | (15 runs) 5.71 (0.19) -> 5.64 (0.10)       | 839MB -> 836MB                                       |
    | `nix-instantiate nixos/release-combined.nix -A tested` | (5 runs) 222.6 (6.1) -> 218.2 (2.3)        | 36.51GB -> 36.35GB                                   |

    Putting some statistical meaning into the timings using an alpha of 2.5% (z = 1.96) leads to a result of everything but the first command being statistically significant (maybe because it's the only --eval?). So we really got ourselves about 1-3% better speed, which is not much, but it's not nothing. Also 0.4-5% less memory consumption, depending on the expression.
2. Cruft can be removed, cleaner repository state. Not that significant, since aliases are in their own file in `<nixpkgs/pkgs/top-level/aliases.nix>` and are therefore very isolated from everything else.

Disadvantages of removing aliases:
1. Deprecation needed -> People will eventually see warnings, potentially lots of them
   Doing a quick analysis of alias introduction times using `cat pkgs/top-level/aliases.nix | rg '[0-9]+-[0-9]+-[0-9]+' -o | cut -d- -f-2 | sort | uniq -c`, we get an mean of 5.5 and a median of 3 new aliases per month over the last ~4 years. This means if aliases were to be deprecated at a constant rate, we'd see about 5 new deprecations every month, so about 30 every release. Now of course not everybody uses every nixpkgs attribute, so it will be less than that for individual users. Considering that aliases will only get introduced for popular packages (since unpopular ones very little people use to begin with, so aliases wouldn't get introduced a lot), I estimate that an average user will encounter about 0-2 new alias deprecations per release. This amount will of course be bigger with "power" users, using lots of different packages. Considering that there are quite a few people only upgrading their version every 1-2 years, this can add up.

Assigning subjective weights to these points: The first advantage receives a weight of 4, the second advantage a weight of 1, the disadvantage gets a weight of 20 because it's user-facing and annoying. If I didn't forget anything, this adds to a resulting 14 points for *not* deprecating aliases. In short, the cost of deprecating aliases is much bigger than the cost of leaving them in.

## Links

https://github.com/NixOS/nixpkgs/pull/32776#pullrequestreview-84012820

https://github.com/NixOS/nixpkgs/issues/18763#issuecomment-406812366

https://github.com/NixOS/nixpkgs/issues/22401#issuecomment-277660080

https://github.com/NixOS/nixpkgs/pull/45717

https://github.com/NixOS/nixpkgs/pull/19315

https://github.com/NixOS/nixpkgs/pull/45717#issuecomment-418424080

# Drawbacks
[drawbacks]: #drawbacks

Drawbacks of this approach in comparison to keep doing what has been done until now:
- Deprecating lots of attributes will increase evaluation time, as the conditions for warnings/throws need to be evaluated.
- The usage of the `deprecation` function has to be learned, it's not as simple as just removing the attribute, adding a trace or replacing it with a throw. Adding good docs will help for this.

# Alternatives
[alternatives]: #alternatives

- Not doing anything. keep doing deprecation on a case-by-case basis, judging the most appropriate action.
- Having deprecation be a 0/1 thing: Either it's not deprecated and doesn't throw an error, or it is deprecated and it throws an error. The obvious disadvantage of this is that when people upgrade their nixpkgs, they can't expect their code to still work, even if they had no single warning message in the last version.
- Not including the expected removal date in the warning message. The disadvantage of this is that it's just not as nice to the user. Seeing a time of removal gives the user a good sense of either "I need to fix this now" or "This can wait". The advantage of this would be a more flexible deprecation policy, enabling us to remove values at whatever time one wishes.

# Unresolved questions
[unresolved]: #unresolved-questions

- What about errors in library functions? Correct or keep backwards compatible? #41604
- What about function arguments?
- Introduce automatic capture of attribute path in nixpkgs for better error messages? Is this even possible?

# Future work
[future]: #future-work

From the scope defined above it should be possible to implement a program that creates an index over nixpkgs, containing every attribute path that's intended to be used (I have a prototype of this working). This can then be used for automatically verifying backwards compatibility for commits. This can also can be used for providing a tags file to look up where functions and attributes are located, which is a much needed feature for nixpkgs. Another possible application of this is to provide documentation for attributes not directly where they're defined, but in a `attribute-docs.nix` file.

Similarly, the above defined allowed state transitions can be used to check that commits or pull requests don't do invalid transitions. Also a periodic automated job could run to set deprecation values to `null` if the values aren't needed anymore. Related, one could implement a program that returns which attributes are in which deprecation stage, doing some statistics with them.