---
feature: redistribute-redistributable
start-date: 2024-12-15
author: Ekleog
co-authors: (find a buddy later to help out with the RFC)
shepherd-team: (names, to be nominated and accepted by RFC steering committee)
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: https://github.com/NixOS/nixpkgs/issues/83884
---

# Summary
[summary]: #summary

Make Hydra build and provide all redistributable software, while making sure installation methods stay as fully free as today.

# Motivation
[motivation]: #motivation

Currently, Hydra builds only free software and unfree redistributable firmware.
This means that unfree redistributable software needs to be rebuilt by all the users.
For example, using MongoDB on a Raspberry Pi 4 (aarch64, which otherwise has access to hydra's cache) takes literally days and huge amounts of swap.

Hydra could provide builds for unfree redistributable software, at minimal added costs.
This would make life much better for users of such software.
Especially when the software is still source-available even without being free software, like MongoDB.

# Detailed design
[design]: #detailed-design

We will add a `runnableOnHydra` field on all licenses, that will be initially set to its `free` field, and set to `true` only for well-known licenses.

Hydra will build all packages with licenses for which `redistributable && runnableOnHydra`.
It will still fail evaluation if the ISO image build or the Amazon AMIs were to contain any unfree software.

This will be done by evaluating Nixpkgs twice in `release.nix`.
Once with `allowUnfree = false` like today, plus once with `allowlistedLicenses = builtins.filter (l: l.redistributable && l.runnableOnHydra) lib.licenses`.
Then, most of the jobs will be taken from the allowlisted nixpkgs, while only the builds destined for installation will be taken from the no-unfree nixpkgs.

The list of jobs destined for installation, that cannot contain unfree software is:
- `amazonImage`
- `amazonImageAutomaticSize`
- `amazonImageZfs`
- `iso_gnome`
- `iso_minimal`
- `iso_minimal_new_kernel`
- `iso_minimal_new_kernel_no_zfs`
- `iso_plasma5`
- `iso_plasma6`
- `sd_image`
- `sd_image_new_kernel`
- `sd_image_new_kernel_no_zfs`

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

With these changes, here is what could happen as things currently stand, if the licenses were all to be marked `runnableOnHydra`.
This is not meant to be indicative of what should happen or not, but indicative of what could happen.
Each package's individual `license` field setup is left to its maintainers, and nixpkgs governance should conflict arise.
This RFC does not mean to indicate that it is right or wrong, and is not the right place to discuss changes to this field.
Should one have disagreements on any specific package in this list, please bring that up to that package's maintainers.

It is also suggested in this RFC that people, upon marking licenses as `runnableOnHydra`, check all the derivations that use this license.
They could then have to mark them as either `hydraPlatforms = []`, `preferLocalBuild = true` and/or `allowSubstitutes = false`.
This might be useful for packages like TPTP:
they may not yet be marked as such due to these flags having no impact on unfree packages;
but would take gigabytes on Hydra for basically no local build time improvement

With this in mind, Hydra could start building, among others:
- CUDA
- DragonflyDB
- MongoDB
- Nomad
- NVIDIA drivers
- Outline
- SurrealDB
- TeamSpeak
- Terraform
- Unrar
- Vagrant
- NixOS tests that involve such software (eg. MongoDB or Nomad)

And Hydra will keep not building, among others:
- CompCert
- DataBricks
- Elasticsearch
- GeoGebra
- Widevine CDM

# Drawbacks
[drawbacks]: #drawbacks

The main risk is that NixOS could end up including unfree software in an installation image if:
1. we forgot to add it to the list of no-allowed-unfree jobs, and
2. a maintainer did actually add unfree software to that build.

This seems exceedingly unlikely, making this change basically risk-free.

The only remaining drawback is that Hydra would have to evaluate Nixpkgs twice, thus adding to eval times.
However, the second eval (with no-unfree) should be reasonably small and not actually evaluate all packages, as it is only used for installation media.

# Alternatives
[alternatives]: #alternatives

### Having Hydra actually only build FOSS derivations, not even unfree redistributable firmware

This would likely break many installation scenarios, but would bring us to a consistent ethical standpoint, though it's not mine.

### Keeping the status quo

This results in very long builds for lots of software, as exhibited by the number of years people have been complaining about it.

### Having Hydra redistribute redistributable software, without verifying installation media

This would be slightly simpler to implement, but would not have the benefit of being 100% sure our installation media are free.

### Having Hydra redistribute redistributable software, with a check for the installation media

This is the current RFC.

### Building all software, including unfree non-redistributable software

This is quite obviously illegal, and thus not an option.

### Not having the `runnableOnHydra` field on licenses

This would make it impossible for Hydra to build them as things currently stand:
Hydra would then risk actually running these packages within builds for other derivations (eg. NixOS tests).

This would thus only be compatible with changes to Hydra, that would allow to tag a package as not allowed to run, but only to redistribute.
Such a change to Hydra would most likely be pretty invasive, and is thus left as future work.

# Prior art
[prior-art]: #prior-art

According to [this discussion](https://github.com/NixOS/nixpkgs/issues/83433), the current status quo dates back to the 20.03 release meeting.
More than four years have passed, and it is likely worth rekindling this discussion, especially now that we actually have a Steering Committee.

Recent exchanges have been happening in [this issue](https://github.com/NixOS/nixpkgs/issues/83884).

# Unresolved questions
[unresolved]: #unresolved-questions

Is the list of installation methods correct?
I took it from my personal history as well as the NixOS website, but there may be others.
Also, I may have the wrong job name, as I tried to guess the correct job name from the various links.

### How large are the packages Hydra would need to additionally store?

`nix-community`'s Hydra instance can give us approximations.
Its `unfree-redist-full` channel is currently 215G large, including around 200G of NVidia kernel packages and 15G for all the rest of unfree redistributable software.
Its `cuda` channel is currently 482G large.

It might make sense not to build the cuda/nvidia builds on Hydra for now, at least not before culling a bit the packages, considering there are eg. more than a hundred different NVidia kernel packages.
Hence, this RFC suggests not setting `runnableOnHydra` for the relevant derivations before cleaning up a bit the NVidia ecosystem in nixpkgs.

# Future work
[future]: #future-work

Modifying Hydra to allow building and redistributing packages that it is not legally allowed to run.
This would be a follow-up project that is definitely not covered by this RFC due to its complexity, and would require a new RFC before implementation.
