---
feature: feature_parameter_names
start-date: 2024-01-10
author: Dmitry Bogatov
co-authors: (find a buddy later to help out with the RFC)
shepherd-team: (names, to be nominated and accepted by RFC steering committee)
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

Nixpkgs has mechanisms to exports compile-time options of the upstream build
system through the named parameters of derivation function, but the names of
these parameters are inconsistent from package to package and from feature to
feature.

# Motivation
[motivation]: #motivation

Developer interface for building environment with specific features enabled and
disabled is more complicated and and involves more "you just have to know it"
compared to already existing systems.

1. Writing generic code is complicated. Even if list of packages all have
   optional dependency on package `foo`, duck-typing like

```
map (p: p.override { with_foo = false; }) packages
```

won't work because of lack of uniformity, leading to ugly code downstream.

2. If user wants to globally disable support for optional feature or build
   dependency as much as possible, he needs to set multiple feature parameters,
   and he still can miss some. E.g, it is not obvious whether the following
   list is enough to cover everything X11-related:

```
withX
withX11
x11Support
enableX
```

3. There is no clear separation between feature parameters and other arguments
   to the package. With naming mandated by this RFC, distinction is clear to
   cater to automatic queries.

4. Side effect of the proposdd migration plan is compiling of almost
   exahustive list of known feature parameters.


# Detailed design
[design]: #detailed-design

When upstream build system has options to enable or disable a optional features
or configure some compile-time parameter, like path to default mailbox or
initial size of hash map, derivation function (for short, package) MAY expose
compile-time options via feature parameters that match
`^(with|conf|enable)_[a-z_0-9]$` regular expression. For short, we will refer
to them as `with`-parameter, `conf`-parameter and `enable`-parameter correspondingly,
collectively as `feature parameters`.

Note that this RFC does not establish any requirements whether package should or
should not have any feature parameters, this decision is left at descretion of
the package maintainer(s). This RFC only concerns itself with the naming of the
feature flags. Further text will use wording "corresponding feature parameter"
with implicit caveat "provided that package maintainer decided to export
underlying build system configuration option".

Feature parameters MUST only be used according to the rules outlined in this
RFC. They MUST NOT be used for any other purpose.

Boolean feature parameter MUST be either `with`-parameter or `enable`-parameter.
Non-boolean (e.g. string or integer) feature parameter MUST be an `conf`-parameter.

Feature parameter that incurs additional runtime dependency MUST be a `with`-parameter.

<details><summary>Example</summary>

```
{ lib, stdenv, mpfr, with_mpfr ? true }:

stdenv.mkDerivation {
  # ... snip ...
  configureFlags = lib.optionals with_mpfr ["--with-mpfr"];
  buildInputs = lib.optionals with_mpfr [ mpfr ];
  # ... snip ...
}
```

</details>

As special provision, if optional vendored dependency is exposed by feature
parameter, it MUST be `with`-parameter.

If feature parameter does not incur extra runtime dependencies, corresponding
feature parameter MUST be `enable`-parameter and its name SHOULD correspond to
the upstream naming, unless is causes major inconsistency with other packages.
Consistency of feature parameters across nixpkgs is more important than
matching to the names used by particular upstream. What constitutes a major
inconsistency is left at discretion of the package maintainer(s).

<details><summary>Example</summary>

If upstream uses `--enable-translations` to denote support for NLS,
corresponding feature parameter SHOULD be `enable_nls`, since it is much more
common term for enabling programs to produce output in non-English languages.

</details>

<details><summary>Rationale</summary>

This distinction between `with` and `enable` feature parameters is based on Autotools naming conventions.

