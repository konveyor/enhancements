---
title: run-krm-functions-from-crane-cli
authors:
  - "@amundra"
reviewers:
  - "@jmatthew" 
  - "@dzager"
  - "@shurley"
  - "@ernelson"
  - "@dymurray"
  - "@spampatt"
approvers:
  - "@jmatthew" 
  - "@dzager"
  - "@shurley"
  - "@ernelson"
  - "@dymurray"
  - "@spampatt"
creation-date: 2022-06-21
last-updated: 2022-06-21
status: implementable
see-also:
  - "N/A"  
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Run KRM functions from crane CLI

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions
- Do we really need the capability to execute KRM functions integral to the Crane CLI?

## Summary
This enhancement discusses a possible solution to run KRM functions from the crane CLI. It proposes to add the crane CLI subcommand that can execute a KRM function.

On Crane, adding the subcommand to execute functions will ensure that -
- It can execute a containerized function against the given input resources and store the transformed output to a destination directory

## Motivation
At present, we can not utilize the KRM functions in the crane. To leverage the benefits of KRM functions and aligning with the community standards, we are proposing to add the capability in the crane CLI to execute a function. 

### Goals

- Propose solution to handle execution of KRM functions in crane.
- Implement the proposed solution.
 

## Proposal
- We can add a subcommand in the crane cli that can run KRM functions with the arguments. The underlying functionality can be built on top of [kyaml](https://pkg.go.dev/sigs.k8s.io/kustomize/kyaml) package that provides libraries to run containerized function images.

### User Stories

#### Story 1
As a crane cli user, I would like to run a single KRM function with crane subcommand using function image as an argument and other command-line flags, such that the function is applied against the input resources and produces the transformed output. 

#### Story 2
As a crane user, I would like to run multiple functions in a predefined manner such that they are applied to input resources and produces a transformed output.

## Alternatives
- Alternative to creating an integrated solution to crane cli, we can leverage the already built solutions that provide similar functionality like [KPT](https://github.com/GoogleContainerTools/kpt) and [Kustomize](https://github.com/kubernetes-sigs/kustomize).
