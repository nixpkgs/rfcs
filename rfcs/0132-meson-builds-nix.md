---
feature: meson_builds_nix
start-date: 2022-08-25
author: Anderson Torres
co-authors: @p01arst0rm
shepherd-team: @Ericson2314 @edolstra @thufschmitt
shepherd-leader: @Ericson2314
related-issues:
---

# Summary
[summary]: #summary

Use meson as an alternative build system for Nix.

# Motivation
[motivation]: #motivation

Currently, the Nix evaluator and its companion toolset generated from the Nix source tree are built using the typical `./configure` shell script that relies on autoconf, along with standard GNU Make utility.

Over time, this build system has been modified to keep up with the development needs of Nix project. However, it has reached a state where the build system became clunky and plastered, hard to understand and modify, consequently making improvements to the project as a whole very difficult.

In particular, many changes have been introduced that impact compatibility outside Linux and NixOS niches. These issues can hinder development on other platforms, including but not limited to Unix-like systems.

In light of this state of things, we propose a novel alternative to the current building infrastructure.

We expect to accomplish, among other goals,

- better code structuring and documentation;
- improved support across platforms, especially outside NixOS and Linux;
- shorter build times;
- improved testing;
- more reasonable dependency management;
- an overall improved user experience.

# Detailed design
[design]: #detailed-design

## What is Meson?

Meson is an open source, multiplatform build system crafted to be fast, correct and user-friendly.

According to its main site,

> The main design point of Meson is that every moment a developer spends writing or debugging build definitions is a second wasted. So is every second spent waiting for the build system to actually start compiling code.

Among its features, we highlight:

- user-friendly non-Turing complete DSL
  - Python-esque syntax and functional, stateless semantics
- multiplatform support
  - among operating systems: Linux, *BSD, Apple MacOS, Microsoft Windows NT
  - among programming environments: GCC, Clang, Xcode, Visual Studio etc.
  - among programming languages: C, C++, D, Fortran, Java, Rust etc.
    - supports command customization
  - cross compilation
  - many useful modules included (pkg-config, filesystem inspection, internationalization etc.)
- out-of-source build support
  - indeed, meson does not support inside-source build
- Comprehensive documentation
  - including tutorials, reference manuals and real world projects using it

## What is Ninja?

Ninja is a small, speed-focused build tool that fills a similar role of Unix `make` or its GNU counterpart `gmake`.

Its main feature is a low-level approach to build description. Ninja is bare-bones and constrained by design, having only the necessary semantics to describe build dependency graphs, relegating decision-making to superior tools like Meson or CMake. Where other build systems act like high level languages, Ninja acts like an assembly.

Albeit Ninja DSL is human-readable, it is not convenient to be manually written by human beings. As said before, Ninja is commonly used in tandem with other, higher-level build system in a two-pass fashion. In our present use case, the Meson interpreter converts Meson files to Ninja files that will be consumed by Ninja tool to effectively execute the building/deployment commands.

## Design

A carefully crafted set of Meson files should be included in order to describe how to deploy the Nix repository, generating all the expected artifacts (command line tools, libraries, configuration files, documentation etc.)

This novel building infrastructure should be able to provide at least feature parity with the current quasi-autotools implementation, albeit in a different user interface.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

## Example interaction

Here is a table comparing some expected interactions:

| Action    | Current                        | Meson+Ninja                          | Meson, backend-agnostic              |
|-----------|--------------------------------|--------------------------------------|--------------------------------------|
| Configure | `./configure --enable-gc=true` | `meson setup build_dir -Dgc=enabled` | `meson setup build_dir -Dgc=enabled` |
| Build     | `make`                         | `ninja -C build_dir build`           | `meson -C build_dir compile`         |
| Install   | `make install`                 | `ninja -C build_dir install`         | `meson -C build_dir install`         |
| Uninstall | `make uninstall`               | `ninja -C build_dir uninstall`       | `meson -C build_dir uninstall`       |

## Implementation

Currently, @p01arst0rm is working on an implementation from scratch.

# Drawbacks
[drawbacks]: #drawbacks

Below we will list some possible drawbacks and balances to them.

## Complexity and Bootstrap Issues

A new build system, and any build system for that matter, indirectly brings its own dependencies to the Nix project.

Ideally, such dependencies should be minimal, both in number and complexity, with extra good points for dependencies already present (e.g. the ubiquitous C compiler).

About this specific point, a non-negligible drawback is: the reference implementation of Meson is written in Python. At least theoretically, it brings the necessity of including a Python 3 interpreter on the _bootstrap route_ of Nix.

However, some points can be laid out on the opposite side:

1. By design, the meson reference evaluator depends only on Python, avoiding the use of extra libraries.
2. Also by design, the implementation language is never exposed to the meson DSL. It allows the possibility of implementing alternative evaluators in other programming languages.
   1. Indeed, Muon is an alternative implementation of Meson written in C.
3. In order to evaluate the impact of this new bootstrap route, we should also evaluate the current bootstrap route, in order to have a fair comparison.

In principle, the same criticisms and answers can be laid out for Ninja too; however, Ninja is written in C++, a language already used to implement Nix. Therefore, the bootstrap route suffers little to no alteration here.

## Learning the new system

A somewhat subjective but non-negligible issue is the entry barrier of this new build system. Switching from a known build system to one unknown is not without its problems.

