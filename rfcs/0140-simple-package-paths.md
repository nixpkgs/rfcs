---
feature: simple-package-paths
start-date: 2022-09-02
author: Silvan Mosberger
co-authors: Robert Hensing
pre-RFC reviewers: Thomas Bereknyei, John Ericson, Alex Ameen
shepherd-team: (names, to be nominated and accepted by RFC steering committee)
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: https://github.com/NixOS/nixpkgs/pull/211832
---

# Summary
[summary]: #summary

Auto-generate trivial top-level attribute definitions in `pkgs/top-level/all-packages.nix` from a directory structure that matches the attribute name.
This makes it much easier to contribute new packages, since there's no more guessing needed as to where the package should go, both in the ad-hoc directory categories and in `pkgs/top-level/all-packages.nix`.

# Motivation
[motivation]: #motivation

- It is not obvious to package contributors where to add files or which ones to edit. These are very common questions:
  - Which directory should my package definition go in?
  - What are all the categories and do they matter?
  - What if the package has multiple matching categories?
  - Why can't I build my package after adding the package file?
  - Where in all-packages.nix should my package go?
- Figuring out where an attribute is defined is a bit tricky:
  - First one has to find the definition of it in all-packages.nix to see what file it refers to
    - On GitHub this is even more problematic, as the `all-packages.nix` file is [too big to be displayed by GitHub](https://github.com/NixOS/nixpkgs/blob/nixos-22.05/pkgs/top-level/all-packages.nix)
  - Then go to that file's definition, which takes quite some time for navigation (unless you have a plugin that can jump to it directly)
    - It also slows down or even locks up editors due to the file size
  - `nix edit -f . package-attr` works, though that's not yet stable (it relies on the `nix-command` feature being enabled) and doesn't work with packages that don't set `meta.position` correctly).
- `all-packages.nix` frequently causes merge conflicts. It's a point of contention for all new packages

# Detailed design
[design]: #detailed-design

This RFC consists of two parts, each of which is implemented with a PR to Nixpkgs.

## PR 1: The unit directory standard

This part establishes the new _unit directory standard_ in Nixpkgs.
This standard is internal to Nixpkgs and not exposed as public interface.
This standard must be documented in the Nixpkgs manual.

### File structure

Create the initially-empty directory `pkgs/unit`, called _unit base directory_, in Nixpkgs.

Check the following using CI:
- The unit base directory must only contain subdirectories of the form `pkgs/unit/${shard}/${name}`, called _unit directories_.
- `name` is a string only consisting of the ASCII characters a-z, A-Z, 0-9, `-` or `_`.
- `shard` is the lowercased first two letters of `name`, expressed in Nix: `shard = toLower (substring 0 2 name)`.
- Each unit directory must contain a `package.nix` file and may contain arbitrary other files.

### Semantics

Introduce code to automatically define `pkgs.${name}` for each unit directory as a value equivalent to
```nix
pkgs.callPackage pkgs/unit/${shard}/${name}/package.nix { }
```

Optionally there may also be an overriding definition of `pkgs.${name}` in `pkgs/top-level/all-packages.nix` equivalent to
```nix
pkgs.callPackage pkgs/unit/${shard}/${name}/package.nix args
```

with an arbitrary `args`.

Check the following using CI for each unit directory:
- `pkgs.${name}` is defined as above, either automatically or with some `args` in `pkgs/top-level/all-packages.nix`.
- `pkgs.${name}` is a [derivation](https://nixos.org/manual/nix/stable/glossary.html#gloss-derivation).
- <a id="req-ref"> The `package.nix` file must not transitively refer to files outside its unit directory.

## PR 2: Automated migration

When possible, automatically migrate to the unit directory standard for all _satisfiying definitions_ `pkgs.${name}`, meaning derivations defined as above using `callPackage`.

Automatic migration is only possible if:
- Files only need to be moved, not changed, with the exception of `pkgs/top-level/all-packages.nix`
- The Nixpkgs package evaluation result does not change

All satisfying definitions that can't be automatically migrated due to the above restrictions will be added to a CI exclusion list.
CI is added to ensure that all satisfying definitions except the CI exclusion list must be using the unit directory standard.
This means that the unit directory standard becomes mandatory for new satisfying definitions after this PR.

This PR will cause merge conflicts with all existing PRs that modify moved files, however they can trivially be rebased using `git rebase && git push -f`.
However, to have the least amount of conflicts, this migration should be performed soon after a release when ZHF is over and the PR rate slows down.
This also gives a lot of time to fix any potential problems before the next release.

Manual updates may also be done to ensure further correctness, such as
- [CODEOWNERS](https://github.com/NixOS/nixpkgs/blob/master/.github/CODEOWNERS)
- Update scripts like [this](https://github.com/NixOS/nixpkgs/blob/cb2d5a2fa9f2fa6dd2a619fc3be3e2de21a6a2f4/pkgs/applications/version-management/cz-cli/generate-dependencies.sh)

## Examples
[examples]: #examples

To add a new package `pkgs.foobar` to Nixpkgs, one only needs to create the file `pkgs/unit/fo/foobar/package.nix`.
No need to find an appropriate category nor to modify `pkgs/top-level/all-packages.nix` anymore.

With some packages, the `pkgs/unit` directory may look like this:

```
pkgs
└── unit
   ├── _0
   │  ├── _0verkill
   │  └── _0x
   ┊
   ├── ch
   │  ├── ChowPhaser
   │  ├── CHOWTapeModel
   │  ├── chroma
   │  ┊
   ┊
   ├── t
   │  └── t
   ┊
```

# Interactions
[interactions]: #interactions

## Shard distribution

The shard structure nesting unit directories under their lowercased two-letter prefix [leads to a distribution](https://gist.github.com/infinisil/95c7013db62e9f23ab2bc92165a37221) into shards as follows:
- There's 17305 total non-alias top-level attribute names in Nixpkgs revision [6948ef4deff7](https://github.com/nixos/nixpkgs/commit/6948ef4deff7a72ebe5242244bd3730e8542b925)
- These are split into 726 shards
- The top three shards are:
  - "li": 1092 values, coming from the common `lib` prefix
  - "op": 260 values
  - "co": 252 values
- There's only a single directory with over 1 000 entries, which is notably GitHub's display limit, so this means only 92 attributes would be harder to see on GitHub

These stats are also similar for other package sets for if this standard were to be adopted for them in the future.

## Migration size
Due to the limitations of the standard, only a limited set of top-level attributes can be automatically migrated:
- No attributes that aren't derivations like `pkgs.fetchFromGitHub` or `pkgs.python3Packages`
- No attributes defined using non-`pkgs.callPackage` functions like `pkgs.python3Packages.callPackage` or `pkgs.haskellPackages.callPackage`.
In the future we might consider having a separate namespace for such definitions.

Concretely this [can be computed](https://gist.github.com/infinisil/4f2bd165c2603fc28ab536f39ac2fd27) to be 81.2% (14036) attributes out of the 17280 total non-alias top-level Nixpkgs attributes in revision [6948ef4deff7](https://github.com/nixos/nixpkgs/commit/6948ef4deff7a72ebe5242244bd3730e8542b925).

And the initial automatic migration will be a bit more limited due to the additional constraints:
- No attributes that share common files with other attributes like [`pkgs.readline`](https://github.com/NixOS/nixpkgs/tree/cb2d5a2fa9f2fa6dd2a619fc3be3e2de21a6a2f4/pkgs/development/libraries/readline)
- No attributes that references files from other packages like [`pkgs.gettext`](https://github.com/NixOS/nixpkgs/blob/cb2d5a2fa9f2fa6dd2a619fc3be3e2de21a6a2f4/pkgs/development/libraries/gettext/default.nix#L60)
These attributes will need to be moved to the standard manually with some arguably-needed refactoring to improve reusability of common files.

## Package locations

`nix edit` and search.nixos.org will automatically point to the new location without problems, since they rely on `meta.position` to get the file to edit, which still works.

## Git and NixOS release problems

- The migration will cause merge conflicts for a lot of PRs, but they are trivially resolvable using `git rebase && git push -f` due to Git's file rename tracking.
- Commits that change moved files in `pkgs/unit` can be cherry-picked to the previous file location without problems for the same reason.
- `git blame` locally and on GitHub is unaffected, since it follows file moves properly.

## `callPackage` with `nix-build -E`

A commonly recommended way of building package directories in Nixpkgs is to use `nix-build -E 'with import <nixpkgs> {}; callPackage pkgs/applications/misc/hello {}'`.
Since the path changes `package.nix` is now used, this becomes like `nix-build -E 'with import <nixpkgs> {}; callPackage pkgs/unit/he/hello/package.nix {}'`, which is harder for users.
However, calling a path like this is an anti-pattern anyway, because it doesn't use the correct Nixpkgs version and it doesn't use the correct argument overrides.
The correct way of doing it was to add the package to `pkgs/top-level/all-packages.nix`, then calling `nix-build -A hello`.
This `nix-build -E` workaround is partially motivated by the difficulty of knowing the mapping from attributes to package paths, which is what this RFC improves upon.
By teaching users that `pkgs/unit/<shard>/<name>` corresponds to `nix-build -A <name>`, the need for such `nix-build -E` workarounds should disappear.

## Removing custom arguments

While this RFC allows passing custom arguments, doing so means that `pkgs/top-level/all-packages.nix` will have to be maintained for that package.
In specific cases where attributes of custom arguments are of the form `name = value` and `name` isn't a package attribute, they can be avoided without breaking the API.
To do so, ensure that the function in the called file has `value` as an argument and set the default of the `name` argument to `value`.

This notably doesn't work when `name` is already a package attribute, because then the default is never used and instead overridden.

# Drawbacks
[drawbacks]: #drawbacks

- This standard can only be used for top-level packages using `callPackage`, so not for e.g. `python3Packages.requests` or a package defined using `haskellPackages.callPackage`
- It's not possible anymore to be a [GitHub code owner](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) of category directories.
- The existing categorization of packages gets lost. Counter-arguments:
  - It was never that useful to begin with.
    - The categorization was always incomplete, because packages defined in the language package sets often don't get their own categorized file path.
    - It was an inconvenient user interface, requiring a checkout or browsing through GitHub
    - Many packages fit multiple categories, leading to multiple locations to search through instead of one
  - There's other better ways of discovering similar packages, e.g. [Repology](https://repology.org/)
- This breaks `builtins.unsafeGetAttrPos "hello" pkgs`. Counter-arguments:
  - We have to draw a line as to what constitutes the public interface of Nixpkgs. We have decided that making attribute position information part of that is not productive. For context, this information is already accepted to be unreliable at the language level, noting the `unsafe` part of the name.
  - Support for this could be added to Nix (make `builtins.readDir` propagate file as a position)

# Alternatives
[alternatives]: #alternatives

## A different unit base directory

Context: `pkgs/unit` is the unit base directory

Alternatives:
- Don't nest the directory under `pkgs`
  - This is future proof in case we want to make the directory structure more general purpose, but this is out of scope
- Another name than `unit`, proposed were `auto`, `pkg`, `mod`, `component`, `part`, `comp`, `app`, `simple`, 

## Alternate `pkgs/unit` structure

Context: The structure is `pkgs/unit/${shard}/${name}` with `shard` being the lowercased two-letter prefix of `name`.


Alternatives:
- A flat directory, where `pkgs.hello` would be in `pkgs/unit/hello`.
  - (+) Simpler for the user and code.
  - (-) The GitHub web interface only renders the first 1 000 entries when browsing directories, which would make most packages inaccessible in this way.
    - (+) This feature is not used often.
      - (-) [A poll](https://discourse.nixos.org/t/poll-how-often-do-you-navigate-through-directories-on-github-to-get-to-a-nixpkgs-package-file/21641) showed that about 41% of people rely on this feature every week.
  - (-) Bad because it makes `git` and file listings slower.
- Use three-letter or hour-letter prefixes.
  - (-) Also leads to directories containing more than 1 000 entries, see above.
- Use multi-level structure, e.g. a two-level two-letter prefix structure where `hello` is in `pkgs/unit/he/ll/hello`
  - (+) This would virtually a unlimited number of packages without performance problems
  - (-) It's hard to understand, type and implement, needs a special case for packages with few characters
    - E.g. `x` could go in `pkgs/unit/x-/--/x`
  - (-) There's not enough packages even in Nixpkgs that a two-level structure would make sense. Most of the structure would only be filled by a couple entries.
  - (-) Even Git only uses 2-letter prefixes for its objects hex hashes
- Use a dynamic structure where directories are rebalanced when they have too many entries.
  E.g. `pkgs.foobar` could be in `pkgs/unit/f/foobar` initially.
  But when there's more than 1 000 packages starting with `f`, all packages starting with `f` are distributed under 2-letter prefixes, moving `foobar` to `pkgs/unit/fo/foobar`.
  - (-) The structure depends not only on the name of the package then, making it harder to find packages again and figure out where they should go
  - (-) Complex to implement

## Alternate `package.nix` filename

- `default.nix`
  - (+) `default.nix` is already a convention most people are used to.
  - (-) We don't benefit from the usual `default.nix` benefits:
    - Removing the need to specify the file name in expressions, but this does not apply because this file will be imported automatically by the code that replaces definitions from `all-packages.nix`.
      - (+) But there's still some support for `all-packages.nix` for custom arguments, which requires people to type out the name
        - (-) This is hopefully only temporary, in the future we should fully get rid of `all-packages.nix`
    - Removing the need to specify the file name on the command line, but this does not apply because a package function must be imported into an expression before it can be used, making `nix build -f pkgs/unit/hell/hello` equally broken regardless of file name.
  - (-) Not using `default.nix` frees up `default.nix` for an expression that is actually buildable, e.g. `(import ../.. {}).hello`, although we don't yet have a use case for this that isn't covered by `nix-build ../.. -A <attrname>`.
  - (-) Using `default.nix` would tempt users to invoke `nix-build .`, which wouldn't work and making package functions auto-callable is a known anti-pattern.
- `pkg-fun[c].nix`
  - (+) Makes a potentially transition to a non-function form of packages in the future easier.
    - (-) There's no problem with introducing versioning later with different filenames.
    - (-) We don't even know if we actually want to have a non-function form of packages.
  - (-) Abbreviations are a bit jarring.

## Filepath backwards-compatibility

Context: The migration moves files around without providing any backwards compatibility for those moved paths.

Alternative:
- Have a backwards-compatibility layer for moved paths, such as a symlink pointing from the old to the new location, or for Nix files even a `builtins.trace "deprecated" (import ../new/path)`.
  - (-) It would give precedent to file paths being a stable API interface, which definitely shouldn't be the case (bar some exceptions).
  - (-) Leads to worse merge conflicts as the transition is happening, since Git would have to resolve a merge conflict between a symlink and a changed file.

## Reference check

Context: There's a [requirement](#user-content-req-ref) to check that unit directories can't access paths outside themselves.

Alternatives:
- Don't have this requirement
  - (-) Doesn't discourage the use of file paths as an API.
  - (-) Makes further migrations to different file structures harder.
- Make the requirement also apply the other way around: Files outside the unit directory cannot access files inside it, with `package.nix` being the only exception
  - (-) Enforcing this requires a global view of Nixpkgs, which is hard to implement

## Restrict design to try delay issues like "package variants" {#no-variants}

TODO: This section might be outdated, it's a bit ambiguous

- <a id="def-variant-attribute"/> *variant attribute*: an attribute that defines a package by invoking it with non-default arguments, for example:
 ```
   graphviz_nox = callPackage ../tools/graphics/graphviz { withXorg = false; };
 ```

We perceived some uncertainty around [package variants](#def-package-variant) that led us to scope these out at first, but we did not identify a real problem that would arise from allowing non-auto-called attributes to reference `pkgs/unit` files. However, imposing unnecessary restrictions would be counterproductive because:

 - The contributor experience would suffer, because it won't be obvious to everyone whether their package is allowed to go into `pkgs/unit`. This means that we'd fail to solve the goal "Which directory should my package definition go in?", leading to unnecessary requests for changes in pull requests.

 - Changes in dependencies can require dependents to add an override, causing packages to be moved back and forth between unit directories and the general `pkgs` tree, worsening the problem as people have to decide categories *again*.

 - When lifting the restriction, the reviewers have to adapt, again leading to unnecessary requests for changes in pull requests.
 
 - We'd be protracting the migration by unnecessary gatekeeping or discovering some problem late.

That said, we did identify risks:

 - We might get something wrong, and while we plan to incrementally migrate Nixpkgs to this new system anyway, starting with fewer units is good.
    - Mitigation: only automate the renames of simple (`callPackage path { }`) calls, to keep the initial change small
 
 - We might not focus enough on the foundation, while we could more easily relax requirements later.
    - After more discussion, we feel confident that the manual `callPackage` calls are unlikely to cause issues that we wouldn't otherwise have.

## Allow `callPackage` arguments to be specified in `<unit>/args.nix`

Context: Custom `callPackage` arguments have to be added to `all-packages.nix`

Alternative:
Expand the auto-calling logic according to:
Unit directories are automatically discovered and transformed to a definition of the form
```
# If args.nix doesn't exist
pkgs.${name} = pkgs.callPackage ${unitDir}/package.nix {}
# If args.nix does exists
pkgs.${name} = pkgs.callPackage ${unitDir}/package.nix (import ${unitDir}/args.nix pkgs);
```

- (+) It makes another class of packages uniform, by picking a solution with restricted expressive power.
- (-) It does not solve the contributor experience problem of having too many rules.
      `args.nix` is another pattern that contributors need to learn how to use, as we have seen that it is not immediately obvious to everyone how it works.
  - (+) A CI check can mitigate the possible lack of uniformity, and we see a simple implementation strategy for it.
- (-) Complicates the unit directory structure with an optional file

# Unresolved questions
[unresolved]: #unresolved-questions

# Future work
[future]: #future-work

All of these questions are in scope to be addressed in future discussions in the [Nixpkgs Architecture Team](https://nixos.org/community/teams/nixpkgs-architecture.html):

- Expose an API to get access to the package functions directly, without calling them
- Add a meta tagging or categorization system to packages as a replacement for the package categories. Maybe `meta.tags` with `search.nixos.org` integration. Maybe https://repology.org/ integration. See also https://github.com/NixOS/rfcs/pull/146.
- Making the filetree more human-friendly by grouping files together by "topic" rather than technical delineations.
  For instance, having a package definition, changelog, package-specific config generator and perhaps even NixOS module in one directory makes work on the package in a broad sense easier.
- This RFC only addresses the top-level attribute namespace, aka packages in `pkgs.<name>`, it doesn't do anything about package sets like `pkgs.python3Packages.<name>`, `pkgs.haskell.packages.ghc942.<name>`, which may or may not also benefit from a similar auto-calling
- Improve the semantics of `callPackage` and/or apply a better solution, such as a module-like solution
- Potentially establish an updateScript standard to avoid problems like, relates to Flakes too
- What to do with different versions, e.g. `wlroots = wlroots_0_14`? This goes into version resolution, a different problem to fix
- What to do about e.g. `libsForQt5.callPackage`? This goes into overrides, a different problem to fix
- What about aliases like `jami-daemon = jami.jami-daemon`?
- What about `recurseIntoAttrs`? Not single packages, package sets, another problem