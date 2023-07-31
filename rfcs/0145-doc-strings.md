---
feature: doc-comment-standard
start-date: 2023-03-27
author: hsjobeki
co-authors: --
shepherd-team: @DavHau; @sternenseemann; @asymmetric
shepherd-leader: @lassulus
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

Propose a standard format specification for doc-comments.

This RFC includes two concerns that define a doc-comment:

- Outer format rules to allow distinction between regular comments and doc-comments
- Inner format rules that describe the required format of a doc-comment.

However, both concerns relate closely to each other; It makes sense and reduces bureaucracy to address both in a single RFC.
# Definitions

- **doc-comment**: A comment following a specific structure, intended to document the code's API.
- **internal comment**: A free-form comment whose target audience is someone insterested in the code's implementation.


# Motivation
[motivation]: #motivation

The following are the envisioned goals.

- Be able to generate documentation from code for any nix expression.
- Make doc comments easy to write and read
- Standardize a format for doc comments that further RFCs can extend
  - E.g. extend with code examples that can be converted to tests (aka doctests).

This RFC is a significant change to existing documentation conventions.
It allows distinguishing between internal and doc comments (see "Definitions" section). Having distinction is essential because internal comments should not end up in generated documentation.

> Note: Generating static documentation is a controversial topic in nixpkgs.[^1][^2][^3][^4] We found that it is very hard to generate accurate documentation statically. A correct solution would involve the evaluation of expressions in some way.