However, the Meson development team strives to keep the DSL easy to learn and pleasurable to use. It should not be hard to become familiar with the Python-esque syntax of meson, and its functional, stateless approach is certainly a feature highly appreciated by the Nix community as a whole.

The huge advantages of implementing Meson surpass the small drawbacks of learning it.

Further, the current barrier of entry should be evaluated in order to have a fair comparison. 

## Source code changes

Further to the obvious inclusion of meson files (and the removal of the old quasi-autotools ones), there is a reasonable expectation of code refactoring.

However, such refactorings are completely validated on the long term goals of Nix, in particular the improvements on portability.

## End users

The most known end user of Nix is certainly Nixpkgs. However, there are many other Linux distributions that already keep Nix on their repositories (15 families, according to Repology). There is also a reasonable expectation of affecting those package managers' devteams.

However, most (if not all) of those distributions already have Meson and its companion tool Ninja in their respective package databases (53 families, according to Repology), given that many open source projects use them as build system.

## Transition between old and new build infrastructure

The transition between old and new build systems should be as smooth and controlled as possible.

# Alternatives
[alternatives]: #alternatives

The alternatives are

- Do nothing

  It would keep the current code confusing and harder to work with, as stated on the motivation section.

- Use CMake

  Indeed, CMake has many noteworthy advantages:

  - Supports typical high level idiomatic constructions.
  - Can generate GMake- and Ninja-compatible scripts.
    - By design, Meson does not provide a Make backend.
  - Both Meson and CMake support Microsoft Windows NT and Apple MacOS platforms
    - As well as MSVC and XCode programming environments.

  However, CMake DSL is arguably more complex and cumbersome, whereas Meson is more polished.

- Evaluate other building systems (waf, scons, premake, xmake etc.)

  About this, a principle should be observed:

  Per the bootstrap route issue discussed above, build tools strongly tied to other programming languages are severely discouraged.

  E.g. waf and scons are Python libraries in essence, whereas premake and xmake are Lua libraries. They can't be decoupled of their implementation languages.

  Another indispensable attribute to be considered is cross-compilation support.

- Use Bazel

  For the sake of completeness, there is Bazel, a Google LLC-backed build system that sells itself as "fast, scalable, multi-language and extensible".

  Advantages:

  - Fast, scalable, multi-language and extensible.
    - Your mileage may vary.
  - Trusted by industry leaders.
  - Backed by Google.

  Disadvantages:

  - [Too tied to Google internals](https://bazel.build/about/faq#why_isn%E2%80%99t_all_development_done_in_the_open)
  - [Not fully open source yet](https://bazel.build/about/faq#are_you_done_open_sourcing_bazel)
  - Written in Java
    - Java bootstrap is fairly complex, currently relying on _open-source abandonware_, as demonstrated by [Bootstrappable](https://bootstrappable.org/projects/java.html) project.
    - At the current time there is no alternative implementation of Bazel in another programming language.
  - Backed by Google.

# Prior Art

For the sake of completeness, and pushed by the advent of [RFC Process updates](https://github.com/NixOS/rfcs/pull/150), we are including here a list of projects that are in different stages of transition to Meson:

- [GNOME](https://wiki.gnome.org/Initiatives/GnomeGoals/MesonPorting)
- [X.Org Server](https://gitlab.freedesktop.org/xorg/xserver)
  - [XTS](https://gitlab.freedesktop.org/xorg/test/xts), the X.Org Test Suite
- [MPV](https://mpv.io), the commandline video player
- [GIMP](https://gitlab.gnome.org/GNOME/gimp), the GNU Image Manipulation Program
- [Mesa](https://mesa3d.org/), the open source implementations of many low-level, graphics-related libraries

# Unresolved questions
[unresolved]: #unresolved-questions

Questions that deserve further inquiry:

- Unexpected interactions with Meson and Ninja
  - Specially, vendoring and reproducibility.

# Future work
[future]: #future-work

- Update project's continuous integration and related stuff;
- Deprecate and remove the current quasi-autotools scripts
  - Preferably, the removal should be allocated to a minor version release.
- Evaluate the positive and negative impacts of such a change
  - Specially the most subjective ones.

# References
[references]: #references

- [Meson](https://meson.build/)
  - [Muon](https://muon.build/), a C99 alternative implementation
- [Ninja](https://ninja-build.org/)
  - [Samurai](https://github.com/michaelforney/samurai), a C99 alternative implementation
- [Meson tutorial](https://mesonbuild.com/Porting-from-autotools.html) comparing autotools and Meson
- [NetBSD tutorial](https://wiki.netbsd.org/pkgsrc/how_to_convert_autotools_to_meson/) comparing Meson and autotools
- [Boostrappable Builds](https://bootstrappable.org/)
- [CMake](https://cmake.org/)
- [Premake](https://premake.github.io/)
- [Waf](https://waf.io/)
- [SCons](https://scons.org/)
- [Xmake](https://xmake.io/)
- [Bazel](https://bazel.build)

- [Free-to-read book from the creator of Meson](https://nibblestew.blogspot.com/2021/12/this-year-receive-gift-of-free-meson.html)

- [Current work in progress from @p01arst0rm](https://github.com/NixOS/nix/pull/3160)
