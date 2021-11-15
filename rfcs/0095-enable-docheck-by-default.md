---
feature: enable-docheck-by-default
start-date: 2021-06-24
author: gytis-ivaskevicius
co-authors:
shepherd-team: @Ericson2314 @nh2 @edolstra
shepherd-leader: @Ericson2314
related-issues:
---

# Summary
[summary]: #summary

Enable `doCheck` by default when using `stdenv.mkDerivation` function.

# Motivation
[motivation]: #motivation

I believe that an additional quality gate would be beneficial to the derivations build process.

Resolution of this RFC is expected to remove [these comments](https://github.com/NixOS/nixpkgs/blob/8c563eaf7049d82fbe95b0847ac5ae6e5554e2fa/pkgs/stdenv/generic/make-derivation.nix#L61-L67)
either by enabling `checkPhase` by default or rejecting this RFC.

# Detailed design
[design]: #detailed-design

The basic idea is quite simple
- New `doCheck`/`doInstallCheck` semantic should be implemented.
- By default `doCheck` option should be enabled as long as `stdenv.hostPlatform == stdenv.buildPlatform`.
- Non-reproducible test prevention should be implemented.
- All failing packages should be fixed or updated with `doCheck = false;`

**New `doCheck`/`doInstallCheck` semantics:**
In addition to booleans, `doCheck`/`doInstallCheck` should also accept strings.
- String value should be considered as `false`
- It should be used as a place for comment on why the check is disabled. For
  example: "Requires X11 server" or "Requires network access".

**Non-reproducible tests prevention:**
There are multiple options. Here I am going to list a few:
1. `chmod a-w -R /build`
2. [User namespaces](https://lwn.net/Articles/532593/)
3. Generate unique identifier from existing sources and compare it with
   identifier generated after executing `checkPhase`. `exit 1` if values
   mismatch. (Identifier can be generated by something simple like `du -s`)
4. [unionfs](https://en.wikipedia.org/wiki/UnionFS)

# Drawbacks
[drawbacks]: #drawbacks

- Increased build time.
- More non-deterministic build failures.
- Extra dependencies for the test framework.
- Upstream tests don't often reveal downstream packaging/integration issues, because most are functional tests that are unlikely to break.

# Alternatives
[alternatives]: #alternatives

The impact of avoiding this is a lack of assurance of delivered package quality.

# Unresolved questions
[unresolved]: #unresolved-questions

"Non-reproducible tests prevention" implementation is to be decided. I feel
like `du -s` is the right way to go about it since it is simple/fast and I
expect it to be quite reliable.