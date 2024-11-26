**Feature:** Enhance package meta fields to dictate mergebot behavior  
**Start Date:** `[Today's Date in YYYY-MM-DD format]`  
**Author:** Lassulus  
**Co-authors:** Scriptkiddi (additional co-author to be determined)  
**Shepherd Team:** `[To be nominated and accepted by the RFC Steering Committee]`  
**Shepherd Leader:** `[To be appointed by the RFC Steering Committee]`  
**Related Issues:** `[Links to implementation PRs will be added here]`

## Summary

This RFC proposes the introduction of four new merge strategies for `nixpkgs`, aiming to provide more nuanced merge permissions.
One of the main goals of this RFC is to extend the capabilities of the mergebot beyond PRs of just r-ryantm.
These strategies facilitate automatic updates and backports from trusted sources and allow maintainers the choice between requiring full consensus or just a single approval for merges.

## Motivation

![](https://pad.lassul.us/uploads/f24e0ff3-117a-4167-8f60-79ca508b8c1c.png)

The current `nixpkgs` contribution model relies heavily on a group of *committers* (approximately 200 at the time of writing), who possess exclusive rights to merge PRs.
This model has remained unchanged since its inception and poses limitations on community contributions and maintainer workload.
Implementing granular merge permissions will enable a broader contribution base and lessen the burden on existing committers by empowering package maintainers with more control over their respective packages.

## Detailed Design

The current design of the mergebot is limited in functionality. To tackle this issue we propose to implement per package strategies. These would be set inside a meta attribute in the package.

We propose augmenting package meta fields with a new attribute set that specifies a list of merge strategies.
The mergebot will execute all listed strategies in parallel and will proceed with the merge upon one successful strategy, contingent on passing ofborg checks as well.

![](https://pad.lassul.us/uploads/965545d8-8575-41a8-9746-6bcc66f8bb0e.png)

### Proposed `mergeBot` Attribute Structure

```nix
meta = {
  mergeBot.strategies = ["automerge-updates", "full_consensus", "..."];
}
```
### Merge Strategies
Currently supported strategies by nixpkgs-merge-bot as of the time of writing:
- **maintainer_update:** Currently, this strategy allows package maintainers to merge updates from r-ryantm, triggered by mention of the bot.

Proposed strategies are:

- **automerge_updates:** Merges PRs from r-ryantm automatically if ofBorg checks pass, triggered by PR creation or mention. We strongly recommend to have at least one passthrough test.
- **automerge_backport:** Automatically merges backport PRs if ofBorg checks pass, triggered by PR creation or mention.
- **full_consensus:** Requires approval from all maintainers before merging. The bot will notify which maintainers' approvals are pending, triggered by PR approval.
- **single_maintainer:** Allows a single maintainer's approval to trigger a merge, facilitated by a mention of the bot.

### Limitation to `pkgs/by-name` Directory

To ensure accuracy in processing PRs, only those within the `pkgs/by-name` directory will be considered, due to the challenge of verifying if a PR solely affects the intended files.

### Future extensions

For now we propose that new merge strategies or significant changes to current ones go through their own RFCs. The mergebot team is deciding what is deemed significat in that case.

## Examples and Interactions

1. **Single Maintainer Mode:** A maintainer of the `nano` package, set with `strategies = ["single_maintainer"]`, can initiate a merge by commenting with a specific bot command. The bot merges the PR if the ofBorg checks successed.

2. **AutoMerge Mode:** An update PR for `ttyplot` created by r-ryantm, with `meta.mergeBot.strategies` set to `[ "automerge_updates" ]`, gets automatically merged after successful ofBorg checks.

## Drawbacks

- **Security Risks:** There's a potential for malicious actors to manipulate the merge bot to authorize merges improperly. A mitigation for these pull requests could be running validity checks (e.g file size limits, valid nix files, ..)
- **Reliability Concerns:** A malfunction in the merge bot could disrupt numerous contributors' workflows.
- **Code Quality** As no review is done by seasoned commiters, the code quality could drop. We want to face that by holding monthly meetings and review the merged PRs to see if the code quality drops and improving on the bot in that case.

## Alternatives

- **Maintain Current Model:** This approach may not be sustainable long-term due to scalability issues.
- **External Tools:** Utilize third-party GitHub actions or apps, like Mergify, and adapt them to our needs.

## Prior Art

https://github.com/NixOS/rfcs/pull/50
https://github.com/NixOS/rfcs/pull/128

## Unresolved Questions

*To be completed*

## Future Work

- **Strategy Expansion:** Developing additional merge strategies and drafting RFCs to introduce them.
- **Enhanced Documentation:** Creating comprehensive documentation for the new merge strategies.
