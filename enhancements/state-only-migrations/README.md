---
title: state-only-migrations
authors:
  - "@pranavgaikwad"
reviewers:
  - "@djwhatle"
  - "@sseago"
approvers:
  - "@djwhatle"
  - "@sseago"
creation-date: 2021-06-10
last-updated: 2021-06-10
status: provisional
see-also:
  - "N/A"  
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# State Only Migrations

State Only Migration is a migration of stateful resources of an application. It will enable users to deploy their applications on the target clusters using an external mechanism such as GitOps and use MTC to migrate the stateful components alone. This document defines the scope of _State Only Migration_ in MTC and proposes solution to implement it. 

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]


## Summary

The applications running on Kubernetes typically store their state in _Persistent Volumes_. Additionally, some applications may also use Kubernetes resources to persist application specific data. State Only Migration will address both of these scenarios with some limitations on inclusion of Kubernetes resources. It will facilitate migration of _Persistent Volumes_ and Kubernetes resources together.

## Motivation

With more and more OpenShift users leveraging GitOps workflows for continuous application delivery, it is important for us to think about usage of MTC in an environment where the users will migrate their applications using a mix of MTC and their existing GitOps/Pipeline solutions. 

### Goals

- Allow selecting _Persistent Volumes_ to migrate from the source namespaces. 

- Allow mapping names of _Persistent Volumes_ from the source to target namespaces.

- Allow including a part of the Kubernetes resources in the migration using a list of GVKs and a label selector. 

### Non-Goals

- Allow incremental migration of Kubernetes resources.

## Proposal

A new itinerary 'StateTransfer' will be introduced in MigMigration controller. This itinerary will address the migration of Persistent Volumes and Kubernetes resources. The _Persistent Volume_ migration bits of it will be exactly same as _Stage_ itinerary in that the users will have an option of chosing either _Direct_ or _Indirect_ migration, _Direct_ being the     default choice. For the Kubernetes resources, the users can provide a list of GVKs and a custom label selector per Migration. Unlike _Stage_ itinerary, image resources will not be a part by default unless explicitely included by the user using the GVK inclusion list.
 
It is possible to simply change _Stage_ itinerary to include Kubernetes resources with some minor additions to PV migration flow. However, introducing new itinerary has some advantages. Since _Stage_ and _Final_ itineraries are not affected by the change, the existing MTC users can continue using them as they did before. Any new bugs will be localized to the new itinerary. A separate itinerary also makes it easier to itegrate Scribe in the future.

### User Stories [optional]

#### Story 1
As a user, I will be able to use MTC to migrate Persistent Volume data given that I already migrated rest of the resources using an OpenShift GitOps solution. 
#### Story 2
As a user, I will be able to selectively migrate a part of Kubernetes resources in the source namespaces.

### Implementation Details/Notes/Constraints [optional]

_StateTransfer_ itinerary will be a copy of the _Stage_ itinerary with image migration disabled by default unless otherwise enabled by including the image resources.
### Persistent Volume migration

The PV migration will work as it would in _Stage_ or _Final_ migration with some additional features specific to state only migration:

#### Selecting PVs

The users will be able to select PVs to be included in the migration in the UI. This will be achieved by setting `PvSkipAction` on any PVs excluded from the migration. This is already possible in our current API and does not need any change. Most changes around this will be addressed in the UI.

#### Mapping PV names

It is possible that when the users deployed their applications in the target cluster, the names of the resources changed due to usage of generateName or alike. In such scenarios, we need the ability to map a source PV to a target PV with a different name. To achieve that, we take an optional `targetName` for each PVC. The optional name will be provided by the user in the wizard and saved in the PV object on the MigPlan. It will be further propagated to the DVM CR:

_migplan_types.go_

```golang
// PVC
type PVC struct {
	[...]
	TargetName   string          `json:"targetName,omitempty"`
}
```
_directvolumemigration_types.go_

```golang
type PVCToMigrate struct {
	[...]	
	TargetNamespace  string     `json:"targetNamespace,omitempty"`
	TargetName       string     `json:"targetName,omitempty"`
	[...]
}
```
### Migrating Kubernetes resources

Apart from Persistent Volumes, _StateTransfer_ will also allow users to include a subset of Kubernetes resources from their source namespaces. 

The users will be able to provide a list of resources to include in the Velero Backup along with a label selector. The list of GVKs and the label selector will be used when creating a Velero Backup.

#### Resource inclusion/exclusion

By default, _StateTransfer_ will include following resources:

```yml
- serviceaccounts
- persistentvolumes
- persistentvolumeclaims
- pods
- secrets
- configmaps
```
The users will also be able to provide their own list of resource in addition to the above defaults. The list will be stored on _MigMigration_ CR:

_migmigration_types.go_

```golang
type MigMigrationSpec struct {
	[...]
	// Comma separated list of included Kubernetes resources in Velero backup
	IncludedResources string `json:"includedResources,omitempty"`
}
```

Currently, MTC uses a list of `EXCLUDED_RESOURCES` which is provided in the MigrationController CR. That list will not be respected in the _StateTransfer_. 

#### Resource list and label selector

State only migration does not aim to include all the resources from source namespaces in a migration. Instead, it will give the users a label they can add on their resources. 
The users will be expected to add the labels on the resources they want to migrate. With the list of resources and the label selector, all the resource kinds in the list matching the label selector will be included in the Velero Backup.

### Risks and Mitigations

What are the risks of this proposal and how do we mitigate. Think broadly. For
example, consider both security and how this will impact the larger OKD
ecosystem.

How will security be reviewed and by whom? How will UX be reviewed and by whom?

Consider including folks that also work outside your immediate sub-project.

## Design Details

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to...
  -  keep previous behavior?
  - make use of the enhancement?

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.
