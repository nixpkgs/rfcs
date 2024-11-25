---
feature: nix-language-version
start-date: 2022-12-12
author: @fricklerhandwerk @yorickvp
co-authors: @thufschmitt @Ericson2314 @infinisil
shepherd-team: @piegamesde @sternenseemann @gabriel-doriath-dohler
shepherd-leader: @sternenseemann
related-issues: https://github.com/NixOS/nix/issues/7255
---

# RFC 137 – Nix language versioning

# Summary
[summary]: #summary

- Introduce a convention to determine which version of the Nix language grammar to use for parsing and evaluating Nix expressions.
- Add parameters to the Nix language evaluator, controlling the behavior of deprecation warnings and errors.
- Codify a versioning policy for the Nix language specification

# Motivation
[motivation]: #motivation

The stability of Nix language has been praised on multiple occasions, e.g. [Nix and legacy enterprise software development: an unlikely match made in heaven](https://talks.nixcon.org/nixcon-2022/talk/QQPBFW/).
Yet, as with any software system, in order to accommodate new insights, we want to allow the Nix language to evolve.
This sometimes involves backward-incompatible ("breaking") changes that currently cannot be made without significant downstream disruption.

Therefore we propose mechanisms and policies for introducing changes to the Nix language in a controlled and deliberate manner.
It aims to avoid breaking existing code, to prevent inadvertently breaking reproducibility, and to minimise maintenance burden for implementors and users.

## Motivating examples

Incompatible changes from the past:

- [A changelog of Nix (language) versions](https://code.tvl.fyi/about/tvix/docs/lang-version.md), as reflected in `builtins.langVersion`
- There have been other, sometimes breaking changes to the language that have not resulted in an increment of the language version (e.g. the recent `fetchGit` changes).
- The `builtins.toJSON 1.000001` output [changed in Nix 2.12](https://github.com/NixOS/nix/issues/8259).

Possible future changes that are in discussion:

- Remove URL literals (currently implemented via experimental-features)
- Remove the old `let { body = ... }` syntax
- Disallow leading zeroes in integer literals (such as `umask = 0022`)
- Disallow `a.x or y` if `a` is not an attribute set
- Simplifying semantics of `builtins.toString` and string interpolation
- `__functor` and `__toString`, probably
- Remove `__overrides`
- [Make `builtins` more consistent](https://github.com/NixOS/nix/issues/7290), e.g. not exposing `map`, `removeAttrs`, `fetchGit` and and others in the global scope
- Fix [imprecision in string representation of floating point numbers](https://github.com/NixOS/nix/pull/6238)
- Make the `@`-pattern consistent with destructuring
- A syntax to index into lists, e.g. `[ 1 2 3 ].0 == 1`
- Use `,` to delimit elements of a list expression, like `[ 1, 2, 3 ]`
- Do something about `?` meaning two different things depending on where it occurs (`{ x ? "" }:` vs `x ? ""`)
- Better support for static analysis
- [Syntax for hexadecimal numbers](https://github.com/NixOS/nix/pull/7695)
- [Fix multiline strings](https://github.com/NixOS/nix/issues/3759)

Other discussions around language changes:

- [Make path semantics less surprsing](https://github.com/NixOS/nix/issues/7338)
- [RFC 110](https://github.com/NixOS/rfcs/pull/110) proposing a alternatives to the `with` expression
- [Nix 2 – a hypothetical syntax](https://md.darmstadt.ccc.de/s/nix2)
- [Nix language changes](https://gist.github.com/edolstra/29ce9d8ea399b703a7023073b0dbc00d)

# Detailed Design

## Design goals

1. New versions of the Nix language evaluator should to stay backward compatible with existing Nix expressions.

1. A Nix expression written in a newer version of the language should never work with older evaluators not supporting that version.

1. Changes to the language, especially backward-incompatible changes, should remain a rare exception.

## Language versioning

1. The language version consists of two integers, denoting a major and a minor component.

   <details><summary>Arguments</summary>

   - (+) Can distinguish additions from breaking changes
       - (-) This may not be needed for our use case, since any addition to an expression will break for older evaluators even if the major version matches
       - (+) Weaker forms of this are already in use:

         - `builtins.foo ? workaround`
         - `builtins.langVersion < 6`
         - `builtins.nixVersion`

         An explicit version declaration would would make obvious to readers what to expect and allow for better error messages.

   - (+) Simplifies evaluator implementation: pure additions can be guarded by the minor version cleanly within a major version's evaluator code
   - (-) Requires more characters to account for the added expressiveness
       - This may be relevant depending on where it has to be encoded
   - (+) Follows well-known convention of [Semantic Versioning](https://semver.org/)
   - (-) May be confused with the version number of the Nix release

   [`builtins.langVersion`]: https://github.com/NixOS/nix/blob/26c7602c390f8c511f326785b570918b2f468892/src/libexpr/primops.cc#L3952-L3957

   </details>

   <details><summary>Alternatives</summary>

    - The language version is a natural number.
       - (+) Formally decouples the Nix language version from the Nix version
         - (+) The Nix language is supposed to change much less often than the rest of Nix
         - (-) There are two version numbers to keep track of
         - (-) Makes more evident that the Nix language is a distinct architectural component of the Nix ecosystem
       - (+) It's currently handled that way, no change needed apart from documentation
            - See [`builtins.langVersion`] (currently undocumented)
       - (+) Simple and unambiguous
       - (+) Concise, even in the long term, since the language is supposed to change very rarely
   - [Calendar Versioning](https://calver.org/)
       - (+) Provides information on when changes happened
           - (-) This is not needed because only compatibility information is needed
       - (-) Requires a minimum amount of characters
           - This may be relevant depending on where it has to be encoded
       - (+) Restricting to only the year would force language changes to be rare
           - (+) This would allow obvious synchronisation points with Nixpkgs releases
           - (-) It may be too much policy encoded in a mechanism
   - Use version numbers of Nix stable releases for specifying the version of the Nix language
      - (+) More obvious to see for users what the current Nix version is rather than `builtins.langVersion`
      - (-) Would tie alternative Nix language evaluators to the rest of Nix
        - (-) One can add a command line option such that it is not more effort than `nix --version`
          - (+) That requires adding another built-in to the public API
        - (-) Using a language feature requires an additional steps from users to determine the current version
            - (-) Requires adding another command line option to the public API
      - (+) The Nix language version is decoupled Nix version numbering
        - (+) It changes less often than the Nix version
          - (-) That was probably due to making changes being so hard
            - (+) The language changing slowly is a desirable property for wider adoption
        - (-) There are two version numbers to keep track of

   </details>

1. The language version for Nix expressions is denoted in special syntax, at the beginning of a parse unit.
   A parse unit is any text stream, e.g. a file or string.
   The evaluator must ignore a shebang line (starting with `#!) at the start of files, to leave room for additional tooling.

   The language version declaration is optional if it is instead made in an external per-project file.
   The details of the per-project file syntax are out of scope for this proposal.

   <details><summary>Arguments</summary>

   - (+) Will prevent older evaluators from evaluating expressions written in a newer language version following this proposal (no forward compatibility)
   - (+) Precedent: [Perl `use VERSION`]
   - (-) The errors on older evaluators will be opaque
     - (+) Syntax can be made self-describing and human-readable to alleviate that to some extent
   - (-) The syntax has to be fixed forever if one wanted to provide meaningful errors on language upgrades
     - This has the same trade-offs as when introducing the new syntax to begin with
   - (-) Editor support is made harder, since it requires switching the language based on file contents
     - (+) Making the language version accessible at all will probably outweigh the costs
   - (+) Using a per-project file avoids littering every file with version declarations.

   </details>

   <details><summary>Alternatives</summary>

   - Use a magic comment at the beginning of the file
       - (+) Allows for gradual adoption: opt-in until semantics is implemented in Nix *and* the first backward-incompatible change to the language is introduced
         - (-) This will produce surprising results if the next language version preserves syntax but changes semantics (forward compatibility)
         - (-) Requires the first language version following this proposal to be syntactically incompatible with the current language to avoid forward compatibility
       - (+) Can be made self-describing and human-readable
       - (+) Follows a well-known convention of using [magic numbers in files](https://en.m.wikipedia.org/wiki/Magic_number_(programming)#In_files)
       - (-) May make the appearance that changing the language is harmless
         - (+) The convention itself is harmless and independent of the development culture around the language
         - (-) There is a chance of abusing the magic comment for more metadata in the future
       - (-) At least one form of comment is forever bound to begin with `#` to maintain compatibility
         - Forward compatibility is undesirable anyway
       - (-) Requires support by all tooling, lose semantics otherwise

   - Use `assert builtins.langVersion` in the first line of the file
       - (+) Produces more telling error messages in existing evaluators
       - (+) Future evaluators could be augmented to treat this as specially for better errors
         - (-) Special treatment may confuse users: why does `assert` at the beginning of a file work differently than somewhere else?
       - (-) Bulky expression that can only be replaced by the magic string solution or kept forever

   - Denote the language version in the file extension
     - (+) Sidesteps misinterpretation by keeping metadata out of the actual data
     - (-) In general it does not prevent forward compatibility with current evaluators.
     - (+) Makes accidental mixing of versions impossible at the syntax level
         - Have to specify the file extension when importing a file
         - (-) Have to rename all files in a project to change the version
           - This is somewhat worse than replacing a magic string which is fixed to the beginning of the file
     - (-) Makes filenames longer, introduces visual noise
         - This is the cost of being explicit
     - (-) Enforces narrow restrictions on what information can be encoded and how
       - The only reasonable alternative is `-`, e.g. `default.7-nix`
         - `.`
             - (-) Nixpkgs has been packaging Linux kernels as `linux-${major}.${minor}.nix`
               - This may break backwards compatibility of newer evaluators with existing code in surprising ways
         - No separator
             – (-) Hard to discern visually
         - `-`
             - (+) Visually not intrusive
         - `_`
             – (-) Visually more intrusive
         - `^`
             – (-) Overlaps with derivation output syntax
         - All of the following characters will interfere with some tooling:
             - `!` - shells
             - `"` - shells
             - `#` - URLs
             - `$` - shells
             - `%` - URLs
             - `&` - shells, URLs
             - `+` - URLs
             - `,` - natural language
             - `/` - paths, URLs
             - `:` - URLs
             - `;` - shells
             - `=` - URLs, Nix language
             - `?` - URLs
             - `@` - URLs, Nix language
             - `\` - Windows paths
     - (-) `default.nix` resolving needs specification:
         - If for `import ./foo`, all of `./foo/default.nix{6,7,8}` exist, pick the one matching the version used by the evaluator, otherwise fail
             - Then you'd have to specify a file using a different version explicitly

   - Language versioning per "project" in a sidecar file
       - (+) This would easily allow inheriting the language version across imports (obviating many specifications in this proposal)
       - (-) There is currently no notion of "project" in the Nix language
         - (-) Attempting to establish one would be a large undertaking and not immediately help solving the problem at hand
       - (-) Cannot be introduced gradually, particularly relevant for a large codebase like Nixpkgs
       - (-) It would require a separate language to encode project metadata such as the language version
           - The `edition` field was [removed from the flakes schema](https://github.com/NixOS/nix/commit/e5ea01c1a8bbd328dcc576928bf3e4271cb55399) for that reason, as it not not allow distinguishing data from metadata
           - (+) Other languages do the same (Python, Haskell, Rust, JavaScript, ...)
             - (-) Recursive (albeit smaller) problem of managing the additional language for project metadata

   </details>

1. The following syntax is used for declaring the language version of a Nix expression:

   ```
   version \d*.\d*;
   ```

   This implies that if no language version is specified in a Nix file, it is written in version 6 (the version implemented in the stable release of Nix at the time of writing this RFC).

   The syntax is open for bikeshedding.
   Alternatives should be very short and self-describing.

   <details><summary>Arguments</summary>

   - (+) Short and fairly self-descriptive
   - (+) Not an invasive change

   </details>

   <details><summary>Alternatives</summary>

   - `use v\d*\.\d*;`
     - (+) Shorter
     - (-) Does not explain much
   - `Nix language version \d*\.\d*`
     - (+) Says it all
     - (-) Very long
   - `with import <language> \d*\.\d*;`
     - (+) Allows for forward compatibility hacks such as better error messages
     - (-) Will likely mislead beginners to think this is has the same semantics as the original `with import ...;`

   </details>

1. As an exception, the language version declaration is optional for Nix expressions passed directly as an argument to the evaluator and on the REPL, and can be specified with a separate parameter to the evaluator.
   If no version is specified in a bare Nix expression, assume the most recent language version supported by the evaluator.

   This is to ease use of the REPL and evaluating ad hoc expressions.

   <details><summary>Alternatives</summary>

   - Make it mandatory
       - (-) This will be very inconvenient to use in the REPL

   - Assume the evaluator's current version
       - (-) When the evaluator advances in language version, evaluation may fail on existing code
       - (-) Defeats the purpose of explicit versioning: Which evaluator to use for a given file is left unspecified
           - (-) Following the latest evaluator version may inadvertently break the code for older evaluators
       - (+) Don't have to look up the latest version of the Nix language when writing code
       - (+) Does not clutter the file names for what is supposed to be the latest version of the code

   </details>

1. Each time the language specification (currently as embodied by the Nix language evaluator) is changed:

   - When backward compatibility is preserved, the minor version number is incremented.

     A backward compatible change to the language means that all existing expressions written for the prior version will still evaluate to the same result.
     Examples include additions of `builtins`, operators, or syntactic constructs.

   - When breaking changes are introduced, the major version number is incremented.

     Examples include semantic changes or removal of `builtins`, operators, or syntactic constructs.

   Values are decidedly not covered by versioning, but must instead stay the same indefinitely.
   In particular, there must not be, e.g., string values internally tagged with different language versions.
   This constraint can be loosened with a follow-up RFC.

   <details><summary>Arguments</summary>

   - (+) The principled solution: guarantees reproducibility given a fixed language version
   - (+) Makes explicit current practice of adding features to the language, and allows introducing breaking changes in a controlled fashion, which is currently not possible at all
   - (-) Implies additional overhead in development effort:
     - Either for Nix maintainers to accommodate that practice in the release lifecycle
       - For example, one would have to batch language changes for a major version bump to limit the number of increments
         - (+) This would be beneficial for alternative implementations in terms of churn and effort to keep up
     - Or implementors of alternative evaluators catching up with changes
       - (+) Specifying the language precisely via the version actually offers alternative implementations an alternative to catching up: only support a given language version
   - (+) This essentially nudges one to organise Nix language (specification or evaluator implementation) development to be more independent of the rest of Nix
     This is good, since it in turn forces stronger separation of concerns and more architectural clarity
   - (-) Prohibits best-effort attempts at evaluating expressions with possibly incompatible evaluators
     - (+) With the proposed level of strictness, one doesn't have to rely on best effort but can instead be explicit
   - (-) Fixing evaluator bugs (i.e., clearly unintended behavior) after releases would technically require a version bump and therefore (theoretically) cooperation by expression authors
     - This could be communicated with an increment in the Nix patch-level version, as is already practice

   </details>

   <details><summary>Alternatives</summary>

   - Bump major version only when evaluation result on prior version evaluators would be *substantially* different
     - (+) Leaves room for judgement by developers
       - (+) Allows controlling progression of versions to some degree
     - (-) Conversely, leaves room for sneaking in breaking changes unannounced
       - (-) This loses compatibility guarantees we'd get from a stricter paradigm
       - (-) Deprives expression authors of ability to be selective with evaluator versions
     - (-) Due to hashing this is often not much different from taking *any* change into account

   </details>

1. Semantics are preserved across file boundaries for past language versions.
   In other words, code written in an old language version evaluates to the same result when used from the new versions for all input values which are legal on the old version.

   This should be fairly straightforward to implement since values passed around in the evaluator can carry all the information needed to force them.
   Newer parts of the evaluator can always wrap their values in interfaces that are accepted by older parts, as far as possible.

   Examples:
   - [Best-effort interoperability](#best-effort-interoperability)
   - [Preserving semantics across version boundaries](#preserving-semantics-across-version-boundaries)

  When new value types are added to the language:

  - Passing new values to functions is allowed, as it cannot be prevented anyways due to laziness and composite types (like lists).
  - Any values of unknown type to code from an older Nix version are treated as the opaque "external" type (which already exists for things like plugins).
    Attempts at using them other than passing them around will thus cause type errors.

   <details><summary>Arguments</summary>

   - (+) This allow for incremental evolution of code bases without having to change existing code at all.
   - (+) Expressions that are valid today will still be valid as long as a given evaluator supports the language version they are written in
   - (+) The root of evaluation is always at a language version determined by the user, and authors of expressions in newer language versions are responsible (and made aware of the fact by the file name signaling proposed here) to interoperate with existing code
   - (-) Passing around incompatible values (e.g. builtins or data types) between language versions can lead to surprising errors
     - (+) We can postpone dealing with particular issues as they arise, but the general setup should support most cases on a best(-and-minimal)-effort basis
     - (+) In any case, such breakages will only happen when adding new code, and will never break existing code

   </details>

   <details><summary>Alternatives</summary>

   - Do not support version interoperability at all
     - (+) Avoids any unforseen issues at no cost
     - (-) Prohibits incremental changes, as any language update will require updating all files involved
   - Do not support passing values to functions from older language versions
     - (-) Calling functions is the most common use case, and you can hardly do anything without it

   </details>

1. The backward compatibility mechanism must be "zero cost" when not used, meaning that no performance overhead must be paid when no legacy Nix files are imported.

   <details><summary>Arguments</summary>

   - (+) This additional implementation constraint encourages being conservative with substantial changes.

   </details>

1. It is not possible to import expressions written in newer versions.

   Example: [Expressions are not forward compatible](#expressions-are-not-forward-compatible)

1. The Nix evaluator provides options to issue deprecation warnings and errors against a language version newer than the one under evaluation.
   A detailed design is provided in the next section.

   <details><summary>Arguments</summary>

   - (+) This allows systematic migration of existing code written for prior evaluators to the most recent language version
   - (+) It will notify users about what's going on instead of just breaking
   </details>

1. Language versions prior to 6 are not supported.

   <details><summary>Arguments</summary>

   - (+) Does not require additional development effort
   - (+) Prior langauge versions are not fully supported by current code already, and the rest of this proposal argues to deprecate old versions in the future in order to keep the implementation manageable
   - (-) Legacy code will not get support for managing compatibility

   </details>

1. The Nix language evaluator provides a command to output the most recent Nix language version.
   This command provides options to list all Nix language versions supported by the evaluator.

   <details><summary>Arguments</summary>

   - (+) This is for convenience to determine which features are available

   </details>

1. Whenever Nix drops support for evaluating prior language versions, a major version bump is required.

   Example: Assuming the current language version is 8.0, the Nix release version is 2.20, and support for language version 6 is dropped, the next Nix release must be version 3.0

   <details><summary>Arguments</summary>

   - (+) This enforces that existing code that works will not break inadvertently when upgrading Nix

   </details>

   <details><summary>Alternatives</summary>

   - Separate Nix development from the Nix language entirely and keep it outside of the scope of this proposal
     - (-) Currently the upstream Nix language evaluator and compatibility of expressions in Nixpkgs is closely tied to the rest of Nix, and have to take that into account
       - Further separation of concerns is out of scope for this proposal

   </details>

## Deprecation warnings and errors

1. Each language construct to deprecate relative to a prior version is given a symbolic name.
   There is a way to refer to all language constructs.

   Examples: `url-literal`, `let-body`, `int-leading-zeros`

   These symbolic names must not be reused in future versions.
   Names for experimental language constructs of prior versions can be reused.

   <details><summary>Arguments</summary>

   - (+) Disallowing reuse precludes dealing with change of meaning across versions
   - (+) Not considering experimental features simplifies their handling and is not required by them being exempt from compatibility guarantees.

   </details>

1. The following options for issuing warning and errors are supported:

  - Default:
      - Issue warnings on deprecated language constructs without considering imported expressions written in prior versions
  - Don't warn (selection):
      - Do not issue warnings for selected language constructs
  - Errors instead of warnings (selection):
      - Throw an error instead of a warning for selected language constructs
      - The error setting overrides the warning setting
  - Recursive (version bound):
      - Issue warnings or errors on imported expressions written in prior versions (higher or equal than the version bound)
  - Verbose mode. During evaluation:
    - In non-verbose mode, issue a message once for each deprecated construct
    - In verbose mode, issue a message for each occurrence

   For example, this can be exposed as the following flags:

   ```
   --lang-no-warn=all
   --lang-no-warn="url-literal let-body int-leading-zeroes non-attr-select"
   --lang-error=all
   --lang-error="url-literal let-body int-leading-zeroes non-attr-select"
   --lang-warn-recursive=6
   --lang-warn-verbose
   ```

   The naming may need some bikeshedding. For example, one could use the same syntax as with C-compilers (probably not though):

   - `-Wall`
   - `-Wno-`
   - `-Werror=`

   <details><summary>Arguments</summary>

   - (+) Disallowing reuse avoids dealing with change of meaning across versions
   - (+) Not considering experimental features simplifies their handling and is not required by them being exempt from compatibility guarantees.
   - (-) Maintenance burden for everyone using these old constructs, or evaluating old revisions of Nixpkgs.
   
   </details>

   <details><summary>Alternatives</summary>

   - Opt-in warnings
     - (-) Can't really make breaking changes as people won't be warned ahead of time
   - No warnings, just errors
     - (-) Doesn't offer a transition window to users
     - (+) Easier to implement

   </details>

1. A the end of the evaluation, print statistics and explanations.
   The specifics of displaying warnings and errors is up to implementation, but should include the symbolic name of the langauge construct in question.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

## Show the current Nix language version

```console
nix --language-version
7.0
```

```console
nix --supported-language-versions
6.1
6.2
6.3
7.0
```

## Version interoperability

### Versioned built-ins can be passed across file boundaries

```nix
# a.nix
version 6.1;
builtins.langVersion
```

```nix
# b.nix
version 7.0;
[ (import ./a.nix) builtins.null ]
```

```console
$ nix-instantiate --eval b.nix
[ 6 null ]
```

### Expressions are not forward compatible

```nix
# a.nix
version 6.1;
import ./b.nix
```

```nix
# b.nix
version 7.0;
builtins.null
```

```console
$ nix-instantiate --eval a.nix
error: unsupported Nix language version 7.0
```


### Best-effort interoperability

```nix
# a.nix
version 6.1;
{ increment }:
increment 1
```

```nix
# b.nix
version 7.0;
import ./a.nix { increment = x: x + 1 }
```

Since `increment` written in version 7.0 carries its own implementation with it, forcing it within an expression written in version 6.1 just works:

```console
$ nix-instantiate --eval b.nix
2
```

### Preserving semantics across version boundaries

```nix
# a.nix
version 6.1;
{ float };
toString float
```

```nix
# b.nix
version 7.0;
{
  old = import ./a.nix 1.1;
  new = toString 1.1;
}
```

```console
$ nix-instantiate --eval b.nix --strict
{ old = "1.100000"; new = "1.1"; }
```

### Pathological example

Usually existing code will be interacted with by calling functions.
When passing values from newer versions to functions from older versions of the language, interoperatbility can only be supported on a best-effort basis.

```nix
# a.nix
version 6.1;
{ value }:
value + 1
```

Here we pretend that language version 7 introduced a new value type and syntax for complex numbers:

```nix
# b.nix
version 7.0;
import ./a.nix { value = %5 + 7i%; }
```

```console
$ nix-instantiate --eval bnix
error: unsupported value type `complex` at built-in operator `+`
```

In the following example, assume version 7.0 *removed* floating point numbers, such that they can no longer be used.

```nix
# a.nix
version 6.1;
1.1
```

```nix
# b.nix
version 7.0;
(import ./a.nix) * 2
```

```console
$ nix-instantiate --eval b.nix
error: unsupported type `float` for multiplication
```

This is consistent with best-effort interoperability:
Old code keeps working on its own, and new code has to be adapted because it was not there before the breaking change.

While this requires additional effort to adopt the new language version, expression authors can always recourse to writing new code in older versions while using newer evaluators.
This in fact allows for creating compatibility wrappers as needed.

## Deprecation warnings

```
nix eval --json ./test.nix
warning: URL literals are deprecated (url-literal)
         please replace this with a string: "https://nixos.org"

       at test.nix:1:1:

            1| https://nixos.org
             | ^

"https://nixos.org"

warning: The following deprecated features were used:
  - url-literal (httsp://nixos.org), 1 time

  Add `--lang-warn-verbose` to show all occurrences
  Use `--lang-no-warn=url-literal` to disable this warning.
  Use `--lang-error=url-literal` to issue errors instead of warnings.
```

## Drawbacks

Allowing multiple language versions to coexist complicates implementation of evaluators and support tooling, and makes comprehensive test coverage harder.
All else being equal, it may increase maintenance burden and the likelihood of introducing bugs.

Providing a pathway for introducing breaking changes bears the risk of version proliferation.
We argue though that the implementation overhead incurred by the strict compatibility requirements will by itself balance that out.
At least such a trade-off now could then be made to begin with, as currently breaking changes cannot be made at all.

# Alternatives

- Keep the language as implemented by Nix compatible, but socially restrict the usage of undesirable features.
    - (+) Roughly matches the current practice, no technical change needed
    - (+) Maintains usability of old Nixpkgs versions (up to availability of fixed-output artifacts)
    - (+) Does not break third-party codebases before making a decision, keeping Nix a dependable upstream
        - (-) This proposal does not allow for breakages unless there is some eventual phase-out of support
    - (-) Strict enforcement requires extra tooling that this proposal would obviate
    - (-) The implementation of the features that are no longer desirable still incur complexity and maintenance cost
        - (-) It's still not really possible to make changes to the language

- Introduce changes to the language with language extensions or feature flags
  - (-) Combinatorial explosion
      - See [Haskell language extensions] for real-world experience
  - (-) Even more maintenance overhead
  - (+) Allows gradual adoption of features
      - (-) We already have experimental feature flags as an orthogonal mechanism, with the added benefit that they don't incur support costs and can be dropped without loss

- Never make breaking changes to the language
    - (+) No additional maintenance effort required
    - (-) Blocks improvements
    - (-) Requires additions to be made very carefully
        - (-) Even incremental changes are really expensive that way
    - (-) Makes solving some well-known problems impossible

- Continue current practice
    - (-) There is no process for breaking changes
    - (-) Breaking changes are not always announced
    - (-) There are no means of determining compatibility between expressions and evaluator versions

# Prior art

- [Perl `use VERSION`]

  [Perl `use VERSION`]: https://perldoc.perl.org/functions/use#use-VERSION

  Many similarities, with versions declared per file and having to deal with interoperability.

- [Rust `edition` field]

  Rust has an easier problem to solve.
  Cargo files are written in TOML, so the `edition` information does not have to be part of Rust itself.

  [Rust `edition` field]: https://doc.rust-lang.org/cargo/reference/manifest.html#the-edition-field

- [Haskell language extensions]

  Haskell allows enabling separate language features per file.

  [Haskell language extensions]: https://downloads.haskell.org/ghc/latest/docs/users_guide/exts/intro.html

- JaveScript modules
    - .cjs and .mjs extensions for commonjs/es-modules syntax variants
    - `function() { "use strict"; return 10 }`

- [Go language changes proposal](https://github.com/golang/proposal/blob/master/design/28221-go2-transitions.md#language-changes)

  The document features a discussion of multiple approaches to evolve the language without breaking existing code, and comes to the similar conclusions as our proposal, mainly:
  - One needs a way to specify the language version
    - Interestingly, Go developers decided to equate the language version with the compiler release version, despite admitting that this may be confusing
  - Keep support for code written in older language versions
  - Breaking down into features that can be enabled separately is not practical

- [Flakes `edition` field]

  There had been an attempt to include an `edition` field into the Flakes schema.
  It did not solve the problem of having to evaluate the Nix expression using *some* version of the grammar.

  [Flakes `edition` field]: https://discourse.nixos.org/t/nix-2-8-0-released/18714/6

# Future work
[future]: #future-work

- Define a roadmap to introduce the next language versions, for example:
  - Major version 7 commits to changes that do not require additional work on Nixpkgs:
    - Introduce the version declaration, required to distinguish major versions 6 and 7
    - Remove `builtins.langVersion`, as it's not needed any more
    - Deprecate URL literals
    - Deprecate the `let-body` syntax
    - Drop support for leading zeroes on integers
    - Formalise change in float representation
  - Major version 8 is released only when version annotations according to version 7 are fully supported in Nixpkgs

