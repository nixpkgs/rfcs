---
feature: intensional_store
start-date: 2017-08-11
author: Wout.Mertens@gmail.com
co-authors: (find a buddy later to help our with the RFC)
shepherd-team: Shea Levy, Vladimír Čunát, Eelco Dolstra, Nicolas B. Pierron
shepherd-leader: Shea Levy
related-issues: (will contain links to implementation PRs)
---

# Intensional Store

## TODO / to explain

- flesh out Trust DB locations and updating/querying/merging multiple
- query service
- efficient distribution of mappings
- explain the benefits of late binding and how it improves installs on low-power systems
- script that uses nix make-content-addressable in /var/lib/nix to generate a store
- how binary caches provide \$cas and mappings

## Summary

This RFC builds on the implementation of RFC 62 to maximize its benefits.

- Decouple metadata (Trust DB) from the Store, keep per-user
- Move the Store to `/var/lib/nix`
- Content checksums include the runtime dependencies
- Store can be shared read-write on a network share
- `nix-daemon` becomes optional
- Store paths optionally provide dependency list in-band

### Benefits

By using content hashes instead of output hashes, we can:

- optimize resource usage
- reduce binary substitution trust to a single lookup
- make the Nix store network-writeable and world-shareable
- predefine mappings from output to content hash without building
- store paths can be verified without access to the Nix Store DB
- public trust mappings allow detecting non-reproducible builds
- easily switch between single- and multi-user setup

Additionally, this is an opportunity to move the Nix store to a filesystem location supported by most non-NixOS systems, namely `/var/lib/nix`.

By "cleaning up" the filesystem state of Nix, a host of possibilities emerge:

- Boot a cloud VM to a specific system by passing a `$cas` name for stage2. The stage1 will auto-download the stage2 if it's missing and switch to it.
- Cross-compiling can generate `$cas` entries that are reused for native compiles via `$out` mapping. This is useful on low-resource platforms.
- The Nix store doesn't require any support or metadata. On embedded systems, all management of the store can be performed outside the system.
- References to `$cas` entries, such as profiles, are no longer tied to a single system.
- A FUSE filesystem could auto-install `$cas` entries as they are referenced, hanging the I/O until the entry is downloaded and verified.
- You can copy a store from some other install, and immediately use profiles without having their metadata.
- Different Nix tooling and metadata implementations can use the same store

… and so on. Decouplying systems brings exponential possibilities.

### Drawbacks

There are some small drawbacks:

- We have to assume that we can always create a working `$cas` derivation, even if the build stores the build path in an opaque way. Otherwise, their build path would have to be a symlink to the `$cas` entry.
- By removing the derivation name from the store paths, the store becomes more opaque and requires good tooling for manual management.
- Garbage collection is more complex when the store is shared between hosts.
- An attacker can provide malicious `$out` => `$cas` mappings. This is already somewhat possible via binary caches. To remedy, trust mappings might be signed by a trusted key.
- We have to assume that a `$cas` collision is impossible in practice.
- `$cas` entries without metadata are opaque, and might contain malware or illegal content. If nothing references it, there is no problem with the content. Garbage collection takes care of unused entries.
- A hash collision would allow inserting malware into a widely used `$cas`. This is already possible without `$cas`, but trusting the hashes may lead to wider cache use. Remedies include using secure hashes, scanning for malware, using multiple hashes and comparing between binary caches, …

### Terminology

We assume the following process when wanting to install a given package attribute `$attr`:

- Nix evaluates the desired expressions and determines that a certain output hash `$out` is required
- `$out` is looked up in the Trust DB, to possibly yield `$cas`, its Content Addressable Storage hash.
- If `$cas` is known:
  - If `$cas` is present in the store, `$attr` is already installed; Done.
  - If `$cas` is present on a binary cache, it is downloaded to the store; Done.
- `$out` is built using the normal mechanisms (see RFC 62 for more details)
- Its `$cas` is calculated (see Preparing for more details).
- The generated metadata is stored in the Trust DB; Done.

### A note on reproducibility

There is no need for a given `$out` to always generate the same `$cas`. It allows better resource use, but doesn't change anything about this RFC. There is no obligation that a single `$out` only stores a single `$cas` entry.

