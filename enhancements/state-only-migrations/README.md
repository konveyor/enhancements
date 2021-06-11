---
title: state-only-migrations
authors:
  - "@pranavgaikwad"
reviewers:
  - "@shawn_hurley"
  - "@alaypatel07"
approvers:
  - "@shawn_hurley"
  - "@alaypatel07"
  - "@djwhatle"
creation-date: 2021-06-10
last-updated: 2021-06-25
status: implementable
see-also:
  - "../crane-2.0/state-transfer/dvm-crane-feature-parity/README.md"  
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# State Only Migrations

State Only Migration is a migration of stateful resources of an application. It will facilitate migrating application state leaving stateless components of the application be migrated using an external mechanism such as OpenShift GitOps, Pipelines, etc. This document defines _State Only Migration_ in the context of MTC, and proposes API changes for implementing the additional features needed.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] User-facing documentation is created

## Summary

The applications running on Kubernetes typically store their state in _Persistent Volumes_. Additionally, some applications may use Kubernetes resources such as Secrets, Configmaps to persist application specific data which would qualify as state. State Only Migration will define a new migration workflow using _Stage_ and _Final_ migrations as building blocks to allow users migrate Persistent Volumes and a subset of Kubernetes resources. It will offer all the PV migration features currently offered by DVM with some additional features specifically aimed at making state transfer easier for users. 

## Motivation

With more and more OpenShift users leveraging GitOps workflows for continuous application delivery, we envision that the need for migrating stateless components of apps will diminish over time. The users will likely use their GitOps/Pipeline solution to redeploy the appication manifests to the target clusters. However, the persistent data created on the source cluster remains an open problem. MTC already handles PV migration pretty well and we believe the addition of State Only Migration to its capabilities will place MTC at the forefront of this problem domain. 

### Goals

- Define _State Only Migration_

- Define additional features needed for State Transfer and propose MTC API changes

- Define additional features needed in Final migration for selecting a subset of resources and propose API changes

- Introduce the idea of integrating Crane 2 state transfer library in DVM

### Non-Goals

- Propose controller changes in DVM

> This document only contains API changes as exposed by MTC. The actual implementation may be handled in crane-lib. Please see [this enhancement](../crane-2.0/state-transfer/dvm-crane-feature-parity/README.md) for implementation details of these features in crane-lib.

## Proposal

State Only Migration will be a mix of _Stage_ and _Final_ migrations. the users will perform _Stage_ migration to migrate PV data to the target cluster. This can be done as many times as needed. Finally, the users will perform _Final_ migration to migrate a subset of Kubernetes resources they qualified as state. The _Final_ migration can only be performed once. The individual workflows will be inline with MTC's current behavior.

It is assumed that the most Kubernetes resources will be migrated externally. There are two possible scenarios in this case - either the PVC objects will not be migrated externally or the PVC objects will be migrated externally. In the former case, MTC already has the ability to provision new PVCs and no additional change is needed. In the latter case, MTC cannot migrate to PVCs which are already provisioned in the target cluster and may have a different name than the source namespace. We will modify the DVM API to allow providing an optional target PVC name for each source PVC. Furthermore, the PVCs can also be present in different namespace than the source. This is already handled in DVM. The combination of namespace and name mapping will facilitate State Only Migration. This also enables migrations within the same cluster in that users can leverage DVM where they wish to change the underlying storage.

In the _Final_ migration, MTC assumes that the users will be migrating the entire source namespace. In State Migration, the users will be interested in selecting a subset of resources from the namespaces. The _Final_ migration will be updated to allow passing a list of GVKs to select from the source namespaces along with an optional label selector which will act as an additional filter for the included resources. Both of these features are offered by Velero. The _Final_ migration will simply propagate them as additional Backup options to Velero.

In an effort to eventually integrate Scribe in DVM, we will also begin replacing some of the internals in DVM with Crane 2 state transfer library. Hence, it is possible that the actual implementation of the API changes in this doc will take place in Crane 2 library. They will be covered in their own docs and referenced here.


### Implementation Details/Notes/Constraints [optional]

#### Selecting PVCs

The users will be able to select PVCs to be included in the migration from the UI. This will be achieved by setting `PvSkipAction` on any PVCs excluded from the migration. This is already possible in our current API and does not need any change. Most changes around this will be addressed in the UI. As @sseago suggested, this will require some minor changes in existing MigPlan validations that prevent attached PVCs be skipped from the migration. `PvSkipAction` currently only applies to PVCs which are not connected to any of the applications in source namespaces.

#### Mapping PVC names

We will accept an optional `targetName` for each PVC object. Since the primary user interaction in MTC happens through MigPlan, the `targetName` field will be introduced in MigPlan and DVM resources both. MigPlan will simply propagate the values to DVM:

