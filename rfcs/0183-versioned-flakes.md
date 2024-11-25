---
feature: versioned-flakes
start-date: 2024-11-25
author: domenkozar
co-authors: None
shepherd-team: None
shepherd-leader: None
related-issues: None
---
## Summary

Introduce a standardized versioning schema for Nix flakes using [SemVer](https://semver.org/),
following lessons learned from [RFC 144](https://github.com/NixOS/rfcs/pull/144/files).

Adds to flakes:

- `version` attribute to `inputs` in the top-level of `flake.nix` to specify version of the input to be resolved.
- An attribute `version` in the top-level of `flake.nix`.

## Motivation

Currently, Flakes lack a formal versioning mechanism, leading to potential duplication of dependencies and inefficient evaluations.

By adopting a versioning schema similar to Rust's Cargo system, we can utilize a SAT solver to manage dependencies more effectively, minimizing redundancy and improving performance.

## Explanation

- **Version Schema:** Implement a versioning system for flakes that follows SemVer conventions (e.g., `1.0.0`, `2.1.3`).
- **Input Versioning:** Allow flake inputs to specify version constraints (e.g., `inputs.nixpkgs.version = "^3.0.0";`) following [Rust Caret Requirement Syntax]([https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#version-requirement-syntax](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#caret-requirements)).
- **Dependency Resolution:** Integrate a SAT solver to resolve dependencies based on specified versions, ensuring compatibility and reducing duplication.

### Examples

**Defining a Flake with a Version:**

```nix
{
  version = "1.0.0";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs";
    nixpkgs.version = "^1.0.55";
  };

  outputs = { self, nixpkgs }: { /* ... */ };
}
```

### Implementation

Nix would resolve versions as part of the evaluation, shipping a SAT version resolver.

`flake-compat` wouldn't support version resolving.

[flakestry.dev](https://github.com/flakestry/flakestry.dev) would provide an HTTP API for resolving by preevaluating flakes.
