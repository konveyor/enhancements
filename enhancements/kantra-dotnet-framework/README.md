---
title: kantra-dotnet-support
authors:
  - "@djzager"
reviewers:
  - "@pranavgaikwad"
  - "@shawn-hurley"
  - "@eemcmullan"
approvers:
  - "@pranavgaikwad"
  - "@shawn-hurley"
  - "@eemcmullan"
creation-date: 2024-05-16
last-updated: 2024-05-16
status: implementable
see-also:
  - "../dotnet-provider/README.md"  
---

# Kantra .NET Support

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

For these open questions, we lack the requisite knowledge/experience with
Windows containers to answer these questions definitively before accepting the
enhancement. This enhancement should be updated when these are answered
definitely one way or another, especially if they change the implementation
details.

1. Currently, Kantra assumes that it can create a single network via `podman`
   or `docker` and use that for all containers it starts. Can the network of
   Linux and Windows containers be shared?
1. Currently, Kantra creates volume(s) to be shared with each of the providers
   it starts. Can volumes be shared between Linux and Windows containers?
1. How does one start Windows and Linux containers on the same host?

## Summary

It is essential for the success of the Konveyor project that we develop a
method for analyzing Windows-only .NET applications in a meaningful and user
friendly way. At the time of this enhancement, analyzing Windows-only
applications (like
[nerd-dinner](https://github.com/konveyor/analyzer-lsp/tree/ecb5c4a6c7a881a5694d0f655c2070376e9c3d67/external-providers/dotnet-external-provider/examples/nerd-dinner))
requires a significant amount of pre-work on the part of the user in order to
perform the analysis via a Windows binary. This enhancement improves on that
experience by 1) providing a Windows container images for the provider and 2)
enabling Kantra to manage the Windows container on a Windows host to perform
the analysis. 

## Motivation

Much like in [External Provider for Analyzer LSP for .NET/C#
Applications](../dotnet-provider/README.md), this is important to us because
many enterprise users have large catalogs of Windows-only .NET Framework
applications. Providing rules-based analysis for these applications would be of
tremendous benefit to those users. Making this analysis more accessible and
user friendly is the motivation for this enhancement.

### Goals

* Propose the publishing of Windows containers for the essential command-line
  analysis components.
* Propose modifications to Kantra to support running Windows containers to
  perform the analysis.

### Non-Goals

* Discuss in-cluster (ie. via the hub) analysis of .NET Framework applications
* Discuss ruleset(s) for .NET Framework applications.

## Proposal

Introduce a new .NET Provider for Kantra. We will do this by providing
dotnet-external-provider, analyzer-lsp, and kantra as windows container images.
Using [alizer](https://github.com/devfile/alizer) to determine the language and
frameworks used, we can differentiate between .NET projects that can be
analyzed in a Linux container versus .NET Framework projects that must be
analyzed in a Windows container. With this knowledge, when we encounter a
Windows-only .NET Project, we will break away from the normal `analyze`
subcommand flow and analyze the application using the dotnet-external-provider
and kantra Windows containers. Once we execute the analysis, we can retrieve
and output the results.

### User Stories

#### Analyze Cross-Platform .NET Project (.NET 5 or greater)

This story is no different technically than a user wanting to analyze a `go` or
`java` project today. Kantra would need to start a Linux container for the
dotnet-external-provider, construct the appropriate provider configuration, and
perform the analysis. The work required here would simply be recognizing
projects in this category and constructing the provider settings correctly.

#### Analyze Windows-only .NET Project (.NET Framework 4.5 through .NET Framework 4.8)

This story is **fundamentally** different than how Kantra runs today.
First, this mode requires `docker` to be installed + ready to run Windows containers.
When we detect we are looking at a Windows-only .NET Project, we'll run the remainder
of the containerized workloads using Windows containers.

### Implementation Details

## Drawbacks

Attempting to analyze Windows-only projects using Windows containers alongside
Linux containers sharing a network and volumes could drastically increase the
complexity of the `kantra analyze` command if not totally undo some of the
basic assumptions of the analysis procedure.

## Alternatives

More detail can be found in the [discussion doc](./discussion.md) created to
discuss some of the alternative means of reaching the goal.
