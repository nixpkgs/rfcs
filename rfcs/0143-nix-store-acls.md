---
feature: nix-store-acls
start-date: 2023-01-16
author: Alexander Bantyev
co-authors: Silvan Mosberger, Théophane Hufschmitt
shepherd-team: John Ericson, Théophane Hufschmitt, Eelco Dolstra
shepherd-leader: Théophane Hufschmitt
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

Implement a way to only allow user access to a store path if they provide proof that they have all the necessary sources available, or had the access permission explicitly granted to them.

# Motivation
[motivation]: #motivation

Currently, the Nix Store on a local machine is world-listable and world-readable.

This means that it's not possible to share a machine between multiple users who want to build proprietary software they don't want other people to look at.

Also, it makes storing secrets in the Nix store even more dangerous then it could be.

Finally, it means that substituter access is all-or-nothing: either a user can access the cache and download everything there is in there just by knowing store paths (even without having the source code or Nix expressions available), or they can't download anything.

This can be rectified by adding access control list (ACL) functionality to Nix, making it possible to set ACLs on store paths.

This RFC targets the use-case of protecting proprietary software on a shared nix store or substituter. It could potentially be used to store secrets (such as passwords) in the Nix store, but this is potentially succeptible to some attacks; for example, some store paths are content-addressed (including files simply imported into the store), thus it may be possible to brute-force their contents based on the world-readable hash part of the store path. It is definitely **not** intended for storing small (<64 bits of entropy) secrets.

# Detailed design
[design]: #detailed-design

Change the implementation of the Nix daemon (and, potentially, nix-serve, depending on the chosen "Remote Store" implementation) to:

