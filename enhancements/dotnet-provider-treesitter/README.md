---
title: enhance-dotnet-analysis-tree-sitter
authors:
  - "@djzager"
reviewers:
  - "@eemcmullan"
  - "@shawn-hurley"
  - "@pranavgaikwad"
approvers:
  - "@eemcmullan"
  - "@shawn-hurley"
  - "@pranavgaikwad"
creation-date: 2024-08-23
last-updated: 2024-08-23
status: provisional
see-also:
  - "/enhancements/dotnet-provider/README.md"
  - "/enhancements/kantra-dotnet-framework/README.md"
---

# Enhance .NET Analysis with Tree-sitter

## Summary

This enhancement proposes migrating the dotnet-external-provider from relying
on the `csharp-ls` language server to using
[Tree-sitter](https://tree-sitter.github.io/tree-sitter/) for analyzing .NET
applications.

## Motivation

The current reliance on `csharp-ls` for .NET code analysis has shown
limitations in scalability and complexity. Specifically, it struggles to
accurately resolve references, leading to incomplete or inaccurate results.

Although the main motivation to this enhancement is to make .NET analysis
functional, this has the added benefit of making it possible to analyze .NET
Framework applications in Linux containers. I was able to generate reliable
Abstract Syntax Trees (ASTs) with .NET Framework source files.

### Goals

- Transition the dotnet-external-provider from `csharp-ls` to Tree-sitter for
    code analysis.
- Expand the rule structure to include the class.
- Improve the accuracy and performance of finding meaningful references in .NET
    codebases.

### Non-Goals

- This enhancement does not aim to implement cross-file analysis or handle
    project-wide symbol resolution, which may be better addressed by tools like
    Roslyn in future work.
- It does not seek to replicate the full feature set of `csharp-ls` but rather
    to focus on improving the accuracy and usability of specific analysis queries.

## Proposal

### User Stories

#### Story 1 As a developer, I want to write simple rules that allow me to find
all instances of a specific class or method within a given namespace, so that I
can refactor or replace deprecated APIs more effectively.

#### Story 2 As a project maintainer, I want to improve the performance of code
analysis across large .NET codebases, reducing the time it takes to identify
relevant code patterns.

### Implementation Details/Notes/Constraints

- **Rule Structure**: The new rule structure will require a namespace, an
    optional class, and an optional pattern (though class would be required with
    pattern). This will simplify both the writing of rules and the internal logic
    required to match these patterns.
- **Tree-sitter Integration**: The transition will involve integrating
    Tree-sitter to parse the AST of .NET files, focusing on namespace and class
    declarations, as well as method and member access expressions.

### Security, Risks, and Mitigations

- The change in analysis tool does not introduce new security risks.

## Design Details

### Test Plan

- **Unit Tests**: Implement unit tests for the new rule structure and
    Tree-sitter integration.
- **Performance Tests**: Conduct performance testing on large codebases to
    ensure that the new approach is at least as fast as, if not faster than, the
    current `csharp-ls`-based analysis.
- **End-to-End Tests**: Validate that the new system correctly identifies
    patterns across a variety of .NET projects, ensuring accuracy in both simple
    and complex scenarios.

### Upgrade / Downgrade Strategy

Given the dev preview state of this feature in Konveyor, there are no upgrade
considerations.

## Implementation History

- 2024-08-23: Initial proposal created and submitted for review.

## Drawbacks

- **Effectiveness**: While some investigation has been done, the C# grammar is
    well supported in tree-sitter, and the results were promising, there is not
    yet a guarantee that this approach will solve the problem of unreliable
    analysis results.

## Alternatives

- **Use Roslyn**: Shift to using Roslyn for more advanced analysis. While
    Roslyn is powerful, it may be more complex to integrate and may not be suitable
    for all environments, especially non-Windows platforms.
