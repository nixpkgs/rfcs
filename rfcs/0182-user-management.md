---
feature: I'm Gonna Build My Own Home Tool With Blackjack and Modules!
start-date: 2024-09-20
author: Anderson Torres
co-authors: `@nyabinary`
shepherd-team: (names, to be nominated and accepted by RFC steering committee)
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

Enhance Nixpkgs monorepo with a declarative management system for basic users.

# Terminology
[terminology]: #terminology

Henceforth,

- The typical unprivileged user of a system will be called _basic user_;
- The typical privileged user of a system will be called _superuser_;

# Motivation
[motivation]: #motivation

[Nixpkgs](https://github.com/NixOS/nixpkgs) is by a far margin the largest
packageset in this existence, according to
[Repology](https://repology.org/repository/nix_unstable).

In principle, currently the Nix interpreter and evaluator already leverage
Nixpkgs for basic users. However, this raw usage is not very ergonomical,
especially when compared to the structured model provided by NixOS.

In this panorama, the average user has two extreme options here:

- system-wide configuration via NixOS, requiring superuser privileges;
- ad-hoc management via raw Nix commands, especially the not-recommended
  `nix-env`.

This RFC proposes to fill this gap: a structured model of system management for
basic users that does not require superuser privileges to be deployed.

Let's call it `hometool`.

# Detailed Design
[design]: #detailed-design

`hometool` has two components:

1. A driver program, hereinafter called `hometool`.

   Among other related tasks, this tool has the role of realizing the
   description of a home environment.

   As expected from a driver, this tool will rely on other programs to execute
   its intents; the most immediate one being the Nix evaluator.

2. A set of carefully crafted modules, leveraged by Nix Module System.

   The moduleset should reside in `hometool/` directory at the root of Nixpkgs
   monorepo, similar to how it happens to `nixos/` nowadays.

# Expectations
[expectations]: #expectations

The set of components above should provide the following properties:

- Declarativeness

  Users can specify the configuration of their systems in Nix language, by a set
  of detailed descriptions.

- Immutability

  As a consequence of declarativeness, the resulting descriptions are immutable.
  It allows comparing them by knowing precisely what changed between
  configurations.

- Customizability

  Users can derive specialized module configurations from current module set.

- Extensibility

  Users can extend the module set by writing their own.

- Scalability

  `hometool` should be scalable from the simplest to the more complex user
  environment definitions.

- Documentation

  Both the `hometool` and the moduleset should be well documented.

- Human Friendliness

  This `hometool` should be approachable with clear and understandable logging
  messages, plus the aforementioned documentation.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

Here is a small example of what is expected

```console
$> ### help message, possibly pointing to further detailed and/or specialized help, like `--all`, `--modules`
$> hometool help
$> ### generate a sample config
$> hometool sample-config > user-configuration.nix
$> ### because we like to tweak things a bit
$> emacs user-configuration.nix
$> ### build it first without deploying it yet
$> hometool build
$> ### a VM to test it before deploying - for us paranoids!
$> hometool build-vm
$> ### now install it!
$> hometool switch
$> ### list the generations
$> hometool generations list
$> ### list differences between generations
$> hometool generations diff 9 10
$> ### select a specific generation
$> hometool generations switch 10
$> ### remove older generations
$> hometool generations remove 1-8
```

# Drawbacks
[drawbacks]: #drawbacks

- Why not to keep this tool outside Nixpkgs monorepo?

  It can be argued that overlays and other facilities provided by Nix language
  allow to keep entire projects outside the Nixpkgs monorepo.

  However, any of those projects suffer of an unpleasant phenomenon: the second
  class citizen repo.

  Basically, a repository outside the monorepo will not receive the same level
  of care and attention of a project inside the monorepo. It happens because,
  among other things,

  - it is harder to pay attention in multiple repositories at the same time slot
    than only one;
  - it is harder to deal with pieces of code contained in distinct repositories;
  - it is harder to synchronize multiple repositories;

  Because of those hurdles, it becomes easier to ignore the repos outside the
  main monorepo.

  As a consequence, breaking changes in the monorepo are prone to be decided
  without taking the satellite repos into consideration. This situation leads
  the satellite repos to deal with the extra work of synchronizing with the
  recent changes.

  By living inside the main monorepo, the problems exposed above diminishes
  drastically.

  Further, having this home management toolset inside the monorepo brings many
  advantages:

  - No longer dealing with synchronization among projects
  - Better strategies for code deduplication, sharing and refactoring
  - Direct access to the most recent enhancements available

    Indeed this is precisely what happens in NixOS already: when a program is
    added, a NixOS module and test suite can be plugged in the same pull
    request, with few to no bureaucracy.

  - Reputation

    A home management toolset kept in the monorepo has the social add-on of
    being better regarded by the Nixpkgs community as a whole, since there will
    be no barriers for contribution from this same community.

- The monorepo will increase in size.

  The increase in size is expected to be small.

  Since we are building a set of modules to deal with the Nixpkgs packageset, a
  good heuristic approximation of the worst case would be the current size of
  `nixos` directory - after all, we want NixOS without superuser privileges.

  A quick `du` returns something like this.

  ```shell
  $> du -sh nixos/ pkgs/
  024572 nixos/
  379536 pkgs/
  ```

  By the numbers above, obtained locally at 2024-09-13, `nixos` occupies less
  than 7% of the sum of both `nixos + pkgs`. A quick and dirty calculation would
  bring a similar 7% increase in size.

  This is certainly a crude calculation, however we are ignoring many factors
  that will bring these numbers down. Namely:

  - Refactorings

    Similar code can be factored and abstracted.

  - Code sharing

    Code that was initially projected for basic user management can be found
    useful for superuser management too; and vice-versa.

- Code duplications.

  Code duplications can be refactored as they appear.

  Indeed it is way easier to deal with code duplication in a monorepo: since the
  barrier of communication between multiple repositories disappears, a
  duplicated code can be factored more easily, requiring just one pull request
  instead of one per repo.

- Evaluation times of monorepo will increase.

  The increase in evaluation time is a bit harder to measure. However,

  - "increase in evaluation time" was not an argument strong enough for undoing
    the NixOS assimilation;
  - arguably, the increase in evaluation time will be felt by those effectively
    using the hometool; the remaining users will not be signficatively affected.
  - a similar argument for the increase in size can be crafted here.

  The perceived advantages of having this module system surpasses these small
  disadvantages about size and evaluation time.

  Further, outside the scope of Nixpkgs, there are initiatives focused on
  optimizing the Nix evaluators.

# Alternatives
[alternatives]: #alternatives

## The trivial "do nothing"

Trivially keeping the status quo.

## Promote `guix-home`

What? Lisp is cool!

## Promote `home-manager`

Now a serious alternative worthy of consideration.

Home Manager has the most obvious and the most powerful advantage:

It exists and is working well. Further, it is battle-tested and encodes a lot of
knowledge in its 8 years lifespan and more than 3.7k commits.

However, it has some non-negligible disadvantages:

- The current code does not follow the same standards of the current Nixpkgs
  monorepo

  - many modules rely on stringly `extraConfig` instead of structured
    `settings`, contra RFC 0042;

- Many instances of technical debt, something expected in such a long-lived
    project:

    - excessive uses of `with lib;`

    - hacky solutions like the dangerous `mkOutOfStoreSymlink`:
      https://github.com/nix-community/home-manager/issues/3032

    - under-documented workarounds

    - too much reliance on Bash and its nasty idiosyncrasies

On the other hand, starting from a cleaner slate has the opposite set of
advantages and disadvantages: the kickstart of a project is too wide and it is
easier to get lost. On the other hand, there is considerably more freedom to
prototype and test ideas, older and newer.

# Prior art
[prior-art]: #prior-art

## Guix Home

As an example of prior art, there is our Scheme-based cousin, Guix Software
Distribution. Since at least 2022 AD they bring a similar tool, conveniently
called Guix Home.

The nicest thing about this tool is its tight integration with Guix as a whole,
to the point of `home` being a mere subcommand of `guix`.

## Home Manager

Home Manager is a well-established toolset that leverages the Nixpkgs module
system to allow basic users to manage their user-specific environment in a
declarative way.

## Wrapper Manager

Wrapper-manager is a Nix library that allows the user to configure your favorite
applications without adding files into `~/.config`. This is done by creating
wrapper scripts that set the appropriate environment set - variables, flags,
configuration files etc. - to the wrapped program.

## nixos-rebuild

In the Nix world, nixos-rebuild is the master tool in terms of deployment of a
Nix-based configuration, albeit being system-wide.

# Unresolved questions
[unresolved]: #unresolved-questions

Given that this tool will live inside Nixpkgs monorepo, it is expected that
future packages will interact with this new tool. How those interactions should
be dealt?

# Future work
[future]: #future-work

- Update and extend the CI
- Set expectations on portability
  - Especially outside NixOS
  - Especially outside Linux

# References
[references]: #references

- [Home Manager](https://nix-community.github.io/home-manager/)
- [Keeping One's Home
  Tidy](https://guix.gnu.org/en/blog/2022/keeping-ones-home-tidy/), by Ludovic
  Courtès
- [Guix Home
  Configuration](https://guix.gnu.org/manual/devel/en/html_node/Home-Configuration.html)
- [Wrapper Manager](https://github.com/viperML/wrapper-manager)
- [Stop Using nix-env](https://stop-using-nix-env.privatevoid.net/)
- [Overlay for User Packages](https://gist.github.com/LnL7/570349866bb69467d0caf5cb175faa74) by LnL7
- [BuilEnv-based Declarative User
  Environment](https://gist.github.com/lheckemann/402e61e8e53f136f239ecd8c17ab1deb)
  by lheckellman
- [Nix Home](https://github.com/museoa/nix-home/)
  - Forked and archived from [Nix Home](https://github.com/sheenobu/nix-home/)
  - with patches saved from other forks
- [nix-config from akavel](https://github.com/akavel/nix-config/tree/master/.nixpkgs)
  - https://github.com/sheenobu/nix-home/issues/16
  - https://github.com/akavel/nix-config/blob/510f36861cc4a641bd976ad25dd339949b47339d/.nixpkgs/nix-home.nix
  - https://github.com/akavel/nix-config/blob/510f36861cc4a641bd976ad25dd339949b47339d/.nixpkgs/nix-home.sh
  - https://github.com/akavel/nix-config/blob/510f36861cc4a641bd976ad25dd339949b47339d/.nixpkgs/config.nix#L57
- [GNU Stow](https://www.gnu.org/software/stow/)
- [Dropping Home Manager](https://ayats.org/blog/no-home-manager), an article from ViperML
  containing some criticism about home-manager