_[migplan_types.go](https://github.com/konveyor/mig-controller/blob/ee9670d4054b76f9b63fbb12c2395302a14d0d4c/pkg/apis/migration/v1alpha1/migplan_types.go#L758-L763)_


```golang
// PVC
type PVC struct {
	[...]
	TargetName   string          `json:"targetName,omitempty"`
}
```

_[directvolumemigration_types.go](https://github.com/konveyor/mig-controller/blob/ee9670d4054b76f9b63fbb12c2395302a14d0d4c/pkg/apis/migration/v1alpha1/directvolumemigration_types.go#L27-L33)_

```golang
type PVCToMigrate struct {
	[...]	
	TargetNamespace  string     `json:"targetNamespace,omitempty"`
	TargetName       string     `json:"targetName,omitempty"`
	[...]
}
```

#### Resource list

In the _Final_ migration, the users will be able to provide an optional list of resources to be included in the migration. It will be converse of MTC's `EXCLUDED_RESOURCES` in that the the only resources present in the inclusion list will be migrated. The inclusion list will be accepted through MigPlan:

_[migplan_types.go](https://github.com/konveyor/mig-controller/blob/ee9670d4054b76f9b63fbb12c2395302a14d0d4c/pkg/apis/migration/v1alpha1/migplan_types.go#L65-L93)_

```golang
type MigPlanSpec struct {
	[...]

	IncludedResources []*metav1.GroupResource `json:"includedResources,omitempty"`
}
```

The resources present in this list will be propagated to [Velero Backup](https://github.com/vmware-tanzu/velero/blob/c230e9ca10f30a4c0f833af52856de0addc06919/pkg/apis/velero/v1/backup.go#L47). Velero expects the list of resources as a slice of strings. We will be proposing a change in the Velero API to make it a structured type instead. Until the upstream change is accepted, we will continue to convert the list to a slice of strings in MTC. 

#### Label selector

Optionally, the users will be able to provide a label selector to further filter the included resources. The label selector will also be accepted through MigPlan:

_[migplan_types.go](https://github.com/konveyor/mig-controller/blob/ee9670d4054b76f9b63fbb12c2395302a14d0d4c/pkg/apis/migration/v1alpha1/migplan_types.go#L65-L93)_

```golang
type MigPlanSpec struct {
	[...]

	LabelSelector *metav1.LabelSelector `json:"labelSelector,omitempty"`
}
```

It will be propagated to [Velero Backup](https://github.com/vmware-tanzu/velero/blob/c230e9ca10f30a4c0f833af52856de0addc06919/pkg/apis/velero/v1/backup.go#L60)

> Note: The complete resource inclusion hierarchy is explained in detail [in this section](#final-migration-compatibility).

The validations for both of the Backup options above exist in Velero. They will be copied to MTC in MigPlan validations for early detection of errors that may arise due to misconfigured options.

### Risks and Mitigations

#### Migrating into an existing namespace

If a namespace was created as a result of GitOps/Pipeline deployment in the target cluster, it is possible that the UID range of the namespace in the target cluster differ from that of the source. In this case, migrating Persistent Volumes from the source to the target cluster will result in a mismatch in file ownership and the PVs won't be usable as-is. There are various ways to solve this problem. One of the safer solutions is to patch the namespace before migrating PVs such that the annotations are equal to that of source ns. If there are any workloads running in that ns already, the workloads need to be queisced and unquisced once post update. 

> Please note that this is an entire new problem domain which requires its own enhancement with a holistic solution. For the purpose of State Only Migrations, we wish to explore various ways of solving the problem manually and provide a document describing them. 

## Design Details

### Test Plan

**Note:** *Section not required until targeted at a release.*
1. For migration of PVCs, different combinations of namespace/name changes will be tested:

  - Migrating into target namespaces with no pre-provisioned PVCs
  - Migrating into target namespaces with all pre-provisioned PVCs
  - Migrating into target namespaces with some of the PVCs pre-provisioned

2. Possibility of same cluster migrations will be evaluated including migrating to a different ns and migrating to a different PV in the same namespace

3. For migration of a subset of Kubernetes resources in _Final_ migration, different combinations mentioned in the [compatibility matrix](#final-migration-compatibility) will be tested
### Upgrade / Downgrade Strategy

#### Stage migration compatibility

Since the PV mapping will be handled in DVM API, no change is needed in the _Stage_ migration or in any of its itinerary phases. The API changes in MigPlan and DVM both introduce an optional `targetName` variable each. 

#### Final migration compatibility

Since we will be adding optional inclusion list and label selector to the MigPlan which never existed in the API before, we will ensure that the MigPlans created in the previous versions are not adversely affected. Also, the proposed inclusion list differs from the existing `EXCLUDED_RESOURCES` list in that the exlusion list is a global configuration, while the inclusion list is at a MigPlan level. It is important to consider how different configurations will affect each other. Following table lists all possible combinations of the old and the new configurations:

| No 	|   Resources in source ns   	| Global exclusion list 	| MigPlan inclusion list 	| Label Selector 	|           Outcome          	|
|:--:	|:--------------------------:	|:---------------------:	|:----------------------:	|:--------------:	|:--------------------------:	|
|  1 	| A0  A1   B0   B1   C0   C1 	|           -           	|            -           	|        -       	| A0  A1   B0   B1   C0   C1 	|
|  2 	| A0  A1   B0   B1   C0   C1 	|           -           	|            -           	|        1       	|        A1   B1   C1        	|
|  3 	| A0  A1   B0   B1   C0   C1 	|           -           	|          A, B          	|        1       	|           A1   B1          	|
|  4 	| A0  A1   B0   B1   C0   C1 	|           -           	|            C           	|        -       	|           C0   C1          	|
|  5 	| A0  A1   B0   B1   C0   C1 	|           A           	|            -           	|        -       	|      B0   B1   C0   C1     	|
|  6 	| A0  A1   B0   B1   C0   C1 	|           A           	|          A, C          	|        -       	|           C0   C1          	|
|  7 	| A0  A1   B0   B1   C0   C1 	|           A           	|            -           	|        1       	|           B1   C1          	|
|  8 	| A0  A1   B0   B1   C0   C1 	|           A           	|          A, C          	|        1       	|             C1             	|

Both `includedResources` and `labelSelector` are optional variables in MigPlan spec. On the existing MigPlans, these will be defaulted to empty values and Final migration will continue to include all resources in the migration (See case 1 in the table). Downgrades are unaffected as well. The old API will simply ignore the extra options. 

