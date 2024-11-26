---
feature: Declarative Wrappers
start-date: 2020-08-16
author: Doron Behar <me@doronbehar.com>
co-authors: (find a buddy later to help out with the RFC)
shepherd-team: @FRidh, @lheckemann, @edolstra
shepherd-leader: @FRidh
related-issues: POC Implementation at [#85103](https://github.com/NixOS/nixpkgs/pull/85103).
---

# Summary
[summary]: #summary

Manage the environment of wrappers declaratively and deprecate shell based
methods for calculating runtime environment of packages. Make it easier to
debug why env vars are added to an executable, by storing this information
inside `/nix/store/.../nix-support/` of the dependencies that using them
requires runtime environment. Create a new `makeWrapperAuto` hook that will
make the `fixupPhase` read all of the deps environment that's needed and
automatically wrap the executables with the proper environment.

# Motivation
[motivation]: #motivation

Every Nix build produced is stored in a separate path in the store. For a
build to find its runtime dependencies purely, they need to be hardcoded in
the build. Thus, a complete dependency specification is needed.

In case of compiled languages this process is already largely automated;
absolute paths to shared libraries are encoded in the `rpath` of binaries.
Executables that are invoked by binaries need to be dealt with manually,
however. The preferred choice is here to manually patch the source so that
binaries are invoked using absolute paths as well.

This is not always trivial and thus a common work-around is to wrap executables,
setting `$PATH` to include the locations of dependencies. This is a pragmatic
solution that typically works fine, but it comes with a risk: *environment
variables leak*. Executables that shell out may pass along the modified
variables, causing at times unwanted behaviour.

Programs written in interpreted languages tend to import their runtime
dependencies using some kind of search path. E.g., in case of Python there is a
process for building up the `sys.path` variable which is then considered when
importing modules. Like with compiled languages, the preferred choice would also
here be to embed the absolute paths in the code, which is often not done. Note
that in case of Python this *is* done. Like with compiled languages, programs
may shell out and likewise the preferred solution is to patch the invocations to
use absolute paths. Similarly, in case a programs wants to `dlopen` a shared
library this should be patched to include an absolute path, instead of using
`LD_LIBRARY_PATH`.

It is recognized that wrappers setting environment variables are typically not
the preferred choice because of the above mentioned leakage risk, however, often
there is simply not a better or reasonable alternative available.

We have numerous issues regarding wrappers and our wrapper shell hooks. Here's
a list of them, sorted to categories.

* [Missing environment variables](#missing-environment)
* [Orchestrating wrapping hooks](#orchestrating-wrapping-hooks)
* [Issues _Possibly_ fixable by declarative wrappers](#issues-possibly-fixable-by-declarative-wrappers-)
* [Unreported Issues](#unreported-issues-afaik)
* [Other Issues](#other-issues)

## Missing environment

- [pull 83321](https://github.com/NixOS/nixpkgs/pull/83321)
- [pull 53816](https://github.com/NixOS/nixpkgs/pull/53816)

@rnhmjoj & @timokau How unfortunate it is that Python's `buildEnv` doesn't know
to do anything besides setting `NIX_PYTHONPATH` - it knows nothing about other
env vars, which some deps rely upon when eventually used. Declarative wrappers
don't care about the meaning of env vars - all of them are treated equally,
considering all of the inputs of a derivation equally.

- [pull 75851](https://github.com/NixOS/nixpkgs/pull/75851)
- [issue 87667](https://github.com/NixOS/nixpkgs/issues/87667)

Fixable with our current wrapping tools (I guess?) but it's unfortunate that we
have to trigger a rebuild of VLC and potentially increase it's closure size,
just because of a missing env var for only _some_ users. If only our wrapping
requirements were accessible via Nix attrsets, we could have instructed our
modules to consider this information when building the wrappers of the packages
in `environment.systemPackages`.

- [issue 87883](https://github.com/NixOS/nixpkgs/issues/87883) (Fixed)

@jtojnar wouldn't it be wonderful if the wrapper of gimp would have known
exactly what `NIX_PYTHONPATH` to use when wrapping gimp, just because `pygtk`
was in it's inputs? Declarative wrappers would also allow us to merge the
wrappings of such derivation to reduce double wrappings, as currently done at
[`wrapper.nix`](https://github.com/NixOS/nixpkgs/blob/b7be00ad5ed0cdbba73fa7fd7fadcb842831f137/pkgs/applications/graphics/gimp/wrapper.nix#L16-L28)
and
[`default.nix`](https://github.com/NixOS/nixpkgs/blob/b7be00ad5ed0cdbba73fa7fd7fadcb842831f137/pkgs/applications/graphics/gimp/default.nix#L142-L145).

- [issue 85306](https://github.com/NixOS/nixpkgs/issues/85306)
- [issue 84249](https://github.com/NixOS/nixpkgs/issues/84249)

`git-remote-hg` and `qttools` are not wrapped properly.

- [issue 86048](https://github.com/NixOS/nixpkgs/issues/86048)
- [issue 114051](https://github.com/NixOS/nixpkgs/issues/114051)

I guess we don't wrap HPLIP because not everybody want to use these binaries
and hence want these GUI deps in their closure (if they were wrapped with a
setup hook)? Declarative wrappers would allow _some_ users to use the wrapped
binaries and others not to need this wrapping. Via an override or a NixOS
config flag, without triggering a rebuild of HPLIP itself, these users would be
easily satisfied.

## Orchestrating wrapping hooks

- [issue 78792](https://github.com/NixOS/nixpkgs/issues/78792)

@worldofpeace you are correct. All of these setup-hooks are a mess. At least we
have documented, (yet not totally implemented) [this section of the
manual](https://github.com/NixOS/nixpkgs/blob/2df97e4b0ab73f0087af2e6f33e694140150db1b/doc/languages-frameworks/gnome.section.md#L120-L166)

Declarative wrappers will deprecate the usage of our shell based hooks and will
wrap all executables automatically according to their needs, without requiring
the contributor a lot of knowledge of the wrapping system. Also, double
wrappings will become a problem of the past.

- [issue 86369](https://github.com/NixOS/nixpkgs/issues/86369) 

@ttuegel with declarative wrappers, we can symlink all qt plugins into 1
directory and wrap the executable with only 1 `QT_PLUGIN_PATH` in their
environment, which should decrease the plugin load of every qt package.

## Issues _possibly_ fixable by declarative wrappers (?)

- [pull 61213](https://github.com/NixOS/nixpkgs/pull/61213) 

I'm not sure what's the issue there. But, I'm sure that a declarative, Nix
based builder of a Python environment, even if this environment is used only
for a build, should make it easier to control and alter it's e.g `$PATH`.

- [issue 49132](https://github.com/NixOS/nixpkgs/issues/49132)
- [issue 54278](https://github.com/NixOS/nixpkgs/issues/54278)
- [issue 39493](https://github.com/NixOS/nixpkgs/issues/39493)

`GDK_PIXBUF` compatibilities? I haven't investigated them to the details, so feel
free @jtojnar to review me and tell me that declarative wrappers won't help.

## Unreported issues (AFAIK)

Issues that bother me personally, but I haven't bothered to open an issue since
I doubt it would be feasible to fix with our current wrapping ecosystem, excuse
my pessimism `wrap{G,Qt}AppsHook` authors.

`kdeconnect` has `kdoctools` in it's closure because it's wrapper has
`kdoctools` due to it picked by `wrapQtAppsHook`:

```
$ nix why-depends -f. kdeconnect kdoctools
/nix/store/sh42k6cz4j48br4cxi2qn173rys4japp-kdeconnect-1.3.5
╚═══bin/kdeconnect-cli: …xport XDG_DATA_DIRS='/nix/store/m16681i5dhhkhszi9w42ir037jvbnab9-kdoctools-5.71.0/share'${XDG_DA…
    => /nix/store/m16681i5dhhkhszi9w42ir037jvbnab9-kdoctools-5.71.0
```

A similar issue is with `kconfigwidgets.dev` and `kdeconnect`:

```
$ nix why-depends -f. kdeconnect kdeframeworks.kconfigwidgets.dev
 /nix/store/sh42k6cz4j48br4cxi2qn173rys4japp-kdeconnect-1.3.5
 ╚═══bin/kdeconnect-cli: …port QT_PLUGIN_PATH='/nix/store/qssjj6ki7jiskw2kfygvfiy8fxrclwrl-kconfigwidgets-5.71.0-dev/lib/q…
     => /nix/store/qssjj6ki7jiskw2kfygvfiy8fxrclwrl-kconfigwidgets-5.71.0-dev
```

Also similar (but possibly fixable by moving `gobject-introspection` to a
different inputs list?):
 
```
$ nix why-depends -f. beets gobject-introspection.dev
/nix/store/93lfrhm8vp17m8ziqi8vp6v4cff67wkb-beets-1.4.9
╚═══bin/beet: …-expat-2.2.8-dev/bin:/nix/store/y3ym76wrak3300vsjyf3klr52cnzmxwd-gobject-introspection-1.64.1-de…
    => /nix/store/y3ym76wrak3300vsjyf3klr52cnzmxwd-gobject-introspection-1.64.1-dev
```

## Other issues

- [issue 60260](https://github.com/NixOS/nixpkgs/issues/60260)

General, justified complaint about wrappers.

- [issue 95027](https://github.com/NixOS/nixpkgs/issues/95027)
- [issue 23018](https://github.com/NixOS/nixpkgs/issues/23018)
- [issue 11133](https://github.com/NixOS/nixpkgs/issues/11133)
- [pull 95569](https://github.com/NixOS/nixpkgs/pull/95569)

Since our wrappers are shell scripts, `gdb` can't run them. What if we had
written a C based wrapper, that perhaps would read what environment it needs to
set from a JSON file, and it will call the unwrapped original executable? I
need feedback regarding whether `gdb` will play nice with this.

This issue may not directly relate to declarative wrappers, and it is already
addressed in @FRidh's [pull 95569](https://github.com/NixOS/nixpkgs/pull/95569), but perhaps
both ideas could be integrated into an alternative, simpler creation method of
binary wrappers. See [my
comment](https://github.com/NixOS/nixpkgs/pull/95569#issuecomment-674508806).

- `hardware.sane` module installs sane impurly
- [issue 90201](https://github.com/NixOS/nixpkgs/issues/90201)
- [issue 90184](https://github.com/NixOS/nixpkgs/issues/90184)

The current way NixOS enables to configure access to scanner, is via the
`hardware.sane` module, which interacts [just a
bit](https://github.com/NixOS/nixpkgs/blob/5d8dd5c2598a74761411bc9bef7c9111d43d2429/nixos/modules/services/hardware/sane.nix#L34)
wish `saned`, but mostly does nothing besides setting `SANE_CONFIG_DIR` and
`LD_LIBRARY_PATH`. This is bad because:

1. Running `nixos-rebuild` after a change to sane or it's configuration,
   requires the user to logout and login, to see effects of the changes.  
   
2.
[Apparently](https://github.com/NixOS/nixpkgs/issues/90201#issuecomment-683304279)
`LD_LIBRARY_PATH` is cleared when an application runs with extra capabilities.
This causes the current gnome-shell wayland wrapper to unset the
`LD_LIBRARY_PATH` many scanner related programs now rely upon.

Declarative wrappers should enable us to make such applications that use a
`SANE_CONFIG_DIR` and `LD_LIBRARY_PATH` to be configured during the wrap phase,
and get rid of these global environment variables.

# Detailed design
[design]: #detailed-design

The end goal is to make the experience of getting a derivation wrapped as
automatic as possible. A derivation that needs some environment variables in
order to work will get these environment variables set in the wrapper by
`mkDerivation`'s standard `fixupPhase`.  As a start, we'll (me probably) will
introduce a new `makeWrapperAuto` setup hook that will take care of this in the
way described as follows.

As a start, we'll need to think about all packages in Nixpkgs that require
environment variables to be set at runtime. Every such package will put in
`$dev/nix-support/wrappers.json` a list of environment variables that are
"linked" to this package. "linked" means that using this package requires these
environment variables to be set in runtime.

`makeWrapperAuto` will traverse all `buildInputs` and `propagatedBuildInputs`
of a derivation, and look for a `wrappers.json` file in these inputs. It will
collect all the environment variables that need to be set in the resulting
executables, by merging all of the values of the environment variables in all
of the inputs' `wrappers.json` files. `wrappers.json` might look like this for
a given package:

```json
{
  "GI_TYPELIB_PATH": [
    "/nix/...gobject-introspection.../...",
    "/nix/...librsvg.../...",
  ],
  "GIO_EXTRA_MODULES": [
    "/nix/...dconf.../lib/gio/modules"
  ],
  "XDG_DATA_DIRS": [
    "/nix/...gtk+3.../...",
    "/nix/...gsettings-desktop-schemas.../..."
  ],
  "GDK_PIXBUF_MODULE_FILE": "/nix/...librsvg.../lib/gdk.../loaders.cache",
}
```

There is a default behaviour that `makeWrapperAuto` will use, for example:

- The separator between the paths should default to `:`, but for some known
  environment variables that use `;` (e.g `LUA_PATH`), `makeWrapperAuto` should
  know in advance that this is the separator they need. This information should
  be coded in a nix / shell file.
- Some environment variables are not meant to be a list, like
  `GDK_PIXBUF_MODULE_FILE`. This information should be understood by
  `makeWrapperAuto` because it appears in `wrappers.json` as a single string.
- The default behavior for the order used eventually in the wrapper should be
  the order in which the information is gathered, which leaves the lower level
  dependencies' paths to the end. If someone reading this RFC can give examples
  where a special order might be required, let us know.

The information found inside an input's `wrappers.json` will specify the
wrapper information not only for itself, but for all its dependencies
(including transitives) as well. Thus, in contrast to the [POC Nixpkgs
PR](https://github.com/NixOS/nixpkgs/pull/85103) and the [original design of
the
RFC](https://github.com/doronbehar/rfcs/blob/60d3825fdd4e6574b7e5d70264445d1c801368c6/rfcs/0075-declarative-wrappers.md#L251),
prior to [the 1st
meeting](https://github.com/NixOS/rfcs/pull/75#issuecomment-760942876),
traversing all the inputs and the inputs' inputs, will not happen during eval
time and only partly, during build time - every package already built will
provide its reverse dependencies all the information they need about
environment variables.

Most of the work to do will be:

1. Gather information about what environment variables are "linked" to each
   package, and edit these derivations to include a `wrappers.json` in them.
   This should be done with `makeWrapperAuto` as well, see (2).
2. Design the `makeWrapperAuto` shell hook:
  - It should introduce a shell function (to be called `combineWrappersInfo`) that
    will allow piping a JSON string from `builtins.toJSON` and spit a
    `wrappers.json` that will include both what was piped into it, and the
    content from the package's various inputs' `wrappers.json` files.
  - It should wrap a list of executables according to a list of glob patterns,
    defaulting to `["$outputBin/bin/*" "$outputLib/libexec/**/*"]`, according
    to what's currently in this package's `wrappers.json`, during `fixupPhase`.
  - It should be possible to override this list of glob patterns, and specify a
    new list instead of it. The other list can include glob patterns as well,
    or specific executables instead. For example:

```nix
  wrapPrograms = [ "$outputLib/libexec/${pname}/run-server" "$out/share/scripts/*" ];
```

3. Most of the packages with linked environment variables, have lots of reverse
   dependencies, so once `makeWrapperAuto` is ready, it'd nice to have a hydra
   job that will build all of these packages with the `wrappers.json` file in
   them. For instance these packages include:
   - `gdk-pixbuf`
   - `gsettings-desktop-schemas`
   - `pango`
   - `gtk3`

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

When switching to `makeWrapperAuto` from `makeWrapper` there shouldn't be
manual usage of `wrapProgram` for most cases. A package that uses `wrapProgram`
should be able to switch to `combineWrappersInfo` and declare any nontrivial
environment variables with it to get propagated to reverse dependencies and to
it's executables automatically.

Currently I imagine the usage of `combineWrappersInfo` (the name can be debated) as
so:

```nix
  # Propagate GST plugins' path
  postInstall = makeWrapperAuto.combineWrappersInfo {
    envInfo = {
      GST_PLUGIN_SYSTEM_PATH_1_0 = [
        # @out@ should be expanded by `combineWrappersInfo` to what's in `$out`, see:
        # https://github.com/NixOS/nixpkgs/pull/85103#issuecomment-613071343
        "@out@/lib/gstreamer-1.0"
      ];
    };
  };
  '';
```

The above will put a shell command in `postInstall` that's generated by
`combineWrappersInfo` that will put the environment information into
`wrappers.json`.

It should be possible to tell `combineWrappersInfo` to use a symlink directory
instead of a big environment variable, by using for example: 

Other nix attributes along `envInfo` such as:

```nix
  envInfo = {
    # ...
  };
  symlink = {
    THIS_ENV_VAR = "to/this-directory/in-out";
  };
```

`wrapQtAppsHook` and `wrapGAppsHook` should be replaced with `makeWrapperAuto`
which enables removing frequent workarounds such as:

```nix
  # hook for gobject-introspection doesn't like strictDeps
  # https://github.com/NixOS/nixpkgs/issues/56943
  strictDeps = false;
```

And often seen in Python + Qt programs:

```nix
  preFixup = ''
    makeWrapperArgs+=("''${qtWrapperArgs[@]}")
  '';
```

# Drawbacks
[drawbacks]: #drawbacks

Using `wrapProgram` is simpler than using `combineWrappersInfo` and it might be
hard to explain why is there no `wrapProgramAuto`. However, this interface
might get improved in design through this RFC or in the future and in any case
proper documentation should help.

# Alternatives
[alternatives]: #alternatives

Perhaps our shell hooks _can_ be fixed / improved, and we could help make it
easier to debug them via `NIX_DEBUG`. Then it might help us track down e.g why
environment variables are added twice etc. Still though, this wouldn't solve
many issues presented above.

# Unresolved questions
[unresolved]: #unresolved-questions

Discussing the design I guess, here or in the Nixpkgs PR that will follow this
RFC.

# Future work
[future]: #future-work

Fix all wrapper related issues declaratively!
