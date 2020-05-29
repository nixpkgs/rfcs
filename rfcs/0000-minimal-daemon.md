---
feature: minimal-daemon
start-date: 2020-5-28
author: John Ericson
co-authors: (find a buddy later to help out with the RFC)
shepherd-team: (names, to be nominated and accepted by RFC steering committee)
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

`nix-daemon` should be a separate executable.

# Motivation
[motivation]: #motivation

With flakes and other development, we are moving towards a more "batteries included" Nix command line.
We don't want any of those features in the daemon, however, because the daemon is a special trusted process that we should strive to keep as simple as possible.
\[This is comparable to a compilation pipeline, with a concise intermediate representation that nicer user-facing features "desugar" into.\]

There are many things we could do about this, but I mainly want to establish some rough consensus around the problem while taking a small step to signal that consensus.
Originally, each Nix command was its own executable, but then we combined them into one executable.
I think this is fine for the main user-facing commands, but not good for the daemon.

Finally, it's probably best not to give the daemon---a long lived process running with elevated privileges---access to tons of dead code.
C++ doesn't exactly prevent memory errors, and that dead code is just more fodder to be used in some low-level attack.
There are other solutions to this in the long term, but this is the easiest solution in the short term.

\[This RFC is completely separate from the ongoing IPFS work.]

# Detailed design
[design]: #detailed-design

- `nix-daemon` will be a separate executable that only links the nix libraries it needs.
  \[At this time, those libraries are `libnixutil`, `libnixstore`, and `libnixrust`, but this is subject to change.\]

- `nix-daemon` should never need to understand the expression language and depend  `libnixexpr`.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

I certainly hope there are no interactions!
One of the bad things we should seek to prevent with this is the daemon unintentionally growing dependencies on more of the code base.

# Drawbacks
[drawbacks]: #drawbacks

Not much.
Build rules perhaps are slightly more complex as there are both separate and independent executables.

# Alternatives
[alternatives]: #alternatives

 - Do nothing.

 - Something more invasive.
   But I much rather save that for later.

# Unresolved questions
[unresolved]: #unresolved-questions

No known unknowns.

# Future work
[future]: #future-work

There's lots of future work we could do in the vein of modularity.
But there's also different, conflicting directions we could go in.
The point of this small step is to punt on all that for now.