---
title: kubernetes-provider-module
authors:
  - "@mansam"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2024-02-05
last-updated: 2024-03-04
status: implementable
---

# Kubernetes provider module

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

Create a Kubernetes provider for the Konveyor analyzer to perform in-cluster analysis of live applications and their deployment environments. This will include collection and analysis
of a configurable list of namespace-scoped resources. Discovery and analysis of cluster-scoped resources and image manifests could be explored as a follow-up.
The provider will conform to the Konveyor analyzer provider gRPC interface.

## Motivation

Adding a Kubernetes provider to the Konveyor analyzer extends its utility beyond source/binary analysis
by enabling it to analyze an application's deployment environment. Analysis of the live application in
the cluster can tell us things about its resource requirements, availability characteristics, and cloud-provider-specific dependencies which can inform
the migration or modernization process. Live analysis can also be used on the destination cluster of a migrated application to validate that a
migration was successful and that the new deployment is adhering to best practices.

### Goals

* Enable users to perform live analysis of applications hosted in a Kubernetes cluster.
* Enable users to write Kubernetes analysis rules in a concise, expressive, and easy-to-read format.

### Non-Goals

* Modification of live resources or automatic enforcement of rules.

## Proposal

The Kubernetes provider must be able to retrieve resources out of a live cluster, evaluate rules against those
resources, and return the evaluation results with all the details necessary to act on them (resource coordinates, apiVersions, guidance, etc).
It needs to be easy to write rules so that domain experts are more willing to contribute the time and effort to do so,
and the rules language needs to be both expressive and familiar.

### User Stories

#### Story 1

As an application migrator I want to learn about issues with my live application that need to be addressed in order
to successfully migrate or modernize it.

#### Story 2

As a domain expert I want to write suites of analysis rules to furnish migrators with the
details they need to successfully migrate or modernize their applications.

### Implementation Details

The Kubernetes provider will use Rego as its rules language, and will use the Open Policy Agent SDK
as its rules evaluation engine. Rego is expressive, easy to read, and well suited to analysis of structured data such as Kubernetes resources.
Leveraging OPA/Rego allows us to focus on implementing the specifics of cluster analysis and allows rules
authors to write simple rules that are oriented to the data rather than the form in which it is stored (json, yaml, etc).

The provider will initially surface two rule capabilities: One that is oriented towards ease-of-authorship in exchange for limited expressiveness (`rego_expr`), and one that permits freeform authoring of complex Rego modules (`rego_module`).
Ideally, the simpler rule format will strike a balance that enables most rules to be written in it. To facilitate the use of the simpler rule capability,
we will provide a built-in inventory of Rego rules that can be referred to by user-provided rules.

#### Example Rules

```yaml
ruleID: replica-count
effort: 1
category: optional
message: "Low replica count"
when:
  k8s-resource.rego_module:
    module: |
      package policy
      import data.lib.konveyor
      import future.keywords

      incidents[msg] {
      	some item in data.lib.konveyor.deployments
        item.spec.replicas < 2
      	msg := {
            "apiVersion": item.apiVersion,
      		"namespace": item.metadata.namespace,
      		"kind": item.kind,
      		"name": item.metadata.name,
      	}
      }
---
ruleID: replica-count-simple
effort: 1
category: optional
message: "Low replica count"
when:
  k8s-resource.rego_expr:
    collection: deployments
    expression: item.spec.replicas < 2
---
ruleID: unmounted-claims
effort: 1
category: optional
message:
when:
  k8s-resource.rego_module:
    module: |
      package policy
      import data.lib.konveyor
      import future.keywords

      pvcs[pvc] {
        some list in input.namespaces[_]
        some item in list.items
        item.kind == "PersistentVolumeClaim"
        pvc := item
      }
      
      mounted_claims[claim] {
        some pod in data.lib.konveyor.pods
        some volume in pod.spec.volumes
        claim := volume.persistentVolumeClaim.claimName
      }

      incidents[msg] {
        some pvc in pvcs
        claim := pvc.metadata.name
        not mounted_claims[claim]
        msg := {
          "apiVersion": pvc.apiVersion,
          "namespace": pvc.metadata.namespace,
          "kind": pvc.kind,
          "name": pvc.metadata.name,
        }
      }
```

Following after the initial two capabilities should be a third to inspect image manifests that belong to the application
being analyzed. This can likely be implemented by delegating to [Skopeo](https://github.com/containers/skopeo) or [containers/image](https://github.com/containers/image) to
pull the manifests and OPA can be used as the policy engine.

### Security, Risks, and Mitigations

In order to perform live cluster analysis, the provider will need to be furnished with cluster credentials
that have read access to the relevant resources. The provider must tolerate limited permissions to the extent possible.
`list` and `get` permission on the resource types to be inspected should be sufficient.

## Design Details

### Test Plan

Aside from the provider's cluster connection and resource gathering, all the provider's behavior could be tested
with mock Kubernetes resources and simplified test rulesets that exercise each capability. To permit easy testing of the
capabilities, and to make it possible for ruleset authors to test their rules, it needs to be possible to drop in an
alternative k8s client (or resource collection abstraction) that is loaded with known resources.

## Open Questions

Should image manifest analysis be a capability of this provider? A big benefit of doing it in this provider would be that
this provider is already able to discover what images are in use, and it would be able to analyze manifests in the context of the workloads using them.
However, it is unclear how complex accessing the image manifests will be.

## Implementation History

* 1/22/2024: Implementation started in https://github.com/konveyor-ecosystem/k8s-provider
