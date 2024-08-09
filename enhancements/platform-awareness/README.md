---
title: platform-awareness
authors:
  - "@mansam"
  - "@shawn-hurley"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2024-06-07
last-updated: 2024-06-07
status: implementable
---

# Platform Awareness

## Open Questions

1. What platforms, other than k8s-family platforms, need to be supported?
2. Are there common scenarios in which an application would need to be associated with more than one platform?
3. How can we best use the combination of platform and code analysis results to make recommendations about which
   migration or modernization strategy to use?

## Definitions

#### "Platform"

An application's deployment environment. For instance, a k8s cluster or
a VM hypervisor.

#### "Platform-aware application analysis"

Analysis of an application that is able to inspect the application's
deployment environment and identify issues that may prevent it from
being rehosted or replatformed cleanly.

#### "Platform analysis"

Analysis of a platform outside of the scope of a specific application,
and which may require more privileged access to the platform than an
application-scoped analysis. For instance, a platform analysis might
examine cluster-scoped resources, network or security assumptions,
or compute resources.

## Summary

Introduce the concept of "Platforms" into Konveyor, which are
representations of application environments such as Kubernetes clusters,
as top-level objects with their own inventory in the Hub. It should
be possible to assign tags, stakeholders, and other inventory controls
to platforms.

It should be possible to associate an application with the platform it is
deployed on, including any necessary credentials and coordinates to locate
the live instantiation of the application on the platform. It should be
possible to launch a platform-aware analysis from the analysis wizard
and select from related rule sets that are executed by platform-aware
analysis providers, which would analyse the application's context within
the platform and report incidents back to the Hub in the usual way.

Additionally, it should be possible to perform analysis of platforms
outside of the context of a specific application, the results of which
would be reported in a new interface for platform analysis results.

## Motivation

To fully support the Konveyor methodology, it is not enough to only
analyse the source and binaries that comprise an application. Source
analysis is well suited to identifying opportunities and issues for
modernization, but is insufficient for capturing all the details that
are relevant for rehosting or replatforming applications. In order to
best determine which migration strategy to use we need to consider the
environments in which applications are deployed and the assumptions
inherent in those environments. Although some details and assumptions
about the deployed environment may be captured in questionnaires, the
assessment is likely to be incomplete and it may well be the case that
the live instantiation of the application has drifted from what it is
expected to be "on paper".

Automated analysis of the deployment environment can tell us about the
application's resource requirements, its security and availability
characteristics, and other traits which may be present in the live
environment but not in source control or other documentation. Combining
platform-aware analysis with source analysis and questionnaire-driven
assessment would enable Konveyor to recommend a strategy
and migration target.

In addition to platform-aware analysis at the application level,
running analysis rule sets against platforms themselves may help to
identify general problems that would interfere with migration between
platforms. (e.g. what facts about my AWS environment might generally
interfere with migrating from EKS to ROSA, or from vSphere to Kubevirt).

### Goals

* The Hub should inventory platforms and make it possible to associate
  applications with specific platforms.
* It should be possible to run a platform-aware analysis of an application
  in addition to or alongside a code analysis, and it should be possible to
  create and select analysis rulesets for platforms.
* It should be possible to run analysis of a platform with analysis rule sets
  targeting migration to other platforms.

### Non-Goals

* Asset generation (details revealed by analysis may facilitate this, but it's a separate topic)
* Automatic migration

## Proposal

### Personas / Actors

#### Administrator
The administrator for the tool that has access to some application-wide configuration parameters that other users can consume but not change or browse.

#### Architect
A technical lead for the migration project that can create and modify applications and information related to them.

#### Migrator
A developer that should be allowed to run analysis, but not to assess, create or modify applications in the portfolio.

### User Stories

#### Story 1

_As an Administrator, I want to be able to manage Platforms in the Konveyor inventory._

#### Story 2

_As an Architect, I want to be able to create a Deployment to associate an Application with a Platform, which includes
details such as credentials and coordinates to the live instantiation of the application within the platform._

#### Story 3

_As an Architect or Migrator, I want to be able to view Deployments._

#### Story 4

_As a Migrator, I want to be able to select and run a platform-aware analysis for an Application._

#### Story 5

_As a Migrator, I want to be able to select and run a combined code and platform-aware analysis for an Application._

#### Story 6

_As an Architect, I want to be able to select and run an analysis of a Platform._

#### Story 7

_As an Architect, I want to be able to create and manage analysis rulesets for Platforms._

### Implementation Details

#### Hub changes

Two new resources need to be added to the Hub inventory: Platforms, and Deployments.

* <b>Platforms</b>: As a concept in Konveyor, Platforms correspond to clusters, hypervisors, or other runtime environments where applications may be deployed.
  The Platform resource needs to capture the type of environment (EKS, OpenShift, vSphere, etc), coordinates to the environment (URL, AZ, etc),
  and necessary credentials (via Identity relationship). Platforms will need to be exposed with their own management view in the UI.
  Hypothetical resource below for discussion purposes.

```go
type Platform struct {
  ID           uint
  Name         string
  Kind         string
  URL          string // or other yet-to-be-determined means of identifying a provider
  Identity     *Ref
  Deployments  []Ref
  Tags         []Ref
  ...
}
```

* <b>Deployments</b> are necessary to fully capture the relationship between Applications and Platforms. This relationship would capture details
  such as application-scope credentials and coordinates within the cluster such as namespaces.
  Hypothetical resource below for discussion purposes.

```go
type Deployment struct {
  Platform    Ref
  Application Ref
  Identity    Ref
  Locator     string // namespace or other coordinates
}
```

Additionally, new credential types need to be added to provide access to Platforms. An example might
be a ServiceAccount credential type with the SA's name, token, and namespace.

The `Analysis` and `Task` resources will need to be extended to accept a Platform as a subject.

If we assume that a platform-aware analysis is run as part of a source/binary analysis, then no changes need to be
made to the existing application analysis API; the platform-aware portion of the analysis would be performed by
the relevant providers as part of a normal multi-provider analysis, and the results would be reported in the usual way.
This is the most straight-forward approach, but since it would share a single analysis report re-running one half would
necessitate re-running the other. Platform analysis, on the other hand, will require some refactoring of existing analysis
infrastructure and new API endpoints in order to make Platforms a valid analysis subject.


#### UX needs

* An inventory page for managing platforms.
* An inventory page for managing deployments.
* A means of selecting a platform-aware application analysis from the wizard.
* A wizard for performing platform analysis.
* An interface for viewing platform analysis results

#### Analysis providers

Each Platform needs to be supported by an analysis provider and rule sets for that provider.
The value of this feature is in the modernization-and-migration domain knowledge encoded in
the rule sets.

Of particular importance to this feature is a provider that is capable of interacting with
Kubernetes clusters and derivatives. A substantial amount of work toward such a provider has
been done in the [konveyor-ecosystem/k8s-provider](https://github.com/konveyor-ecosystem/k8s-provider)
repository which provides a foundation to continue building on. The provider will need to be able
to consume a variety of credentials (IAM credentials for EKS, ServiceAccount tokens, etc) and generate
the necessary Kubeconfig to interact with the cluster.



### Security, Risks, and Mitigations

The Hub needs to record credentials for Platforms and those credentials need to be passed
to analysis providers so that the Applications on those Platforms can be analysed. The
Hub already records credentials for remote systems and passes them to analysis providers,
but in this case the remote systems are active production environments which raises the
stakes significantly.

## Design Details

### Test Plan

### Upgrade / Downgrade Strategy

## Implementation History

-

## Infrastructure Needed [optional]

Instances of the supported platforms need to be available for testing.