- Apply necessary [POSIX ACLs](https://man7.org/linux/man-pages/man5/acl.5.html) to store paths, automatically update them when users provide proof that they have the necessary source, and allow `trusted-user`s and users with access to the paths in question to manipulate those ACLs manually.
- Add a setting (perhaps `protect-by-default`) to protect all new store paths by default.

An additional invariant that must be ensured at all times is that the complete runtime closure of the store path is available to a user if the store path itself is available.

This should ensure that this change is as seamless as possible for the users: they will still always be able to execute `nix build` or similar for derivations where they have access to all the sources, substituting as much as possible, as though nothing had changed.
If a user needs to be able to access some store paths without having access to their sources (e.g. for proprietary software where sharing the artifacts is ok but sharing the sources isn't), such access can be granted explicitly by the administrators (`trusted-user`'s) or users with access to said path.

Additionally, [RFC 97] can be implemented for some extra security.

[RFC 97]: https://github.com/NixOS/rfcs/pull/97

## Local store

For paths external to the Nix sandbox (added via `nix store add-{file,path}`, paths in the Nix language, `builtins.fetch*`, or flake sources), we add the user to the list when they send the path to the daemon.
We might need to add a flag (like `--protect`) for the selective ACL mode.

For derivations themselves (.drv files), we add the user to the list when they send the derivation to the daemon.

For derivation outputs, we add user to the list when the user requests to realise the derivation and has access to the transitive dependencies, including the sources.

Protected paths and derivation outputs should not be readable by anyone during the build.
Necessary permissions are granted after the build.

There also should be a way to enable the protection (perhaps `nix store access protect`), explicitly grant (`nix store access grant`), and revoke (`nix store access revoke`) access of certain user or group to each individual path.
Naturally, this should only be available to `trusted-user`s and users who already have access to this path.
Additionally, only trusted users should be able to protect a path or revoke permissions, since that could potentially break things for other users.

### Nix language

We define an "access control status" as an attrset `{ protected : bool; users : [string]; groups : [string]; }`.

We change the `derivation` builtin, adding a `__permissions` argument. This argument is supposed to be of form `{ drv : access control status; outputs.<name> : access control status; log : access control status; }`, each attribute being optional. The attributes correspond to the desired permissions of the derivation itself (`.drv` file), outputs of the derivation, and the build log.

We change the `path` builtin, adding a `permissions` argument, which is an access control status representing the desired permissions of the resulting store path.

If either of those arguments are passed, the Nix daemon (or other store implementation) should be notified that the corresponding store object is supposed to be protected (perhaps using a new worker protocol command), and after the build is completed, the ACLs should be set appropriately (only the users specified in the argument, and the building user, should have access to the path).

### Nix Daemon/local store implementation

The Nix daemon (or the local store implementation) should, if necessary, update, and then check whether the user has permission to access all dependencies before accepting a derivation realization request from any client.
After the build is performed, the user should be granted access to all the derivation outputs.
Also, paths dumped to store by users should automatically be accessible by the user, and the access list should be updated should any other user dump the same path in the future.

Whenever the user adds a path to the store (`Add*ToStore`, `ImportPaths`), set the ACL in the filesystem accordingly, like this (but in C++, of course):

```shell
setfacl -R -m user:$UID:rx /nix/store/...
```

Whenever a user tries to build a store path (`BuildPaths*`, `BuildDerivation`), we check if they have access to all dependencies or they are granted permission explicilty, then we build the path if necessary, and then recursively add an entry for them to the ACL of the store path.

There should be a couple of new operations in the worker protocol which allow getting and setting the list of users/groups with access to the path.
If the specified path does not exist yet, those commands should operate on the "future" access status of the path, meaning the access status that will be applied as soon as the path gets added to the store.

Those operations should be emitted by Nix clients before the path is added to the store or a build is completed, and by `nix store access grant`/`nix store access revoke`.

For all other operations working on paths, we check if the user has access before doing anything.

When the access is revoked explicitly, we remove the user from the ACL:

```shell
setfacl -R -x user:$UID:rx /nix/store/...
```

Also, depending on the design of substituters, we might need to handle the new proof-of-source protocol, to prove that the necessary sources are present in the store.
As mentioned, this should only happen if the user requesting realization has the relevant access to those paths.

## Remote stores

For remote stores, this problem is a bit more difficult since sending all the inputs to the remote store every time a dependency has to be downloaded is really expensive.
There should be a protocol to establish proof-of-source, as in proof that the client has access to all the sources of a derivation, before that derivation output is provided.

Simply providing the hash of the inputs as the proof would be one possible (but problematic) implementation.
It is succeptible to replay attacks, furthermore typically obtaining the hash of the inputs is easier then obtaining the inputs themselves.

An improvement on simply providing the hash would be providing the hash of the concatenation of substituter URI and the NAR contents of the input closure.
This still has some potential for MITM attacks, but prevents replays. Using an HTTPS cache should prevent most types of MITM attacks.

Another solution (which can be implemented together with the previous one) is to keep an access list as metadata in the cache, and keep a mapping between local users with access to that list and simple credentials (e.g. simple HTTP auth).
This has the benefit of being the easiest one to implement, but is a hassle to use (requires granting the permissions and setting up credentials externally), and also is succeptible to credential leaks.

Finally, in a situation where local daemons have full access to the cache but restrict local user access, it is possible to leave the substituter logic as is, and offload all the checks to the local daemons.
It should be rather easy to implement, since all the same checks would have to be done for substitution as for simply reusing the locally built outputs.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

## Local example

Alice, Bob, Carol and Eve share a build-machine with a single nix store.

Alice builds some proprietary software from source.
She can do this as usual, by just running `nix build`.
She now can access all the relevant store paths, including the source (which has been copied to the store for the build), the build dependencies, the runtime dependencies, and the resulting derivation output.

She wants to collaborate on this software with Bob.
She shares the source with Bob, he executes `nix build` as well, which is really fast -- the only thing that happens is that he is granted access to all the same paths as Alice.
No actual building occurs.

Now, Alice and Bob want to share the resulting binaries (but not the sources) with Carol.
They can add a `__permissions.outputs.out.users = [ "alice" "bob" "carol" ];` and re-build the derivation (nothing will get re-built, only the permissions will be updated).
Alternatively, either of them can issue a `nix store access grant --recursive --user carol /nix/store/...-software` command, granting Carol access to software and all its runtime dependencies, but none of the sources.
Carol can inspect and run the binary version of the software, and even (theoretically) build other software on top of it.

Finally, an evil Eve wants to steal the software.

The only two ways for her to get access to the software would be either to obtain the sources via some other means, at which point she doesn't really need anything in the store anyways since it would be trivial to produce the binary artifacts on her personal machine, or trick the machine's administrators into explicitly granting her access.

## Remote example

Alice sets up a binary cache.

She uploads some proprietary software (both the source and the realised derivation output) to the store.

Alice wants to collaborate on this software with Bob.
She shares the source with him, and he can now add this substituter to his `substituters` and fetch both the sources and the binary products for local development.

Alice wants to share the resulting binary with Carol.
She grants explicit access to `carol` on the substituter, generates some credentials and adds them to the "password file" for the substituter.
Carol then can add the substituter with these credentials to her `substituters`, and fetch the artifact (but not the source) to run it locally.

Evil Eve still can't steal the software, since to download the path she once again either needs to obtain the source or the credentials to download it from the cache.

# Drawbacks
[drawbacks]: #drawbacks

- This change requires significant refactoring of many Nix components;
- Implementing ACLs will impose a performance penalty. Hopefully this penalty will not be too bad, and also since ACLs are entirely optional this shouldn't affect users who don't need it;
- Short secrets in the store can be brute-forced, since the hash of the content is known, unless [RFC 97] is also implemented.
- Security achieved by ACLs on a multi-user system will depend on the Nix daemon implementation, which has quite a large attack surface;
- Syncing ACLs between machines can be difficult to do properly.

# Alternatives
[alternatives]: #alternatives

# Unresolved questions
[unresolved]: #unresolved-questions

- Does the nix store permission setup belong as a NixOS module in nixpkgs, or as part of the Nix installation, or both? More investigation is needed.
- How to handle the remote store case.

# Future work
[future]: #future-work

