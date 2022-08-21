---
feature: nixpkgs-generated-code-policy
start-date: 2021-10-12
author: John Ericson (@Ericson2314)
co-authors: (find a buddy later to help out with the RFC)
shepherd-team: @L-as @grahamc @sternenseemann
shepherd-leader: @sternenseemann
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

Nixpkgs contains non-trivial amounts of generated, rather than hand-written code.
We want to start systematizing to make it easier to maintain.
There is plenty of future work building upon this we could do, but we stop here for now to avoid needing to change any tools (Nix, Hydra, etc.).

# Motivation
[motivation]: #motivation

Nixpkgs, along with every other distro, also faces a looming crisis: new open source software is increasingly not intended to be packaged by distros at all.
Many languages now support very large library ecosystems, with dependencies expressed in a language-specific package manager.
To this new generation of developers, the distro (or homebrew) is a crufty relic from an earlier age to bootstrap modernity, and then be forgotten about.

Right now, to deal with these packages, we either convert by hand, or commit lots of generated code into Nixpkgs.
But I don't think either of those options is healthy or sustainable.
The problem with the first is sheer effort; we'll never be able to keep up.
The problem with the second is bloating Nixpkgs but more importantly reproducibility: If someone wants to update that generated code it is unclear how.
All these mean that potential users coming from this new model of development find Nix / Nixpkgs cumbersome and unsuited to their needs.

The lowest hanging fruit is to systematize our generated code.
We should ensure anyone can update the generated code, which means it should be built in derivations not some ad-hoc way.
In short, we should apply the same level of rigour that we do for packages themselves to generate code.

# Detailed design
[design]: #detailed-design

## Nixpkgs

1. Establish the policy that all generated code in nixpkgs must be produced by a derivation.
   The derivation should be built by CI (so exposed as some Nixpkgs in some fashion).

2. Implement script(s) for maintainers which automatically builds these derivations and vendors their results to the appropriate places.
   Running such scripts should be sufficient to regenerated all generated code in Nixpkgs.

3. Ensure via CI that the vendored generated code is exactly what running the scripts produce.
   This check should be one of the "channel blocking" CI jobs.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

## Impurities

Many `lang2nix`-type tools have impure steps today.
Since these tools must be only invoked inside the derivations to generate code, the impure inputs must be gotten via fixed output derivations.
This might require changes to those tools

Updating fixed output hashes and similar however is perfectly normal and not affected by this RFC.
The updates can be performed by hand, or with update bots like today.
The update bots would just need to learn to run the regeneration script (or risk failing CI because the vendored generated code is caught as being out of date).

## Idempotency and bootstrapping

The test that the generated sources are up to date will have to work by regenerating those generated sources and then taking a diff.
That means the regeneration process hash to be idempotent in that running it twice is the same as running it once.

This is a bit tricker than it sounds, because many `lang2nix` tools rely on their own output.
E.g. the Nix packaging for `cabal2nix` is itself generated with `cabal2nix`.
Sane setups should work fine --- after all, it would be really weird if two valid builds of `cabal2nix` behaved so differently as to generate different code --- but it still an issue worth being aware of.

(That we continue to vendor code does at least "unroll" the bootstrapping to avoid issues that we would have with, say, import-from-derivation alone.
The vendored code works analogously to the prebuilt bootstrapping tools in this case.)

@sternenseemann reminds me that some `lang2nix` tools might pin a Nixpkgs today, for various reasons.
But in this plan the tools must be built with the current Nixpkgs in the CI job ensuring sources are up to date.
`lang2nix` tools must therefore be kept continuously working when built against the latest Nixpkgs.

## What CI to use?

The easiest, and most important foundational step to do is just add a regular `release.nix` job for Hydra to test.
We might, however, want to catch these issues earlier at PR merge time, with ofborg or GitHub actions.
That is fine too.

## Who does the work?

In the short term, this is a decent chunk of work for `lang2nix` tool authors and language-specific packages maintainers, who must work to ensure their tools and workflows are brought into line with this policy.
That won't always be fun!

On the flip side, a major cost of today's situation is since so many of the workflows are more an "oral tradition" to the maintainers and not fully reproducible, one-off contributors often need a lot of hand-holding.
@sternenseemann tells me he must spend a lot of manual time shepherding PRs, because those PR authors are unable to jump through the hoops themselves.

# Drawbacks
[drawbacks]: #drawbacks

This is now a very conservative RFC so I do not think there are any drawbacks as to the goals themselves.

Bringing our tools into compliance with this policy will take effort, and of course that effort could be spent elsewhere, so there is opportunity cost to be aware of.
But given the general level of concern over the sustainability of Nixpkgs, I think the benefits are worth the costs.

# Alternatives
[alternatives]: #alternatives

None at this time, we had other ideas but they are reframed as future work.
The one proposed here is unquestionably the most conservative one, and basically a prerequisite of all the others.

# Unresolved questions
[unresolved]: #unresolved-questions

None at this time.

# Future work
[future]: #future-work

## Vendor generated code "out of tree"

