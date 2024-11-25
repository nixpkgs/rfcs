---
feature: moderation team
start-date: 2021-08-18
author: tomberek
co-authors: blaggacao
shepherd-team: @ryantm, @7c6f434c, @IreneKnapp
shepherd-leader: @zimbatm
related-issues: https://github.com/NixOS/rfcs/pull/98
---

# Summary
[summary]: #summary

Establish a team to perform moderation.

# Motivation
[motivation]: #motivation

There is not currently any official mechanism for moderation action. It's not
sustainable to have to call on Graham any time there's a spammer or conflict
that requires moderation, and we'd like to help the community become more
self-regulating.

The intention behind this RFC is to codify the current moderation practices.

# Detailed design
[design]: #detailed-design

The Moderation Team is a volunteer group that may receive, invite, and evaluate
applications to the team or alter the composition at any time. The team's
composition, contact information, procedures, moderation log, and announcements
should be available via
[https://nixos.org/community/teams/moderation.html](https://nixos.org/community/teams/moderation.html).
The distribution of work within the team may be treated as an internal matter.
The team shall perform moderation activities on behalf of the community - with
oversight via the RFC process and input from the NixOS Foundation - for
discussions in [official project
spaces](https://nixos.org/community/index.html) as well as unofficial spaces
that seek and reach such an agreement with the team. The team should utilize
the [NixOS Foundation mission](https://nixos.org/community/index.html) and the
following statement during their duties:

```
The NixOS Foundation aims to promote participation without regard to gender,
sexual orientation, disability, ethnicity, age, or similar personal
characteristics. We want to strive to create and foster community by providing
an intentionally welcoming and safe environment where all feel valued and cared
for, and where all are given opportunity to participate meaningfully. The
Foundation will work with the community in service of this goal.
```

Discussions about moderation must not be moderated.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

- The initial Moderation Team - drawn from the existing Discourse and GitHub
  teams - is defined to be @grahamc, @zimbatm, @domenkozar, @Mic92, @garbas,
  and @ryantm.
- Rename the Discourse Team to Moderation Team on
  https://nixos.org/community/teams/discourse.html and utilize
  https://nixos.org/community/teams/moderation.html.
- Establish and publish a clear point of contact for abuse reporting and a
  venue for discussion about moderation specific topics such as a dedicated
  Matrix channel or Discourse topic.

# Drawbacks
[drawbacks]: #drawbacks

* The moderation team has limited guidance from this RFC on the processes and
  procedures of the team.
* This RFC is designed to address a narrow part of a current issue facing the
  community. Additional RFCs may be needed to address additional concerns.
* As this is a controversial topic there is a potential this RFC does not have
  enough detail to be acceptable by the overall community.

# Alternatives
[alternatives]: #alternatives
* Define a constituency of project participants and some (more bottom-up/grassroots) procedure with a foundation in popular support for specific moderators.

* An existing [RFC 98][].
* Do nothing.

# Unresolved questions
[unresolved]: #unresolved-questions

* Does the team require additional guidance?
* Does the NixOS Foundation board want to be involved in this manner?

# Future work
[future]: #future-work

* A potential RFC providing additional guidance and detail for the moderation
  team's activities and functions.
* The role of the moderation team could evolve through an effort similar to
  [RFC 98][] into taking a broader community leadership responsibility as a
  'Leadership Team' or 'Community Team'.
* Work on clarifying community norm guidelines. This can include adopting
  typical governance tools, such as Contributor
  Covenant, Statements of Values, and
  others in order to provide better guiding principles to our community.
 
[RFC 98]: https://github.com/NixOS/rfcs/pull/98
