---
feature: (fill me in with a unique ident, my_awesome_feature)
start-date: (fill me in with today's date, YYYY-MM-DD)
author: (name of the main author)
co-authors: (find a buddy later to help out with the RFC)
shepherd-team: (names, to be nominated and accepted by RFC steering committee)
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

Rotate cache.nixos.org signing key.

If we tolerate that people using stuff older than *X* could get a validation failure by default, fully rotating the key in 2*X* time seems relatively straightforward.

# Motivation
[motivation]: #motivation

Never rotating a key is bad security practice.
The current one has been in use at least since 2015.

# Detailed design
[design]: #detailed-design

Preliminary action plan:
- generate a new key
- make it trusted by default (nix+nixpkgs, perhaps with backports to some branches)
- wait until enough people trust the new key (at least one year, probably)
- switch to signing with the new key
- wait - until paths not signed by new key aren't commonly needed anymore
- make Nix not need signatures for fixed-output derivations
  (this step could be completed anytime earlier, too)
  FIXME: maybe this holds already:
    https://nixos.org/manual/nix/unstable/command-ref/conf-file.html#conf-trusted-public-keys
- stop trusting the old key (nix+nixpkgs)



- - -


# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

This section illustrates the detailed design. This section should clarify all
confusion the reader has from the previous sections. It is especially important
to counterbalance the desired terseness of the detailed design; if you feel
your detailed design is rudely short, consider making this section longer
instead.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

- change nothing, obviously
- also resign old `*.narinfo`.  Maybe it's not too hard.
  It would help people wanting ot use old builds.
- double-sign `*.narinfo` for some time. (also not an exclusive alternative)
  I don't know if consumers support multiple signatures.
  It doesn't seem to give us significant advantage though;
  acceptance of multiple keys seems more advantageous.

# Unresolved questions
[unresolved]: #unresolved-questions

- confirm Nix's (non-)acceptance of FODs without signature
- determine timing (e.g. the *X* above).
  Consider aligning the regular schedule with NixOS releases.

# Future work
[future]: #future-work

What future work, if any, would be implied or impacted by this feature
without being directly part of the work?