The first issue that remains after this RFC is generated code still bloats the Nixpkgs history.
It would be nice to get it "out of tree" (outside the Nixpkgs repo) so this is no longer the case.
In our shepherd discussions we had two ideas for how this might proceed.

It was tempting to go straight to proposing one of these as part of the RFC proper,
but they both contained enough hard-to-surmount issues that we figured it was better to start something more conservative first.

### Dump in other repo and fetch it

We could opt to offload all generated code into a separate repository which would become an optional additional input to nixpkgs.
This could be done via an extra `fetchTarball`, possibly a (somehow synced) channel or, in the presence of experimental features, a flake input.

####  Drawbacks

- This would be a truly breaking change to nixpkgs user interface:
  Either an additional input would need to be provided or fetched (which wouldn't interact well with restrict-eval).

- Generated code becomes a second class as the extra input would need to be optional for this reason.
  This is problematic for central packages that use code generation already today (pandoc, cachix, …).

- Similar Bootstrapping problems as the other alternative below: new generated code needs nixpkgs and a previous version of the generated code.

- `builtins.fetch*` is a nuisance to deal with at the moment and would probably need to be improved to make this work.
  E.g. gcrooting this evaluation only dependency could prove tricky without changes to Nix.

- Extra bureaucracy would be involved with updating the generated repository and the reference to it in nixpkgs.
  Additionally, special support in CI would be required for this.

### Nixpkgs itself becomes a derivation output

This alternative implementation was proposed by @L-as at the meeting.
The idea is that nixpkgs would become a derivation that builds a “regular” nixpkgs source tree by augmenting files available statically with code generation.

The upside of this would be that there would only be one instance of IFD that can ever happen, namely when the source tree is built.
The produced store path then would require no IFD, and it would be obvious what relates to IFD and what doesn't.

In practice, IFD would not be necessary for users of nixpkgs if we can design a mechanism that allows the dynamically produced nixpkgs source tree to be used as a channel.
Then the IFD would only need to be executed when working on nixpkgs.

#### Drawbacks

- This approach creates a bootstrapping problem for the entirety of nixpkgs, not just for the IFD parts.
  It would be necessary to build the new nixpkgs source tree using an old version of the nixpkgs source tree.
  This could either be done using a fixed “nixpkgs bootstrap tarball” which occasionally needs to be bumped manually as code generation tools require newer dependencies, or by pulling in the latest nixpkgs source tree produced by e.g. Hydra.
  The latter approach of course runs the risk of getting stuck at a bad nixpkgs revision which is unable to build the next ones fixing the problem.

- Working on nixpkgs may involve more friction: It'd require a bootstrap nixpkgs to be available and executing the IFD for the nixpkgs source tree, likely involving hundreds of derivations.

- Hydra jobsets would need to be sequenced: First the new nixpkgs source tree would need to be built before it can be passed on to the regular `nixpkgs:trunk`, `nixos:trunk-combined` etc. jobsets.

- Channel release would change significantly: Instead of having a nixpkgs git revision from which a channel tarball is produced (mostly by adding version information to the tree), a checkout of nixpkgs would produce a store path from which the channel tarball would be produced.
  This could especially pose a problem for the experimental Flakes feature which currently (to my knowledge) assumes that inputs are git repositories.

## Import from derivation

Even if we store the generated sources outside of tree, we are still doing the tedious work of semi-manually remaining a build cache (this time of Nix code).
Isn't that what Nix itself is for!

"import from derivation" is a technique where Nix code can simply import the result of a build, with no vendoring generated code in-tree or out-of-tree needed.

There are a number of implementation issues with it, however, that means we can't simply enable it on `hydra.nixos.org` today.
We have some "low tech" mitigations that were the original body of this RFC,
but they still require changing tools (Hydra), which adds latency and risk to the project.

## Reaching developers, more broadly

This proposal is far from the final decision on how language-specific ecosystems packages should be dealt with.
I make no predictions for the far future, it is possible we will eventually land on something completely different.

However, I think this RFC will help us reach a very big milestone where the `lang2nix` ecosystem and Nixpkgs will both be talking to each other a bit better, not just Nixpkgs saying things but not listening to a chaotic and disorganized `lang2nix` ecosystem.
This culture shift I think will be the main and most important legacy of this RFC.

A lot of developers come to the Nix ecosystem, and find that the tools work great for sysadmin-y or power-user-y things (NixOS, home-manager, etc.) but the development experience is not nearly as clearly better than using language-specific tools in comparison.
(I prefer it, but the tradeoffs are very complex.)
With the new both-ways communication described above, I think we'll have a huge leg up in refining best practices so that ultimately we have better developement workflows, and retain these people better.

The developers I am most eager to reach are those of major upstream projects
In turn, I hope these upstream packages and ecosystems might even care about packaging and integration of the sort that we do.
This would create a virtuous cycle where Nix is easier to use by more people, and Nixpkgs is easier to maintain as upstream packages better match our values.
Instead of a situation where distros and upstream projects don't really like each other, we might end up with a situation where they all get along via Nix.