[^1]: [Unmerged PR: nix repl doc-comments](https://github.com/NixOS/nix/pull/1652), 2017
[^2]: [Unmerged PR: doc-strings for functors](https://github.com/NixOS/nix/pull/5527), 2021
[^3]: [discourse: users asking how to obtain documentation](https://discourse.nixos.org/t/how-to-generate-documentation-from-arbitrary-nix-code/22292), 2022
[^4]: [discourse: aksing for consensus on documentation](https://discourse.nixos.org/t/any-consensus-on-documentation-generation-tool-for-nixpkgs-manual/15550), 2021

# Non-goals

- Discuss in which tool doc-comments are parsed and rendered. This could be an external tool, or Nix, or something else entirely, but that's out of scope for this RFC.
- Providing a migration path for existing comments. This is expected to require some amount of manual work. See "Future work" section.
## Current State

Currently, Nix does not have doc comments. Over time some informal conventions have crystallized, but they were never formally specified.
Within this RFC, we want to clarify which convention we should use, harmonize and improve the format for doc-comments to achieve the envisioned goals.

A third-party tool called [nixdoc](https://github.com/nix-community/nixdoc) has emerged, which codifies its own rules as to the internal and external formats of a Nix doc-comment. This tool has seen some adoption, notably for the `nixpkgs.lib` functions.

The format of a doc-comment was, however, never specified. The nix community does not understand the rules required by `nixdoc`, which we believe is one of the reasons for its lack of adoption.

Here is an example of the format understood by *nixdoc*:

```nix
  # nixpkgs/lib/trivial.nix

  /* The constant function

     Ignores the second argument. If called with only one argument,
     constructs a function that always returns a static value.

     Type: const :: a -> b -> a
     Example:
       let f = const 5; in f 10
       => 5
  */
  const =
    # Value to return
    x:
    # Value to ignore
    y: x;
```

## Current problems

### Multiplicity of formats

Within nixpkgs alone, several conventions for doc-comments have emerged, see [1], [2] and [3].

Notably, most doc-comments utilize some fraction of the of CommonMark syntax, even if they are not meant to be rendered.

[1]: https://github.com/NixOS/nixpkgs/blob/master/pkgs/build-support/trivial-builders/default.nix
[2]: https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/make-derivation.nix
[3]: https://github.com/NixOS/nixpkgs/blob/master/nixos/lib/make-disk-image.nix

In general, the format for writing documentation strings is **not formally specified**.

Among the formats encountered in the wild, the one used in `nixpkgs/lib` is the only one intended to be rendered as part of an API documentation, via the nixdoc third-party tool, whose syntax has not been standardized.

Extending the scope of *nixdoc* is not the primary goal. Instead, we should find formal rules for writing *doc-comments*. Tools like *nixdoc* can then implement against this RFC instead of the format relying on nixdoc implementation details.

### Impossible to differentiate from internal comments

The lack of a formal definition of a doc-comment also means there is no reliable way to distinguish them from internal comments, which would result in automatically-produced API documentation which includes the wrong type of comments.

### References to the problems above

> Note: That specific collection of links should show the amount of inconsistency within the Nix ecosystem, which is a target of this RFC.

#### nixpkgs - doc-comment examples

- [lib/attrsets](https://github.com/NixOS/nixpkgs/blob/master/lib/attrsets.nix)
- [trivial-builders](https://github.com/NixOS/nixpkgs/blob/master/pkgs/build-support/trivial-builders.nix)
- [stdenv/mkDerivation](https://github.com/NixOS/nixpkgs/blob/master/pkgs/stdenv/generic/make-derivation.nix)
- [nixos/lib/make-disk-image](https://github.com/NixOS/nixpkgs/blob/master/nixos/lib/make-disk-image.nix)

#### Current tools

We collected some Tools that work directly with the nix AST and comments.
Those would benefit from a standardized doc-comment.

- [noogle](https://noogle.dev) - Nix API search engine. It allows searching functions and other expressions.
- [nix-doc](https://github.com/lf-/nix-doc) - A Nix developer tool leveraging the rnix Nix parser for intelligent documentation search and tags generation
- [manix](https://github.com/mlvzk/manix) - A fast CLI documentation searcher for Nix.

# Detailed design
[design]: #detailed-design

Each subsection here contains a decision along with arguments and counter-arguments for (+) and against (-) that decision.

> Note: This RFC does not promote any tool; It focuses only on the format, not implementation. At the same time, this RFC must be technically feasible.

## Discontinue nixdoc's custom format

**Observing**: Nixdoc currently provides some format that can generate documentation from a small fraction of files in nixpkgs, Namely the files for `.lib` functions. The directory provides a consistent community convention that allows the building of static documentation. Applying nixdoc to files outside that directory is only partially possible due to the lack of a standard doc-comment format.

**Considering**: Should we propose the `nixdoc` format as a community standard?

**Decision**: Discontinue nixdoc's custom format in favor of using something more generic.

<details>
<summary>Arguments</summary>

- (+) Stays primarily compatible with the currently used comments in lib.
    - (-) Outside the ./lib directory, nixodc is not a convention.
- (-) Is not widely understood.
- (-) The format is specific, should be more generic, and follow established conventions. (e.g., Markdown, Doxygen, JSdoc, Haddock)
- (-) The format is too complex and needs a whole list of complex rules to describe the exact format and content rules; this can also be observed in many false-formatted comments that do not render correctly.
    - (-) The sections are implicitly continued.
    - (-) The predefined section headers/keywords are very close to natural language and may accidentally be used.
        - (-) Syntax to start new sections does not follow conventions (e.g., `Example:`) In contrast JSdoc defines `@Example`; Markdown `# Example`; Doxygen `@example`; Haskell-Haddock `>>>`
    - (-) The content is implicitly rendered differently depending on predefined rules. Those rules need to be understood/feel natural.
- (+) Some usages outside of /lib directory indicate the process of adoption.
- (-) Some multiline comments become doc-comments, but it is unclear which comments have such a property.
    
</details>

## `/**` to start a doc-comment

**Observing**: Doc-comments' outer format should be a distinctive subset of regular comments. Nevertheless, it should allow native writing without an IDE or editor support.

**Considering**: `/** {content} */` where `{content}` is the inner format which is discussed later.

**Decision**: use `/** {content} */` as the outer format.

`Example`

````nix
{
  /**
  The identity function

  Describes in every detail why 
  this function is important and how to use it.

  # Examples

  ```
  id "foo"
  ->
  "foo"
  ```

  */
  id = x: x; 
}
````

<details>
<summary>Arguments</summary>

- (+) It is mostly compatible with the currently used multiline comments.
- (+) Is a strict subset of multiline comments, which allows multiline documentation natively.
- (+) Does not need editor support for productive usage. (In contrast to, e.g., `##`)
- (+) Allows copy-pasting content without the need for re-formatting.
  - (-) Partially re-formatting just like in multiline Nix strings (`''`) might still be needed.
- (-) Is visually less distinctive.
- (-) Indentation is more complex and visually less present.
    - (+) It would be most intuitive if its indentation logic implements Nix's magic multiline strings.
- (-) Takes up more vertical space
- (+) Takes up less horizontal space. (In contrast to, e.g., `##` lines do not have to be prefixed)
- (+) Only one character to change the semantics of the whole comment. 
    - (+) Allows to add and remove things from the documentation quickly.
    - (-) Accidentally adding/removing could happen.
    
</details>


## CommonMark as the content of doc-comments

**Observing**: Doc-comments' content should be intuitive to read and write and straightforward to render. The nixdoc convention is only widely adopted in certain places (e.g., /lib) in nixpkgs. It may also come from a need for more understanding of the current self-cooked format.

**Considering**: CommonMark as the content format.

**Decision**: CommonMark is the content of all doc-comments.

> Markdown is the most accepted and understood format for writing documentation in tech projects. Also, following the existing RFC-72 is highly profitable and consistent for the Nix ecosystem.

<details>
<summary>Arguments</summary>

- (+) CommonMark is the official format in nix; Decided in [RFC72](https://github.com/NixOS/rfcs/blob/master/rfcs/0072-commonmark-docs.md).
    - (+) It Would be consistent if this RFC builds upon the previous one.
    - (+) Further Arguments for CommonMark, in general, can be found in RFC72
- (+) Allows copy-paste from and to markdown files. We allow easy refactoring if documentation arises and needs to be split into separate files.
- (-) Strictly binding doc comments content to commonMark might restrict users.
    - (+) Users/Tools can still use regular comments or develop alternative solutions.
- (-) CommonMark does not specify the current rich features from nixdoc, such as predefined sections and structures that could be used as a source for automated toolings. Such as types for type-checking or examples to run automated-tests
    - (+) Future tools can still build their conventions on top of this RFC. They might not directly specify them in an RFC but be a tool developers choose for a specific codebase. However, we have yet to get that tools. So it is good when rich features remain unspecified.
  
</details>

## Place doc-comments before the referenced expression

**Observing**: Doc-comments currently are placed above the expression they document. More precisely, only named attribute bindings `foo = <expr>` can be documented. There is also the need to support documentation for anonymous functions. More generally, there is the need to document anonymous expressions where they are defined.

**Considering**: General reference logic between doc-comments and expressions.

**Decision**: Doc-comment refer to the expression that is **immediately** next to it. Only whitespaces are allowed between doc-comment and following expression.

> Note: We are working on a generic example implementation that covers about 90% of all documentation use cases.

<details>
<summary>Arguments</summary>

- (+) Doc-comments should have only one variant to reduce complexity. Referencing the next node seems straightforward. 
- (-) A variant that references the previous node in the AST should be avoided to reduce complexity.
- (+) Relation between documentation and the referenced implementation is straightforward and back-trackable.
- (-) Concrete Implementation might be complex.
- (-) Unclear if complete documentation might also need backward references. 
    - (-), e.g., rust uses backward references only at the top of file comments.
    - (+) Nix files can only have ONE expression; the next AST Node, in case of top-of-file comments, is thus always only that one expression. (Unlike in Rust)
- (-) Tools need to be smart enough to understand asignments `=` and other forms of creating names for anonymous expressions. (e.g., `callPackage` and `import` )
    - (+) Tools can still come up with other solutions that do not involve calculating everything dynamically from nix code but could also involve a static configuration.
    - (+) The whole `tool point` is an implementation detail as long as it is not impossible. The current tool, `nixdoc`, already proves that it is possible to have static documentation to a certain degree.

</details>

## Single-line doc-comments (do not exist)

**Observing**: Nix offers two variants of comments; single- (`#`) and multi-line comments (`/* */`). There may be use cases where it is desirable to have a form of single-line comments subtyped for doc-comment purposes.

**Considering**: Single-line comment for documentation. (Starting a doc-comment with `#`)

**Decision**: Single-line comments (starting with `#`) **cannot be used** in any form for documentation puposes.

<details>
<summary>Arguments</summary>

- (-) Doc-comments should have only one variant to reduce complexity.  
- (-) documentation will likely take up more than one line.
- (-) If documentation grows bigger than one line, refactoring into a multiline-doc-comment must occur.
- (+) It Would be consistent with providing variants for both nix comments.
- (+) Offer the choice.
- (o) Single lines can also be concatenated to form multi-line documentation.
- (+) Takes up less vertical space
- (-) Visually confusing when every line starts with a reserved character.
    - (-) Potential visual conflicts with the content
- (+) Indentation of the content is clear.

</details>

## Doc-comment examples

This section contains many examples for some different use cases we; Visualize them and emphasize the previously discussed characteristics.

`somefile.nix`

````nix
{
  /**
  Documentation for the fundamental 'id' function
  
  # Examples
  
  ```
  id 1
  =>
  1
  ```
  
  # Type
  
  ```
  id :: a -> a 
  ```
  
  */
  id = x: x;
}
````

### Attribute bindings

`somefile.nix`

````nix
  {
    /**

      mapAttrs is a well-known function 

      # Examples

      ```
      # some code examples
      ```
    
    */
    mapAttrs = f: s: #...
  }
````

### Indentation follows know nix's `''` behavior

Indentation follows the know `''`-nix multiline strings behavior.
Making usage more intuitive and and one point less to think about.

````nix
  {
    /**
    Line 1
      Line 2
    */
    id = f: s: #...
  }
````
->
```markdown
Line 1
  Line 2
```

Or more advanced

````nix
  {
    /**
    # Title

    Some foo bar

    ## Subtitle

    Some more 

      indented 1

        indented 2

    */
    id = f: s: #...
  }
````
=>
```markdown
# Title

Some foo bar

## Subtitle

Some more 

  indented 1

    indented 2

```


### NixOS Module documentation

> Note: NixOS Modules also can produce documentation from their interface declarations. However, this does not include a generic description and usage examples.

`myModule.nix`
````nix
/**
  This is a custom module.
  
  It configures a systemd service.
  
  # Examples
  
  Different use case scenarios
  how to use this module
*/
{config, ...}:
{
  config = f: s: f s;
}
````

### Anonymous function documentation

`function.nix`
````nix
/**
  This is an anonymous function implementation.
  
  It does not have a name yet. 
  Nevertheless, documentation can be right next to the implementation.
  
  The name gets assigned later.
*/
{a, b}:
{
  sum = a + b;
}
````

### Anonymous expression documentation

`exp.nix`
````nix
/**
  This is an anonymous string.
 
  It is the documentation for the "anonymous expression" use case.
  
  Although this example is relatively superficial, there might be use cases.
*/
"-p=libxml2/include/libxml2"
````

# Drawbacks
[drawbacks]: #drawbacks

## Changes the existing comments inside the code base

This could be automated.

Also, the migration could be performed piecemal, starting perhaps with `nixpkgs.lib`, as it is already parsed by a tool (nixdoc), which could be modified for the new standard.

## Requires changes in existing tooling to produce documentation

nixdoc, the tool used to produce Nixpkgs library function documentation, would have to be modified to fit the new format.

This would be a small change, and anyway this RFC is agnostic to the tool used -- an entirely new tool could be developed, or the functionality be included as part of Nix.

# Alternatives
[alternatives]: #alternatives

## All considered outer formats

| Property / Approach | `##` | `/** */` | `Javadoc` | `/*\|` or `/*^`  |
|---|---|---|---|---|
| Inspired by | Rust | Current nixpkgs.lib | C++/Java/Javascript | Haskell Haddock |
| Changes the existing code by | Much | Less | Even More | Less |
| Needs Termination | No | Yes | Yes | Yes |
| Indentation | Clear | like Nix's multiline strings, thus **Intuitive** | Clear | ? |
| Needs vertical space  | No | Yes | Yes | Yes |
| Visual distinction from comments | High | Low | Medium | Medium |
| Needs Autocompletion (Language Server) to continue the next line. | Yes | No | Yes | No |
| Punctuation Variations / Amount of different special characters | 1 (Less) | 2 (Medium) | 2 (Medium) | 3 (More) |
| Markdown compatibility (also depends on indentation clarity) | Good, but visual conflicts with headings `# Title` | Good | Medium | Good |
| breaks when interrupted with newlines | Yes | No | ? | No |
| Simplicity (Brainload) | Medium | Simple | Complex | More Complex |

### Refactoring note

**Observing**: From a refactoring perspective, it might also be interesting to see how many conflicts the different formats would cause.

nixpkgs comments:

- `##` ~4k usages (most of them for visual separation, e.g., `###########`)
- `#` ~20k usages
- `/*` ~6k usages
- `/**` 160 usages (most empty ?)

Choosing `/**` or subsets would cause minor conflicts within current nixpkgs. While this is NOT the main reason for the final decision, it MUST be considered.

## Just free text as a content format

While this allows the most freedom, it is usually considered the best option, not creating any restrictions.

However, [RFC72](https://github.com/NixOS/rfcs/blob/master/rfcs/0072-commonmark-docs.md) defines commonMark as the official documentation format.
This is why we decided to follow this convention.

## Consequences of not implementing this

- By not implementing this feature, Nix gains no ability for tool-generated documentation.
- Documentation will be defined by nixdoc, not by the nix community.
- Many existing comments written for documentation will remain imperceptible.

# Unresolved questions
[unresolved]: #unresolved-questions

- Will `nix` itself implement native support like in rust -> `cargo doc`?

# Future work
[Future]: #future-work

## Migrate the existing comments

Reformatting existing doc-comments in Nixpkgs, but also filtering out false-positives, i.e. those that should not be part of the API documentation.

## Editor support

- Implement displaying the related documentation when hovering over an expression. (lspAction/hover)

Nix already offers a bunch of LSPs, e.g., [nil](https://github.com/oxalica/nil), [rnix-lsp](https://github.com/nix-community/rnix-lsp) are the most common ones.

## nixpkgs Manual tooling

The current tooling needs to be adopted to this change. With supporting the new format the currently existing scope can be retained to build the nixpkgs manual.

## Enhanced Documentation generators

We think that a future documentation tool could be out of one of the two following categories.

- (1) Tools that utilize static code analysis and configuration files. (This is the current approach)

- (2) Tools that use dynamic evaluation to attach name and value relationships and provides more accurate documentation with less configuration overhead.

  > We could solve such concerns in [`tvix`](https://tvix.dev/) or in `nix`, which could vend a tool that pre-evaluates expressions and gathers their respective documentation.

For the beginning it seems promising to start with the static approach (1). In the long term a dynamic approach (2) seems more promising and accurate but requires a much deeper understanding of compilers and evaluators.
However the border between those two variants is not strict and we might find future tools that fit our needs just perfectly.

## Type

- An RFC under construction specifies the used Syntax within the `# Type` Heading.
- The `type` feature should belong in the nix syntax. Try them within the comments first; This is still possible.

- see a [preview](https://typednix.dev) of an eventual future doc-type-syntax.

## Native support in Nix

- `NixOS/nix` could implement native support for doc-comments so that our users do not have to rely on nixpkgs or external tools. Those tools can still exist and provide more custom functionality, but documenting nix expressions should be natively possible.

## References

### Other Conventions

- [Rust](https://doc.rust-lang.org/stable/reference/comments.html#doc-comments)
- [Python](https://peps.python.org/pep-0257/)
- [JSDoc](https://jsdoc.app/)
- [Go Doc Comments](https://go.dev/doc/comment)

### Related tools

- [Nixdoc](https://github.com/nix-community/nixdoc)
- [Rustdoc](https://doc.rust-lang.org/rustdoc/how-to-write-documentation.html)
- [Doxygen](https://www.doxygen.nl/)

## Related discussions

- [@documentation-team (meet @ 13.Apr 2023)](https://discourse.nixos.org/t/2023-04-13-documentation-team-meeting-notes-41/27264)

- There is an actual but rather old PR (@roberth) that uses just comments to show documentation in the nix repl for functions. -> https://github.com/NixOS/nix/pull/1652