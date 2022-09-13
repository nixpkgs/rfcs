---
feature: issues-warnings
start-date: 2022-06-11
author: piegames
co-authors: —
shepherd-team: @lheckemann, @mweinelt, @fgaz
shepherd-leader: @mweinelt
related-issues: https://github.com/NixOS/nixpkgs/pull/177272
---

# RFC: Nixpkgs issues and warnings

## Summary
[summary]: #summary

Inspired by the derivation checks for broken and insecure packages, a new system called "warnings" is introduced. It is planned to eventually replace the previously mentioned systems, as well as the current "warnings" (which currently only prints a trace message for unmaintained packages). `meta.issues` is added to derivations, which can be used to let the evaluation of individual packages fail with a custom message. This will then be used to warn users about packages that are in need of maintenance. Packages that have an open issue for a long time should eventually be removed, although doing so is not part of the RFC.

## Motivation
[motivation]: #motivation

Nixpkgs has the problem that it is often treated as "append-only", i.e. packages only get added but not removed. There are a lot of packages that are broken for a long time, have end-of-life dependencies with known security vulnerabilities or that are otherwise unmaintained.

Let's take the end of life of Python 2 as an example. (This applies to other ecosystems as well, and will come up again and again in the future.) It has sparked a few bulk package removal actions by dedicated persons, but those are pretty work intensive and prone to burn out maintainers. A goal of this RFC is to provide a way to notify all users of a package about the outstanding issues. This will hopefully draw more attention to abandoned packages, and spread the work load. It can also help soften the removal of packages by providing a period for users to migrate away at their own pace.

Apart from that, there is need for a general per-package warning mechanism in nixpkgs – one that is stronger than `builtins.trace`.

## Detailed design
[design]: #detailed-design

### Package issues

A new attribute is added to the `meta` section of a package: `issues`. If present, it is a list of attrsets which each have at least the following fields:

- `message`: Required. A string message describing the issue with the package.
- `kind`: Optional but recommended. If present, the resulting warning will be printed as `kind: message`.
- `date`: Required. An ISO 8601 `yyyy-mm-dd`-formatted date from when the issue was added.
- `urls`: Optional, list of strings. Can be used to link issues, pull requests and other related items.

Other attributes are allowed. Some message kinds may specify additional required attributes.

Example values:

```nix
meta.issues = [{
  kind = "deprecated";
  message = "This package depends on Python 2, which has reached end of life.";
  date = "1970-01-01";
  urls = [ "https://github.com/NixOS/nixpkgs/issues/148779" ];
} {
  kind = "removal";
  message = "The application has been abandoned upstream, use libfoo instead";
  date = "1970-01-01";
}];
```

### nixpkgs integration

The following new config options are added to nixpkgs: `ignoreWarningsPredicate`, `ignoreWarningsPackages`, `traceIgnoredWarnings`. The option `showDerivationWarnings` will be removed. A new environment variable is defined, `NIXPKGS_IGNORE_WARNINGS`.

The semantic and implementation of `ignoreWarningsPredicate` and `ignoreWarningsPackages` directly parallels the existing "insecure" package handling.

Similarly to broken, insecure and unfree packages, evaluating a package with an issue fails evaluation. Ignoring a package without issues (i.e. they have all been resolved) results in a warning at evaluation time.

The previous warnings system in `check-meta.nix` is removed, together with `showDerivationWarnings`. Instead, ignored warnings are printed with `builtins.trace` depending on the new option `traceIgnoredWarnings`.

### Warning kinds

At the moment, the following values for the `kind` field of a warning are known:

- `removal`: The package is scheduled for removal some time in the future.
- `deprecated`: The package has been abandoned upstream or has end of life dependencies.
- `maintainerless`: `meta.maintainers` is empty
- `insecure`: The package has some security vulnerabilities

Not all values make sense as issues (i.e. within `meta.issues`): Some warnings may be automatically generated from other `meta` attributes (for example `maintainerless`). New kinds may be added in the future.

## Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

### Package removal

There are two ways issues interact with the removal of packages: Either they get an issue because they are going to be removed, or they are removed because they have an open issue for a prolonged period of time.

- Instead of removing a package directly, it should first get an issue announcing the planned removal. This will allow users to migrate away beforehand. `removal` must be used as `kind` (This will facilitate automation in the future).
- Before branch-off for a new release, all (leaf) packages with issues that predate the previous branch-off are deemed safe for removal (unless stated otherwise). If a package is removed based on its issue, the issue's message becomes part of the new `throw` alias.

### Propagation across transitive dependencies

When a package has an issue, all packages that depend on it will fail to evaluate until that package is ignored or the issue resolved. Sometimes, this is sufficient.

When the issue requires actions on dependents however, it does not sufficiently inform about all packages that need action. Marking all dependents with that issue is not a good idea either though: it would require users to go through some potentially long dependency chains. Instead, only applications, leaf packages or packages with very few dependents should get the issue.

As an example, take `gksu` with the `gksu` → `libgksu` → `libglade` → `python2` dependency chain (for the sake of the example, ignore that it also depends on EOL Gtk 2). Obviously, `python2` should get an issue. As a leaf/application, `gksu` should get one too (it could be the same, or with an adpated message). For the packages in between, it depends on whether they require individual action or not.

### Backporting

New issues generally should not be added to stable branches if possible, and also not be backported to them, since this breaks evaluation. The same rule applies to other changes to a pacakge's `meta` which may generate a warning and thus lead to evaluation failure too. A notable exception are warnings of kind `insecure`, if there is no fix that can be backported.