[Autoconf ENABLE](https://www.gnu.org/software/autoconf/manual/autoconf-2.66/html_node/Package-Options.html)
[Autoconf WITH](https://www.gnu.org/savannah-checkouts/gnu/autoconf/manual/autoconf-2.72/html_node/External-Software.html)

</details>

If upstream build features and build dependencies do not map one-to-one, then
one `with`-parameter SHOULD be added for every build dependecy and one
`enable`-parameter SHOULD be added for every upstream build feature intended to
be optional. Assertions to preclude incoherent feature configurations MAY be
added.

<details><summary>Example</summary>

```
{ lib
, stdenv
, openssl
# At maintainer(s) discretion some coherent configuration is chosen.
, with_openssl ? true
, wolfssl
, with_wolfssl ? false
, enable_ssl ? with_openssl || with_wolfssl
}:

# Assertions are optional and might be infeasible when package has huge amount
# of feature flags, but in general improve user experience.
assert enable_ssl -> with_openssl || with_wolfssl;

stdenv.mkDerivation {
  # ... snip ...
  configureFlags = lib.optionals enable_ssl ["--enable-ssl"];
  buildInputs = lib.optionals with_wolfssl [ wolfssl ]
             ++ lib.optionals with_openssl [ openssl ];
  # ,,, snip ...
}

```

</details>

Maintainer(s) MAY choose to add both `with` and `enable` feature flags even if
they map one-to-one for consistency with other packages.

<details><summary>Example</summary>

```
{ lib
, stdenv
, openssl
, with_openssl ? true
, enable_ssl ? with_openssl
}:

assert enable_ssl -> with_openssl;

stdenv.mkDerivation {
  # ... snip ...
  configureFlags = lib.optionals enable_ssl ["--enable-ssl"];
  buildInputs = lib.optionals with_openssl [ openssl ];
  # ,,, snip ...

  passthru.features = {
    inherit with_openssl enable_ssl;
  };
}
```

</details>

The rules are to be enforced by static code analyse linter to be written. Since
no static code analyzis is perfect, it shall have support for inhibiting
warnings in unsual cases.

All feature parameters SHOULD be exported in `passthru.features` set. See example above.

Due overwhelming amount of possible combinations of feature flags for some
packages, nixpkgs maintainer is not expected to test or maintain them all, but
SHOULD accept provided technically sound contributions related to
configurations with non-default feature flags. Only configurations that has
name in package set (e.g `emacs-nox`) SHOULD be built on CI.

## The migration process.

### Orgranization rules

By the very nature of this RFC, once it passes, numerous packages in Nixpkgs
will become non-compliant. This is unavoidable, otherwise this RFC would not be
necessary. The following organizational process shall be followed to ensure
that nixpkgs will eventually become compliant with this RFC:

* In existing packages, maintainers MAY start to expose feature parameters and MAY cease exposing them.
* Further contributions MAY change their package to be compliant with the RFC
* Further contributions MUST NOT rename non-compliant feature parameters to another non-compliant name.
* Further contributions MUST NOT rename compliant feature parameters to non-compliant parameter.
* Further contributions MAY rename compliant feature parameters to another compliant name
  for the purposes of consistency with other packages.
* Further contributions that improve compliance with this RFC MAY involve
  maintainers but SHOULD NOT be required to do so because the rules SHOULD be
  clear.
* New packages MUST be compliant with the RFC (either no feature parameters or compliant with this RFC.)

This ensures that count of non-compliant feature parameters in nixpkgs is
non-increasing over the time.

### Documentation changes

* Documentation of [code style](https://github.com/NixOS/nixpkgs/blob/master/CONTRIBUTING.md#syntax) need to
  be updated to mention that feature parameters use snake case convention unlike other variables that use
  `lowerCamelCase'.

### Technical process

Renaming the parameters is handled automatically by the `makeOverrideable`
function, as implemented in [https://github.com/NixOS/nixpkgs/pull/284107](#284107).

Renaming `nonCompliantFoo` into `enable_foo` involves two steps:

 1. Mechanical renaming all mentions of `nonCompliantFoo` in package
    definition.
 2. Adding renaming rule into
    [https://github.com/KAction/nixpkgs/blob/contrib/0/rfc169-proof/out/lib/rfc0169.json](makeOverrideable)
    configuration, if it is not there already.

## Feature parameter naming guidelines

Rules in the first section of this RFC describe different scenarios when
`with`, `enable` and `conf` feature parameters must be used, but coming with
exact name that is clear, concise, describes it effect correctly and is
consistent across the nixpkgs is hard to codify with absolute rule. Instead,
this RFC provides set of guidelines and examples based on scenarios brought up
during the discussion.

Feature flags that require single external dependency SHOULD be named after
that dependency. Prefix `lib` SHOULD be removed. For example,

<details><summary>Example</summary>

```
systemd => with_systemd
libgif  => with_gif
curl    => with_curl
alsa    => with_alsa

# When people say "Xorg" support, they usually mean linking "X11"
# library which is one of the ways to talk X protocol to X server. "xcb"
# being another one.
xorg    => with_x11
xcb     => with_xcb
Qt5     => with_qt5

# If upstream feature name contains multiple words (e.g --enable-frame-pointers)
# then feature parameters SHOULD be snake_cased after upstream.
--enable-frame-pointers => enable_frame_pointers

# If optional dependency package name has multiple words in it, feature
# name SHOULD follow the package attribute, but with dashes replaced
# with underscores.
pulseaudio => with_pulseaudio
dump-init => with_dump_init


```

</details>

When multiple feature flags require the same build dependency, for example
derivation has optional support for FTP and HTTP protocols, any of which
incur dependency on `curl`, the package would look like following:

<details><summary>Example</summary>

```
{ lib
, stdenv
, curl
, with_curl ? true
, enable_ftp ? true
, enable_http ? true
}:

# Assertions are fully optional.
assert enable_ftp -> with_curl;
assert enable_http -> with_curl;

stdenv.mkDerivation {
   # ... snip ...

   configureFlags = lib.optionals enable_http ["--enable-http"]
                 ++ lib.optionals enable_ftp ["--enable-ftp"];
   buildInputs = lib.optionals withCurl [ curl ];

   # ... snip ...

   passthru.features = {
      inherit with_curl enable_ftp enable_http;
   };

   # ... snip ...
}
```
</details>

When build dependency comes from particular package set, it MAY make sense to
name feature parameter after it. E.g build dependency on `qt6.qtbase` SHOULD
have `with_qt6` feature parameter. If that results in feature parameter name
that is too generic, package name from the set MAY be included into the feature
parameter name. Optional dependency on `pythonPackages.pillow` MAY have feature
parameter `with_python_pillow`.

Build dependency on bindings to low-level libraries SHOULD be named after
underlying library. For example, optional dependecy on `pyqt5` Python bindings
to `Qt5` library should have `with_qt5` feature parameter.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

This is how one can query list of feature parameters that incur extra runtime dependencies:

```
$ nix eval --json --impure --expr 'with import ./. {}; builtins.attrNames gnuplot.override.__functionArgs' | jq '.[]| select(startswith("with"))'
"withCaca"
"withLua"
"withQt"
"withTeXLive"
"withWxGTK"
```

I picked the `gnuplot` as example since it is the closest to be compliant with proposed rules.


# Drawbacks

1. The migration process involves renaming feature parameters, so these changes
   will likely conflict with other outstanding changes to the derivation, and potentially
   even with automatic version bumps.

2. While migration is in the process, nixpkgs source will be even less
   consistent than it is today.

3. Proposal migration plan slows down evaluation of all packages, including
   ones without any feature parameters by approximately 1.3%

# Alternatives
[alternatives]: #alternatives

## Do nothing

Avoids all the work and drawbacks, but there is no evidence that consistency
problem will [solve itself evolutionary](https://github.com/NixOS/nixpkgs/pull/234463#issuecomment-1574892207).
For some build dependencies, we have multiple equally popular feature parameter
names, and people keep picking random one when adding new packages.

## Do not address consistency issues across different features

For most features, nixpkgs currently uses multiple names. Instead of process
described in this RFC, we might rename parameters towards the most common one.
That would mean

```
dbus => dbusSupport
qt   => enableQt
sdl  => withSDL  (winner by one usage)
```

## Other ways to do migration

1. Put deprecation logic into individual packages. That would avoid evaluation
   penalty for already compliant packages and packages that has no feature
   parameters, but would be quite verbose and require sophisticated automations
   to introduce and drop deprecation logic.

## Other ways to pass feature parameters

### Group all feature parameters into separate attrset parameter

Instead of writing

```
{ lib, stdenv, enable_foo ? true }:

# uses enable_foo
stdenv.mkDerivation { ... }
```

derivation can be written as
```
{ lib, stdenv, features ? { enable_foo = true; }}:

# uses features.enable_foo
stdenv.mkDerivation { ... }
```

It is definitely looks cleaner, but unfortunately hits several limitations of
the Nix language.

1. Nix language provides way to [introspect function argument names](https://nixos.org/manual/nix/stable/language/builtins.html#builtins-functionArgs),
   but no way to learn their default values. Learning default values is fundamentally impossible, since default values of one parameter might depend
   on the value of another parameter, so default values can't be evaluated until function is called.

2. Overrides become much more verbose. Simple and ubiquitous

   ```
   bar.override { with_foo = true; }
   ```

   becomes unwieldy

   ```
   bar.override (old: old // { features = old.features // { with_foo = true }; })
   ```

   That can be simplified by introducing `overrideFeatures` function, but we
   already have [way too many](https://ryantm.github.io/nixpkgs/using/overrides)
   `override` functions. Also, this version will silently override nothing in case
   of typo.

Any approach that tries to pass attribute set to function will have these
issues. Using existing [config.nix](https://github.com/NixOS/nixpkgs/blob/master/pkgs/top-level/config.nix)
is no different.

This could be viable and actually more elegant solution if Nix to be extended
to be able to introspect default parameter values, but that would make nixpkgs
incompatible with older installations of the Nix.

### Use `foo ? null` instead of `with_foo` feature parameters

Patter of using `foo ? null` where `foo` is a package, and handling
`foo == null` as request to build without optional dependency on `foo`
is already used in some places, but it has following drawbacks, so this RFC
mandates `with_foo` approach:

* Such pattern only applies to with parameters, so it is inconsistent
  with `enable_*` and `conf_*` parameters.

* Disabling support for something by default requires setting the override
  at `callPackage` site, spreading package definition across multiple files.

* This pattern makes code than wants to access attributes of `foo`
  more complicated:

```
assert (foo == null) or (versionAtLeast foo.version "1.0");
# versus simpler version that works even when "with_foo = false".
assert (versionAtLeast foo.version "1.0");
```


## Other ways to name parameters

Other naming conventions were considered and found less appealing.

1. Lisp-style `with-foo` variable names are inconvenient when they need to be
   passed to the builder environment as opposed being used exclusively at
   evaluation phase, since Bash does not support referring to environment
   variables that are not C identifier.

2. Camel case is the most popular naming convention in Nixpkgs at the time of
   writing, but adding prefix results in quite unnatural-looking identifiers
   like `enableSsh` or `withLlvm`. In addition, proposed convention visually
   distinguish feature parameters from other kinds of parameters.

3. Camel case, but allowing to follow upstream spelling, produces naturally
   looking identifies like `enableSSH`, `withLibreSSL`, `enableAudio` that are
   wildly inconsistent between each other.

# Prior art
[prior-art]: #prior-art

As mentioned in motivation part, the best in class of feature flag
configuration system is Gentoo Linux:

- [https://www.gentoo.org/support/use-flags]

There is not much work I can find about this problem in Nixpkgs other than my
previous attempts to solve it on a  per package basis:

- https://github.com/NixOS/nixpkgs/issues/148730
- https://github.com/NixOS/nixpkgs/pull/234463

# Unresolved questions
[unresolved]: #unresolved-questions

This RFC makes it possible to introspect feature parameters of particular
derivation, and makes recomendations to keep feature parameter names consistent
across whole nixpkgs, but still does not provide simple and efficient way to
list all existing feature parameters.

# Future work
[future]: #future-work

There are other configuration scenarios not covered by this RFC:

- Optional dependencies in shell wrappers (e.g [passage](https://github.com/NixOS/nixpkgs/blob/master/pkgs/tools/security/passage/default.nix#L12)).

- Finding way to get list of all existing feature parameters, like [Gentoo does](https://gitweb.gentoo.org/repo/gentoo.git/tree/profiles/use.desc).
  That can be possibly done by building and distributing the index separately, like [nix-index](https://github.com/nix-community/nix-index) does it,
  or requiring every feature parameter used in any package to be listed in some index file kept under version control in nixpkgs.

- Meta-feature-flags, that would allow user to set, e.g `enable_gui = false` on
  the top level of the overlay, and that would disable support for X11, Qt4,
  Qt6, GTK2, GTK3, Motif and all other graphical libraries and toolkits for all
  packages that support doing so.

- Exhaustive list of all existing feature parameters, enabling user to
  review and configure all feature parameters available in one session
  instead of researching feature parameters on per-package basis. That
  would be equivalent of Gentoo's ability to set use flags in
  [/etc/portage/make.conf](https://wiki.gentoo.org/wiki//etc/portage/make.conf#USE).


# Changelog

<details><summary>Folded changelog</summary>

1. Changed wording to not imply that every upstream build system knob SHOULD be
   exported via feature parameters. (Thx: 7c6f434c)

2. Relaxed wording on the name of feature parameters to avoid painting ourselves
   into ugly and non-intuitive names. (Thx: 7c6f434c)

3. Fix typo in regex to be consistent that feature flag name can't have small
   letter after `with|conf|enable` prefix. (Ths: don.dfh)

4. Explicitly mention that static code analysis has support for overrides based
   on human judgement call. (Thx: 7c6f434c)

5. Clarify solution scenarios when build inputs and feature flags don't match
   one-to-one. (Thx: Atemu, 7c6f434c)

6. Refine the deprecation plan to make sure the warning includes the sunset
   timestamp. (Thx: pbsds)

7. Add rules about non-boolean feature parameters. (Thx: Atemu, pbsds)

8. Set expectations for building and maintaining multiple configurations. (Thx: pbsds)

9. Removed non-boolean parameters from "Future Work" section.

10. Relaxed requirements for assertions about conflicting flags (Thx: Atemu)

11. Add guideline so `pythonPackages.pillow` does not get `withPython` feature name. (Thx: 7c6f434c)

12. Mention that vendored dependenices are still `with`. (Thx: 7c6f434c)

13. Elaborate on camel case convention. (Thx: 7c6f434c)

14. Explicitly mention that what feature parameters to introduce is left at discretion of the maintainer (Ths: Atemu)

15. Add clause about passthrough `features` set.

16. Switch to `snake_case` since it looks more consistent and less ugly than alternatives.

17. Mention meta-parameters in the "Future Work" section.

18. Elaborate on benefits if Nix were to allow introducing of default values.

19. Downgrade "MUST be clear" to "SHOULD be clear". This way we won't need another RFC
    for some corner case.

20. Mention other ways to do (or not to do) the migration.

21. Fix incorrect link to the autoconf manual.

22. Avoid word "usually" in the summary. Many (most?) packages has no upstream feature flags.

23. Remove side note about other independent implementations of X protocol.

24. Change the migration plan to do renaming on the `makeOverrideable` level.

25. Add link to the nix variable name code style. (Thx: @h7x4)

26. Drop motivation part about mechanical translation of upstream parameter names. (Thx: infinisil)

27. Drop mention of grace period, since it is not necessary with new migration plan.

28. Document the `foo ? null` pattern and its drawbacks (Thx: Atemu)

</details>
