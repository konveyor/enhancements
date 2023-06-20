---
title: dotnet-provider
authors:
  - "@Chanakya-TS"
reviewers:
  - ?
approvers:
  - ?
creation-date: 2023-06-20
last-updated: 2023-06-20
status: provisional
see-also:
  - https://github.com/konveyor/analyzer-lsp
replaces:
  - 
superseded-by:
  - 
---

# Neat Enhancement Idea

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

## Summary

This enhancement aims to create an external provider for analyzer-lsp. The main purpose
for the provider is to help software migration from .NET Framework to .NET Core. The provider
will use Omnisharp-Roslyn (a popular LSP used by VS-Code) to analyze the .NET framework 
applications based on a custom ruleset.

## Motivation

.NET Framework has been used by many companies to create various applications for Windows. 
However, the newer, up-to-date version of .Net Framework: .Net Core, allows users to create 
applications for a wide variety of platforms. Hence, this enhancement hopes to help companies 
migrate their applications from .Net Framework to .Net Core by developing an external provider
for the analyzer lsp to analyze .Net Framework applications.

### Goals

- Make an external provider for analyzer lsp that uses a pre-existing lsp such as Omnisharp Roslyn.
- Create a sample set of rules for example applications to show the usage of the external provider.

### Non-Goals

- Make the set of rules that can be used for migration from .Net Framework to .Net Core.

## Proposal

### User Stories [optional]

#### Story 1

#### Story 2

### Implementation Details/Notes/Constraints [optional]

### Security, Risks, and Mitigations

**Carefully think through the security implications for this change**

## Design Details

### Test Plan

**Note:** *Section not required until targeted at a release.*

### Upgrade / Downgrade Strategy

## Implementation History

## Drawbacks

## Alternatives

## Infrastructure Needed [optional]