## Nix Store

## FHS compatibility

Since we're working on the store layer, we have the opportunity to split up the current `/nix` directory and make it FHS compliant. This makes it easier to use Nixpkgs where creating `/nix` is not possible.

An [informal discussion](https://discourse.nixos.org/t/nix-var-nix-opt-nix-usr-local-nix/7101) concluded that the Store should be located at `/var/lib/nix` for maximum compatibility.

As for the contents of `/nix/var`, all of it can go elsewhere:

- `/nix/var/log` should go under central or per-user log.
- `/nix/var/nix`:
  - `db`: Store database. Its contents will be spread across Trust DBs.
  - `daemon-socket`: Builder service. Should move to appropriate location for service sockets, like `/run`
  - `gc`: See the Garbage Collection section
  - `profiles`: Are now maintained per user/system
    - User profiles, channels etc go under `$NIX_CONF_DIR/{profiles,channels,auto_roots}` per user
    - System profiles, default user profile, channels etc go under `/nix/var/nix-profiles/{system,user,channels,auto_roots}`
  - `temproots`, `userpool`: Builder service. Should move to appropriate locations for services, like `/var/tmp` and `/var/lib`

### Contents

The Store should be verifiable, and only contain verifiable paths. However, to allow atomic installation over the network, there should be a directory for staging an installation. Some other operations also need supporting directories.

For cosmetics and wildcard expansion, we hide supporting directories from regular view.

Therefore, these are the Store contents, all part of the same mount point to ensure atomic semantics:

- `$cas`: a self-validating derivation. Any path matching the proper hash length is subject to verification at any time, and is be moved to `.quarantaine` if verification fails
- `.prepare`: this directory can be used by anyone to prepare a derivation before adding it to the Store, by picking a non-conflicting subpath
- `.stage`: after preparing, the derivation is moved here
- `.daemon`: if there is a store daemon, it might use this path to prepare installation
- `.quarantaine`: whenever a non-compliant path is encountered, it is moved here
- `.links`: used to hard-link identical store files
- `.gc`: used to communicate about garbage collection
- anything else doesn't belong in the Store and should be removed

The timestamps of files/directories are kept 0, and the user and group ownership are recommended to be a single user, for example `root:root` or `store:store`.
Note that for a shared store, two systems might see different ownership values; this is acceptable.

### Metadata

For a given `$cas` entry, there is objective and subjective metadata.

Objective metadata examples:

- `$cas`
- output size
- late binding information
- runtime dependencies (special case)

Subjective metadata examples:

- `$out`
- name, version
- build timestamp and duration
- build-time dependencies
- nixpkgs commitish (not always applicable)
- name of the attribute in nixpkgs
- own configuration commitish
- list of flakes involved

Objective metadata is a function of the entry contents only and can be calculated by anyone.

Subjective metadata is a function of the build description and execution. It depends on the source, is trusted data, and any desired metadata should be stored in the Trust DB.

#### Runtime dependencies

Runtime dependencies (the Store entries that are necessary for correct operation of this entry) are a special case, because you can't always detect those based on the contents of the entry.
However, if we make the dependencies part of the `$cas` calculation, differences in those will result in a different `$cas`, and so for a given `$cas` there can be only one list of correct runtime dependencies.

Furthermore, in order to know which Store entries belong together, the runtime dependencies are a necessity. Therefore we must provide them as part of the entry. Since we can't change the entries themselves (be they files or directories), we add the metadata as an optional file named `/var/lib/nix/$cas.m`. This file is included in the `$cas` calculation. If the file is missing or altered, the `$cas` won't validate.

The metadata file currently only contains the `$cas` hashes of each runtime dependency, sorted alphabetically, and separated by single newlines. If the file does not match this format exactly, it is invalid.

Other objective metadata that could be useful, such as late binding information, could be placed in this file too, perhaps as a TOML file that begins after the first double newline. This is to be determined.

### Trust DB

The Trust DB contains subjective metadata for `$out` -> `$cas` mappings. Each entry contains:

- `$cas`
- an array of `$out`
- name (including version)
- size
- array of dependency `$cas`
- optional:
  - description
  - build timestamp
  - builder
  - custom metadata, like one or more git repo commit hashes

With this information, a user can quickly find `$cas` entries to install that match a name or description. `nix-build` can find a `$cas` by `$out`.

For a given `$cas`, there can be many entries, one for each trusted source. This can be handled by having one SQLite DB per source (including localhost), and having an order of precedence.

### Sharing the Nix Store

Since the Nix Store (minus supporting directories) contains only self-validating paths, it can be shared "infinitely", only limited by:

- disk space
- network performance
- confidence around hash collision attacks
- confidence around writers corrupting paths without detection

The installation step only involves moving a proposed path from `.prepare` to `.stage`, so no further communication is necessary with the Store daemon.

For single-user installs, the Store can trivially be maintained by the Nix tools, and converting to multi-user is only a matter of changing the permissions.

It would even be possible to use FUSE to automatically download any paths that are referenced in the Store, hanging the I/O request while it's being downloaded.

### Store Daemon

Optionally, a daemon can maintain the Store. In this case, it is recommended be the only user with write access. It performs installations, verifications and garbage collection, described below.

### Preparing

`$cas` entries are prepared by building them in a path that has the same total length as its final `$cas` entry. This means they can be built practically anywhere, in `/tmp`, in a home directory, in the `.prepare` directory in the Store, etc. It is recommended to make a part of the path a unique-per-build string.
To make sure only the build's own references need rewriting, it is recommended to build using only `$cas` entries as dependencies, instead of relying on rewriting paths.

After the build, the runtime dependencies file is generated in-memory (if there are dependencies), the `$cas` is calculated including the runtime dependencies buffer, any occurences of the build path are replaced with `/var/lib/nix/$cas`, and the `$cas.m` file is written (if there are dependencies).

If there undetected build path references, they might cause the finished entry to work incorrectly, and they will cause `$cas` to differ on every build of `$out`. This must be handled on a case-by-case basis. Perhaps we'll need pluggable hash rewriters.

The build can happen by a sandboxing build daemon like `nix-build`, but that is not a requirement.

After preparing, the metadata for the build is added to the user's Trust DB.

### A note on `$cas` calculation

The content hash has to be built up recursively, as described elsewhere. There are some caveats:

- When the build path `$build/$out` is encountered, it should be replaced with `/var/lib/nix/$placeholder`. `$placeholder` is a fixed string used in all `$cas` calculations.
- When `$out` is encountered but not `$build/$out`, it should be replaced with `$placeholder`.
- When `$build` is encountered but not `$out`, it is a possible problem and should be noted.
- The paths of files and directories should be included relative to `$build` (so `$build/$out` is `/$out`), but not in the file contents or symlink targets.
- If there is metadata, the file should be included in the calculation as if it was named `_meta.m`, stored next to `$out`.

Changing the entry to its `$cas` name means:

- Rename `$out` to `$cas`.
- Replace the string `$build/$out` with `/var/lib/nix/$cas`.
- Replace the string `$out` with `$cas`.
- If there is metadata, write the contents to `$cas.m`.

When validating:

- The paths of files and directories should be included relative to `/var/lib/nix`.
- If `$cas.m` exists, it should be included in the calculation as if it was named `_meta.m`.
- When the string `$cas` is encountered, it should be replaced with `$placeholder`.

### Installation

We use rename semantics to provide atomic installations. Prepared `$cas` entries are moved to their final location with a `rename` call, which is atomic but requires the path to be on the same filesystem.

Atomicity is important to ensure that `$cas` entries are always valid. If they are copied instead, they don't self-validate for the duration of the copy.

#### with Store Daemon

Any user with write access to `/var/lib/nix/.prepare` and `/var/lib/nix/.stage` can ask for entries to be installed. To do so:

1. They prepare entries to be stored in `/var/lib/nix/.prepare`, renaming each entry to its `$cas` and `$cas.m`.
1. They atomically move prepared paths to `/var/lib/nix/.stage`, in reverse dependency order, meaning dependencies of an entry are moved first. First the `$cas.m` file is moved and then the `$cas` entry.

When the Store daemon discovers a new `$cas` entry under `.stage`:

1. If the Store already contains this `$cas` entry, it removes this new one, perhaps first verifying the Store copy.
1. It recursively changes ownership of `$cas` and `$cas.m` to itself and timestamps to 0, making sure that write permission is removed for everybody, and read permission is added for anybody.
   If it has no permissions to do this, it instead copies the path into `/var/lib/nix/.daemon`, and another process will need to keep `.stage` clean.
1. The daemon verifies the `$cas`. If it doesn't match, it removes `$cas` and `$cas.m`. Note that a missing or altered `$cas.m` file won't pass validation.
1. If `$cas.m` exists, it checks that all dependencies are already present in the Store. If not, the path is held for a while and deleted if the dependencies don't appear in time (configurable).
1. It atomically moves `$cas.m` into `/var/lib/nix`.
1. It atomically moves `$cas` into `/var/lib/nix`.

Note that to ensure atomicity, `.prepare` and `.stage` need to be on the same filesystem, and either `.stage` or `.daemon` need so be on the same filesystem as the Store.

#### without Store Daemon

Any user with write access to `/var/lib/nix/.stage` and `/var/lib/nix` can install entries. To do so:

1. They prepare entries to be stored in `/var/lib/nix/.stage`, renaming each entry to its `$cas` and `$cas.m`.
1. They atomically move prepared entries to `/var/lib/nix`, in reverse dependency order, meaning dependencies of an entry are moved first, and `$cas.m` is moved before `$cas`

Note that to ensure atomicity, `.stage` needs to be on the same filesystem as the Store.

Note that when two writers are trying to install the same `$cas` or `$cas.m`, one of them might get an error, but the end result will be the same (as long as the `$cas` is self-valid). So multiple writers can also be on separate hosts, in a trusted setting.

### Verification

A path in the Store is verified by calculating its `$cas` (see above). If the `$cas` doesn't match, the path is moved to `/var/lib/nix/.quarantaine`, where a sysadmin has to investigate.

Any process with write access to `/var/lib/nix` and `/var/lib/nix/.quarantaine` can do this, for example the Store daemon.

### Garbage collection

Garbage collection needs to identify store paths that are not used by anything on any of the systems sharing the same store. Here we propose a simple mechanism for coordination, but any mechanism is acceptable.

- A host with store write access decides to run garbage collection.
- It checks that `.gc/running_gc` does not exist or contains a very old timestamp, and writes a unique number to `.gc/will_gc`.
- After waiting long enough to prevent collisions (for example 10 seconds), it reads `.gc/will_gc` and verifies it contains the unique number it wrote.
- It clears out `.gc/` except for the file `.gc/will_gc` and adds the file `.gc/running_gc` containing the current timestamp.
- While it waits for other hosts, it checks the Store for `$cas.m` files that don't have a matching `$cas`.
- Each host's store daemon monitors `.gc/running_gc` at some interval, for example 1 minute.
- While this file exists, the daemon must record its root `$cas` entries, by creating 0-length files named `.gc/$cas`.
- The writer waits long enough for all the hosts to record their GC roots, for example 10 minutes.
- It verifies that `.gc/will_gc` still contains its unique number
- After the wait period expired, the writer host scans for store paths that are part of the own and other GC roots. Each `$cas` is atomically moved to `.gc` and deleted; `$cas.m` is also deleted.
- The `$cas.m` files that still don't have their matching `$cas` are removed. Note that when installing, the `$cas.m` will appear shortly before `$cas` since everything is prepared.
- Finally, the writer host empties the `.gc` directory, leaving the `running_gc` file for last.

For a single-user installation or a non-shared Nix store, none of this is necessary, and the GC process remains unchanged, except for the new locations to search for GC roots. See next section.

## Profiles

A profile is a named reference to a `$cas` store path, to be used in arbitrary ways. To allow GC, the Store must know about all profiles, so they should be available at predictable paths.

The current versioning system is reused: A profile with the name `$profile` is a symlink which points to `$profile-$v-link`, which is a symlink that points to the absolute path of the `$cas` entry. Tools like `nix-env` and `home-manager` can maintain the set of links for a profile.

Since a profile points to an immutable `$cas` path, it is the same across systems and can therefore be part of a network-mounted home directory.

However, a profile link itself is trusted information, and should be shared between users and systems only when they trust each other.

Known paths for profiles:

- `$NIX_CONF_DIR/profiles`: per-user profiles and auto roots
- `/var/lib/nix-profiles`: NixOS system and shared profiles, maintained by the `root` user

Other paths can of course be used, but the GC won't know about them, so a link should be maintained in one of the directories above.

For enabling garbage collection, these directories also contain "auto-roots", generated from builds.

Example: This creates or updates the user-specific profile "vscode", adding Python 3:

```sh
nix-env -p ~/.config/nix/profiles/vscode -i python3
```

## Administration Tasks

### Migration

There is no real need for migrating stores, since `/nix` and `/var/lib/nix` can coexist and the tooling either uses one or the other. However, it is convenient to migrate built artifacts for implementing this RFC.

To migrate an existing output from `/nix/store/$out-$name` to `/var/lib/nix/$cas`, the following approach will work most of the time:

- migrate all its dependencies using the below steps
- generate the metadata file if there are dependencies
- calculate the `$cas` as described above, but instead of handling `$build/$out`, replace all strings of the form `/nix/store/$out-$name` with `/var/lib/nix/$filler/$cas`. `$cas` has the same length as `$out` and the minimum length of `$name` is 1, so there is always room for the full `$cas`. The `$filler` is a string with length `l = length($name) - 1` of the form `./././/`, that is, repeat `./` `floor(l/2)` times and append `/` if `l` is odd.
- do the same with symlinks, but consider relative paths as well
- self-references must be updated after `$cas` is known, as described above
- once `$cas` is determined, the patched derivation can be placed in `/var/lib/nix/$cas` and the metadata recorded in the Trust DB

This process will fail if the derivation refers to the Store in ways that aren't visible, like different string encoding and calculated paths.

### Adding a Store Daemon

To begin managing an existing Store with a Store Daemon, these steps are performed:

- Change permissions on the Store root so only the daemon has write access.
- Ensure `.prepare`, `.stage` and `.quarantaine` with desired permissions.
- For each Store entry
  - Recursively adjust permissions and timestamps
  - Verify entry
    - If invalid, move to `.quarantaine` and try to download replacement from known caches

### Removing a Store Daemon

- Wait for pending installs to complete.
- Stop Store Daemon.
- Change permissions on the Store as desired.

### Repairing an entry

If a `$cas` entry does not validate, it can be downloaded from any cache that has it, without trust. If no cache has it, it cannot be repaired.

## Implementation

- NixPkgs needs to be audited to remove hard-coded `/nix` names, replacing it with the store path variable (TODO look up name).
- The Nix tools and Hydra need to be be branched to support the new location and store semantics. The tools either use the old location and semantics, or the new one.
- The binary cache server needs to serve `$cas` entries as compressed files that include the entry and the `$cas.m` file.
- The trust mapping server needs to serve trust mappings in an incremental way. For example, as a JSON array of added and changed mappings since some timestamp.

## Alternative options

There are a few choices made in this RFC, here we describe alternatives and why they were not picked.

### Keep store at `/nix/store`

Since the names are always different, the `$cas` entries and `$cas.m` files could stay in `/nix/store`. The benefit would be that NixPkgs doesn't have to be audited for hardcoded `/nix` paths.

However, this keeps the problem of some installations not having permission to create a `/nix` directory, and makes it much harder to share the store between hosts (as long as non-`$cas` entries are present).

### No metadata

Not keeping metadata in the Store means that the Store by itself doesn't have enough information to do garbage collection, nor to let a system boot from a given `$cas`

### Keep metadata in directory with Store entry

Since the Store entries can be files or directories, that means that files would have to be put in a directory, for example `$cas` becomes `$cas/_`.
Then directory entries would have to do the same for symmetry. This requires many code changes and requires extra storage, even if an entry doesn't have any runtime dependencies.

### Add name to metadata

The name and version are subjective data. Adding it to metadata would cause `$cas` to change and increases storage. The only consumer of this data is the end user, who should also have the data available in their Trust DB.
