---
feature: bootspec_v2
start-date: 2022-11-01
author: Ryan Lahfa (@raitobezarius)
co-authors: Linus Heckemann (@lheckemann)
shepherd-team:
- @06kellyjack
- @GovanifY
- @hesiod
shepherd-leader: @JulienMalka
---

# Summary
[summary]: #summary

Bootspec v2 is the second revision of the Bootspec document, introduced in [RFC-0125](https://github.com/NixOS/rfcs/blob/master/rfcs/0125-bootspec.md).

These facts are used as the primary input for bootloader backends like systemd-boot and grub, for creating files in `/boot/loader/entries/` and `grub.cfg`.

In this proposal, we aim to tackle known weaknesses of Bootspec v1, namely:

- Multiple initrds support
- Rework of initrd secrets mechanism
- Lack of devicetree
- Further iterations of Bootspec

This document describes **Bootspec v2**.

# Motivation
[motivation]: #motivation

The motivation of Bootspec v1 remains but we address the v1 weaknesses and
includes our experience of running with it for a while.

## Multiple initrds

The Boot Loader Specification, developed by the Userspace API group, allows the
specification of multiple initrd entries [1], with the purpose of merging them into
CPIO archives. This functionality is valuable in the systemd-stub ecosystem,
where various types of initrds are combined, including specific credential
initrds, global credential initrds, system extension initrds, PCR signature
initrds, and PCR public key initrds. Some initrds, such as credential initrds,
are dynamically generated from an EFI System Partition (ESP) location. [2]

Moreover, an additional initrd is frequently used to store CPU microcode. To
ensure compatibility and flexibility, it is essential to rework the initrd
support in the Boot Loader Specification. The proposed changes aim to treat
initrds as a set, allowing bootloaders to handle a list of initrds or a single
initrd in cases where multiple initrds are not supported.

[1]: https://uapi-group.org/specifications/specs/boot_loader_specification/#type-1-boot-loader-entry-keys
[2]: https://github.com/systemd/systemd/blob/main/src/boot/efi/stub.c#L778-L793

## Initrd secrets

Initrd secrets play a crucial role in the NixOS ecosystem, but they have raised
concerns, including security vulnerabilities (e.g., [GHSA-3rvf-24q2-24ww](https://github.com/NixOS/calamares-nixos-extensions/security/advisories/GHSA-3rvf-24q2-24ww
)) and
issues with booting in the Heads platform (e.g., [Issue #1348](https://github.com/linuxboot/heads/issues/1348.
)).

Initrd secrets serve to protect boot-time secrets from exposure within the Nix
store. This is achieved by using runtime scripts that append CPIO archives to
the generation's initrd during bootloader installation. However, the term
"initrd secrets" can be misleading, as the secrets are plaintext and can be
accessed from the ESP partition, offering limited confidentiality and no
integrity.

In practice, initrd secrets are often employed to establish stable fingerprints
for SSH servers within the initrd, or aiding in remote disk decryption on servers.
To address these issues, this RFC proposes moving away from the "appender
script" model in Bootspec v1 and instead adopting a hash map format to
represent secrets and their corresponding values.

This change allows for greater flexibility and enhanced security. For users in
the NixOS ecosystem relying on systemd and its semantics, the proposal suggests
offering a systemd-credentials approach for handling initrd secrets,
potentially enabling encryption using TPM2 if available, or a straightforward
key if TPM2 is not present. Additionally, this approach provides the means to
build using the `LoadEncryptedCredential` abstraction.

For those not using systemd, the hash map format offers full flexibility to
bootloader implementations to determine how to add the secrets, such as
appending to the initrd or other methods.

**Key Takeaway**: This RFC recommends removing the `initrdSecrets` field in
favor of letting consumers defining extensions replacing it. We define such an
extension `org.nixos.initrd-secrets.v1` and propose it to be a hashmap with
string keys and values. The key represents a name used to organize secrets, and
the value denotes an accessible path during bootloader installation.

### Expectations about initrd environment

In general, the semantics of initrd secrets lends themselves to expect the
existence of a files in the stage 1 environment of the initrd to be available
at certain paths.

With this proposed change, we specify that end users should expect that the
initrd environment will populate the referenced files of the initrd secrets
fields in the stage 1 environment at the very start.

More precisely, the predecessor of stage 1 environment is responsible for
filling the filesystem with the expected files, this can be the bootloader or
any prior stage to the stage 1 runtime environment.

For example, systemd credentials are populated by systemd at the very start in
an cooperation with `systemd-boot` and companion files present on the ESP.

Switching from a bootloader backend to another bootloader backend should have
no visible effect on that matter as long as the "files are present where they
are specified in the initrd secret field at the very start of the stage 1
environment" invariant is respected.

There is no specification about how to cleanup prior bootloader backend data
from a boot partition as this is out of scope for this RFC and multi-bootloader
cooperation is an open problem, especially for dual boots or more setups.

### Example

```
"org.nixos.initrd-secrets.v1": {
   "my-private-key": "/etc/nixos/secrets/wireguard-key"
}
```

## Device Tree and Device Tree Overlays

Non-x86 systems often rely on device trees to inform firmware or bootloader about the available devices and hardware support. 

There are two distinct requirements for device trees: 

- generic hardware support, where firmware or bootloader selects the appropriate device tree
- device-specific support, where a hardcoded device tree is required. 

The latter indicates a potentially problematic, non-upstreamed, or in-development platform.

The current Bootspec v1 does not formally encode information about hardcoded device trees or the folder containing available device trees. As a result, unformalized extensions are needed to address these fundamental use cases.

Regarding overlays in NixOS, any expressed overlay is incorporated into the final device tree. This eliminates the necessity to formalize an additional overlays field list, as overlays can be transformed into device trees as needed.

## Further iterations of Bootspec

It has been brought that the RFC process may not be adequate to discuss further
iterations of Bootspec.

While the author disagree with this vision and believe this would be a loss
for increased participation in the elaboration of further iterations and would lead to a 
decrease in discoverability of Bootspec matters.

We propose to move further iterations of Bootspec away from the RFC process.

For this, we propose to follow a lightweight process inside a GitHub repository
containing the specification and other relevant metadata about Bootspec.

In instances of controversy or difficulties to reach consensus among the Bootspec ecosystem
developers and maintainers, we should reach out to the standard RFC process.

# Goals
[goals]: #goals

- Improve non-x86 support in Bootspec, emphasizing the importance of
  devicetrees.
- Address and reduce the risks associated with initrd secrets, providing a
  default "secure" implementation within the systemd ecosystem and offering
  flexibility for other ecosystems.
- Enhance initrd flexibility, allowing developers to optimize their systems by
  supporting multiple initrds, including one for microcode and other specific
  purposes.
- Make further iterations of Bootspec easier to build.

### Non-Goals
[non-goals]: #non-goals

- Store TPM2-related information (hashes)
  We believe that Bootspec is still too immature for this and pcrlock
  (https://github.com/systemd/systemd/pull/28891) offers a more reliable and
  robust solution for generating signed PCR policies.
- Supporting SecureBoot.
  Secure Boot has one maintained implementation that is being upstreamed: https://github.com/nix-community/lanzaboote
  which was enabled by Bootspec v1.
- Specifying how to discover generations. This is desirable, but should not be tied to bootspec directly since bootspec may be useful with diverse discovery mechanisms.
- Address how bootloader backends should handle foreign bootloader data in their stead and manipulate it or how transition from a bootloader to another should happen.

# Proposed Solution
[proposed-solution]: #proposed-solution

- `initrd` will be removed from the v2
- `initrds` will be introduced as a list of initrd (compressed or uncompressed
  CPIO archives), this list can be empty, but the field is **required**
  nonetheless.
- `org.nixos.initrd-secrets.v1` will now be an **official extension** of
  Bootspec which should be an hashmap of strings (key) and strings (value)
  where the key is the "name" of a secret and the value is the **filesystem path** towards a
  secret.
- `fdtdir` will be introduced as an opaque string to a directory in a shape as
  the kernel outputs give them, e.g. U-Boot's extlinux directive `FDTDIR` is an
  example of this behavior — it is **optional**
- `devicetree` will be introduced as a path to a single devicetree that will be
  hardcoded — it is **optional**

All the Bootspec ecosystem will be updated as part of this specification. Regarding the transition period:

- All the Bootspec ecosystem is advised to **emit** the latest document.
- The synthesis feature should be upgraded to **emit** the latest version of the document based on the previous one, thus, by induction, based on no Bootspec document at all, if needed. 
  
This way, we guarantee forward and backward compatibility.

Finally, Bootspec will follow a lightweight process by default for its further iterations and rely on standard RFC process only for conflict resolution.

### Bootspec Format v2
[format-v2]: #format-v2


Using the following JSON:

```json5
{
  // Toplevel key describing the version of the specification used in the document
  "org.nixos.bootspec.v2": {
    // (Required) System type the bootspec is intended for (e.g. `x86_64-linux`, `aarch64-linux`)
    "system": "x86_64-linux",

    // (Required) Path to the stage-2 init, executed by the initrd (if present)
    "init": "/nix/store/xxx-nixos-system-xxx/init",

    // (Required) List of paths to the initrd, can be empty
    "initrds": [ "/nix/store/xxx-initrd-linux/initrd" ],

    // (Required) Path to the kernel image
    "kernel": "/nix/store/xxx-linux/bzImage",

    // (Required) Kernel commandline options
    "kernelParams": [
      "amd_iommu=on",
      "amd_iommu=pt",
      "iommu=pt",
      "kvm.ignore_msrs=1",
      "kvm.report_ignored_msrs=0",
      "udev.log_priority=3",
      "systemd.unified_cgroup_hierarchy=1",
      "loglevel=4"
    ],

    // (Required) The label of the system. It should contain the operating system, kernel version,
    // and other user-relevant information to identify the system. This corresponds
    // loosely to `config.system.nixos.label`.
    "label": "NixOS 21.11.20210810.dirty (Linux 5.15.30)",

    // (Required) Top level path of the closure, in case some spelunking is required
    "toplevel": "/nix/store/xxx-nixos-system-xxx",
    
    // (Optional) FDTDIR is assumed to be a path to a directory in the shape
    // of what `FDTDIR` in U-Boot extlinux would expect.
    // At the time of writing, it is assumed to follow the kernel output shape.
    "fdtdir": "/nix/store/xxx-uboot-fdtdir",
    
    // (Optional) devicetree is assumed to be path to a single devicetree file
    // which will be hardcoded for that generation.
    "devicetree": "/nix/store/xxx-arm64-machine/my-device.dtb"
  },
  // The top-level object may contain arbitrary further keys ("extensions"), whose semantics may be defined by third parties.
  // The use of reverse-domain-name namespacing is recommended in order to avoid name collisions.

  // (Optional) Specialisations are an extension to the specification which allows bundling multiple variants of a NixOS configuration with a single parent.
  // These are shaped like the top level; to be precise:
  //  - Each entry in the toplevel "org.nixos.specialisation.v2" object represents a specialisation.
  //  - In order for the top-level document to be a valid v2 bootspec, each specialisation must have a valid "org.nixos.bootspec.v2" key whose value conforms to the same schema as the toplevel "org.nixos.bootspec.v2" object.
  //  - The behaviour of nested specialisations (i.e. entries in "org.nixos.specialisation.v2" which themselves contain the "org.nixos.specialisation.v2" key) is not defined.
  //  - In particular, there is no expectation that such nested specialisations will be handled by consumers of bootspec documents.
  //  - Each specialisation document may contain arbitrary further keys (extensions), like the top-level document.
  //  - The semantics of these should be the same as when these keys are used at the top level, but only apply for the given specialisation.
  "org.nixos.specialisation.v2": {
    // Each key in this object corresponds to a specialisation as defined by the `specialisation.<name>` NixOS option.
    "<name>": {
      "org.nixos.bootspec.v2": {
        // See above
      }
    }
  },

  // (Optional) Hash map of desired secrets for that generation inside of the initrd.
  // Implementors of a bootloader installation procedure should examine their options
  // to securely make available the secret inside the initrd phase.
  // This may involve leveraging TPM2 via systemd-credentials or any measure you deem
  // to be reasonable in the context.
  // The legacy behavior is to prepare a CPIO archive for each file and
  // extend the `initrds` fields with those CPIO archives.
  // Make sure the location where the secrets are dropped in the initrd are visible
  // for the user.
  "org.nixos.initrd-secrets.v1": {
    "my-private-key": "/etc/nixos/secrets/wireguard-private-key",
  }
}
```

An *optional* field means: a field that is either missing or present, but **never `null`**.

### Risks
[risks]: #risks

- Some of the bootloader backends are quite complicated, and in many cases have
  inadequate tests. We could accidentally break corner cases.
- The bootloader backends are inherently a weak point for NixOS, as it is our
  last option for rolling back. We cannot roll back a broken bootloader. This
  and the previous point are risks, but also help demonstrate the value of
  reducing the amount of code and complexity in the generator.

### Milestones
[milestones]: #milestones

- Update Bootspec with the version 2 of that specification
  - The [Bootspec Rust library](https://github.com/DeterminateSystems/bootspec)
  - The [Bootspec interface in Nixpkgs](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/system/activation/bootspec.nix)
- Implement changes inside at least one bootloader backend.

# FAQ
[faq]: #faq

Familiarize yourself with [Bootspec v1](https://github.com/NixOS/rfcs/blob/master/rfcs/0125-bootspec.md) which may already contain answers.

## Why doing a RFC at all if it's not controversial?

Mentioned in https://github.com/NixOS/rfcs/pull/165#discussion_r1379020798 originally, we made it explicit
to the consumers of Bootspec that further iterations required a RFC. While we could have expected
that further iterations would not be controversial, we still wanted to go through to RFC process
to give reasonable chances to all the ecosystem.

While we disagree that there is no controversial change, as we can see on the DTB discussions here:
https://github.com/NixOS/rfcs/pull/165#discussion_r1379666004.

Further iterations as mentioned in the body of that RFC will be moved to a repository
and we will invite all the people interested into Bootspec to subscribe to that repository
for further developments.

## How will I be able to express complicated logic for initrd secrets, e.g. dynamic secrets?

Some users may have used the appender script to provide dynamic logic that
provides a secret at activation time rather than storing it on the long run.

While this usecase is interesting, it is very advanced and the footgun that
initrd secrets represent cannot be made up by our only goal of supporting that
usecase.

Authors may propose to think about how a filesystem could implement the dynamic
fetching at activation time, e.g. a FUSE secretfs that will dynamically query
the secret engine for a secret and make it available for short time.

If any logic requires user interaction, it is preferable to invest in a custom
bootloader installer logic and use the static fields to refer to secrets that
will be requested.

Finally, as the new initrd secrets are implemented as an **extension** of
Bootspec, nothing prevent an end user to define an non-official extension of
Bootspec to address their own special needs.

# Open Questions
[open-questions]: #open-questions

- Should the initrd secrets work itself with systemd-credentials to load
  further credential and have a closed loop of credentials, this would require
  the activation to run inside systemd:
  https://github.com/NixOS/nixpkgs/pull/258571 !
- What are lessons we can learn from
  https://github.com/aarch64-laptops/edk2/tree/dtbloader-app#dtbloader for
  devicetrees manipulation?

# Future Work
[future]: #future-work

- Continue the migration from filesystem-spelunking into using the bootspec
  data.
- Implement a systemd-credentials based of `org.nixos.initrd-secrets.v1`
- Implement an [Verified Boot for
  Embedded](https://u-boot.readthedocs.io/en/latest/develop/vbe.html)
  bootloader installation script using bootspec data
