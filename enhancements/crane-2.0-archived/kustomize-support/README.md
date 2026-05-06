---
title: crane-kustomizate-support
authors:
  - "@djzager"
reviewers:
  - "@shawn-hurley"
  - "@sseago"
  - "@alaypatel07"
approvers:
  - "@shawn-hurley"
creation-date: 2021-12-23
last-updated: 2021-12-23
status: rejected
see-also:
  - "N/A" 
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Crane Kustomization Support

This document covers support for [Kustomize](https://kustomize.io/) directly in
`crane`.

## Release Signoff Checklist

This enhancement has been **REJECTED**. The functionality proposed in this
enhancement provides no more than `kustomize init --autodetect`. While there was
some discussion about handling the JSON patches from transform to construct base
+ overlays via Kustomize, it was decided that this should be researched and
discussed in more detail in a follow-up enhancement.

## Open Questions

- Should the `crane export`ed manifests be the base **OR** should we construct
    a completely stripped base with the original and transform+apply manifests
    defined as overlays **OR** (easiest) allow the transform+apply manifests to
    be the base?

## Summary

[Kustomize](https://kustomize.io/) is a template-free Kubernetes native
configuration management solution embedded in `kubectl` (and `oc`) as
`apply --kustomization` that allows user to define
["variants"](https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#variant)
as
["overlays"](https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#overlay)
against a
["base"](https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#base)
kustomization without modifying the original (aka "base") manifests.
It's important that Crane help users improve their situation when migrating
their applications. This enhancement proposes a new feature to the `apply`
subcommand that lays makes the output of `crane apply` Kustomize ready.
Afterwards, users can easily leverage the manifests via `kubectl apply
--kustomization`, use the `kustomize` binary directly to edit|add|expand the
supplied Kustomization(s), or onboard the application into a CI/CD system like
[ArgoCD with support for Kustomize](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/).

## Motivation

Make it as easy as possible for users to improve the way their application's are
managed and configured. Additionally, Kustomize and `crane` exist in very
similar problem spaces and it would be wise to lean on work already invested in
Kustomize as much as possible, a couple examples include:

- [Namespace overrides](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/namespace/)
- [Patches](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/)
- [Replacements](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/replacements/)
- [Change replicas](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/replicas/)

### Goals

- Allow user to further modify the results of `crane apply` with `kustomize`
    directly (ie. `kustomize edit namespace foobar`).
- Allow user to apply manifests to cluster with `kubectl apply --kustomize
    ${CRANE_APPLY_OUTPUT_DIR}`

### Non-Goals

- Support CI/CD onboarding from crane directly. The purpose of providing
    Kustomize support is to assist the user whichever route they choose to take.

## Proposal

As the resources from `crane export` are iterated over:

1. The location of the `kustomization.yaml` will be computed using the output
   file path (ie. the same directory where the transformed manifest was
   written).
1. Create the `kustomization.yaml` if it doesn't exist **OR** load it from disk
   if found.
1. Add the manifest just written to the slice of resources in the Kustomization.
1. Write the `kustomization.yaml` to disk and close.

### Implementation Details/Notes/Constraints

This proposal is currently only additive, that it doesn't fundamentally change
the way the `export`, `transform`, and `apply` subcommands work together. If
crane wants to lean more into Kustomize features like patching, generators, etc.
it's possible the manifests will only be useful when using Kustomize (ie. kubectl
or kustomize).

### Security, Risks, and Mitigations

More thoughs need to be added here. What about secrets?

## Design Details

### Test Plan

Must add verification that the `kustomization.yaml` generated via `crane apply`
is valid for use with `kubectl apply --kustomization`.

### Upgrade / Downgrade Strategy

Once implemented, the output directory created||updated by `crane apply` will
include a `kustomization.yaml`. No special considerations for transformations
generated with older crane should be required.

## Alternatives

- All of this functionality could be implemented in a higher level via
    [Crane Runner](https://github.com/konveyor/crane-runner)
    ClusterTasks; one specific
    [ClusterTask](https://github.com/konveyor/crane-runner/blob/742883ce69861510ab80f1c6c253cc759d7091b1/manifests/clustertasks/kustomize-namespace.yaml)
    does this. Assuming even minimal engagement with this feature, it would be
    much easier to support in `crane apply` directly.