## Drawbacks
[drawbacks]: #drawbacks

- People have voiced strong negative opinions about the prospect of removing packages from nixpkgs at all, especially when they still *technically* work.
  - We do not want to encourage the use of unmaintained software likely to contain security vulnerabilities, and we do not have the bandwidth to maintain packages deprecated by upstream. Nothing is lost though, because we have complete binary cache coverage of old nixpkgs versions, providing a comparatively easy way to pin old very package versions.
- There is a slight long-term maintenance burden. It is expected to be similar to or slightly greater than the maintenance of our deprecation aliases.
  - We expect that in the long term, having a defined process for removing unmaintained and obsolete packages, especially compared to deciding on a case-by-case basis, is likely to reduce the overall maintenance burden.
- Some of the example interactions are built on the premise that parts of nixpkgs are under-maintained, and that most users are at least somewhat involved in the nixpkgs development process. At the time of writing this RFC this is most certainly true, but the effects on this in the future are unknown.
  - We hope that drawing attention to packages in need of maintenance can encourage new maintainers -- both from the existing pool of nixpkgs contributors and from non-contributor users -- to step up.

## Alternatives
[alternatives]: #alternatives

An alternative design would be to have issues as a separate list (not part of the package). Instead of allowing individual packages, one could ignore individual warnings (they'd need an identifying number for that). The advantage of doing this is that one could have one issue and apply it for a lot of packages (e.g. "Python 2 is deprecated"). The main drawback there is that it is more complex.

A few other sketches about how the declaration syntax might look like in different scenarios:

```nix
{
  # As proposed in the RFC
  meta.issues = [{
    message = "deprecation: Python 2 is EOL. #12345";
    # (Other fields omitted for brevity)
  }];

  # Issues are defined elsewhere in some nixpkgs-global table, only get referenced in packages
  meta.issues = [ "1234-python-deprecation" ];

  # Attempt to unify both approaches to allow both ad-hoc and cross-package declaration
  meta.issues = {
    "1234-python-deprecation" = {
      message = "deprecation: Python 2 is deprecated #12345";
    };
  };

  # Proposal by @matthiasbeyer
  meta.issues = [
    { transitive = pkgs.python2.issues }
  ];
}
```

## Unresolved questions
[unresolved]: #unresolved-questions

- From above: "Ignoring a package without issues (i.e. they have all been resolved) results in a warning at evaluation time". How could this be implemented, and efficiently?
- ~~Should issues be a list or an attrset?~~
  - We are using a list for now, there is always the possibility to also allow attrsets in the future.
- ~~A lot of bike shedding. (See below)~~
  - Packages may have "issues" that generate "warnings" that have to be "ignored". Issues are explicitly added to packages in `meta.issues`, whereas warnings can be generated automatically from other sources, like other `meta` attributes.

## Future work
[future]: #future-work

- Issues are designed in a way that they supersede a lot of our "insecure"/"unfree"/"unsupported" packages infrastructure. There is a lot of code duplication between them. In theory, we could migrate some of these to make use of issues. At the very least, we hope that issues are general enough so that no new similar features will have to be added in the future anymore.
  - `meta.knownVulnerabilities` is the first candidate to go
  - Unfree package handling will probably be out of scope, since we already have some custom filtering based on licences.
- Inspired by the automation of aliases, managing issues can be helped by tooling as well. This is deemed out of scope of this RFC because only real world usage will tell which actions will be worthwhile automating, but it should definitely considered in the future.
  - There will likely be need for tooling that lists issues on all nixpkgs packages, filtered by kind or sorted chronologically.
  - Automatically removing packages based on time will likely require providing more information whether it is safe to do so or not.
- > If the advisories were a list, and we also added them for modules, maybe we could auto-generate most release notes, and move release notes closer to the code they change. [[source]](https://discourse.nixos.org/t/pre-rfc-package-advisories/19509/4)
  - Issues can certainly be automatically integrated into the release notes somehow. However, this alone would not allow us to move most of our release notes into the packages, because for many release entries breaking eval would be overkill.

## Bike shedding

Here are a few naming proposals, and how well they would be suited to describe different conditions. "broken" and "unsupported" must be acknowledged but – unlike the others – don't imply some required user action. "unfree" is somewhat out of scope because it is unlikely to be replaced anytime soon.

| Kind        | Currently | "Issue" | "Warning" | "Problem" | "Advisory" |
|-------------|-----------|---------|-----------|-----------|------------|
|insecure     | `meta.knownVulnerabilities` |✅|✅|✅|✅|
|unmaintained | `meta.maintainers = []` |✅|✅|❓|❓|
|deprecated   | n/a |✅|✅|✅|❓|
|removal      | n/a |❓|✅|✅|❓|
|||||||
|broken       | `meta.broken`  |✅|❌|✅|❌|
|unsupported  | `meta.platforms` |✅|❓|✅|❌|
|||||||
|unfree       | `meta.license` |✅|✅|❌|❓|


"Advisory" was initially chosen based on the notion of security advisories, but was later dismissed as the project grew in scope. "Issue" and "Problem" are similar words, of which the former is a well-known technical term¹ which should be preferred here.

"Issue" and "Warning" are both good candidates, of which the former implies some required action whereas the latter merely wants to inform. In the end, we decided that packages should have *issues* which should produce *warnings* that can be *ignored*. While this distinction may be a bit unintuitive, it will make it easier to generate warnings from things that are not explicitly marked as issues (e.g. missing maintainers).

¹ Non-native speakers: look up the difference between "issue" and "problem" in English :)