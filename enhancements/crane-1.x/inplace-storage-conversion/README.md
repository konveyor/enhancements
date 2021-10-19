---
title: inplace-storage-class-conversion
authors:
  - "@pranavgaikwad"
reviewers:
  - "@sseago"
  - "@alaypatel07"
  - "@shawn-hurley"
approvers:
  - "@sseago"
  - "@alaypatel07"
  - "@shawn-hurley"
creation-date: 2021-09-27
last-updated: 2021-09-28
status: provisional
see-also:
  - N/A
replaces:
  - N/A
superseded-by:
  - N/A
---

# Inplace Storage Class Conversion

MTC 1.6 introduces state migration which allows migrating PVCs within the same cluster. This feature can be leveraged to migrate PVCs from one storage class to another. When migrating PVCs using state transfer within the same cluster, PVCs can either be migrated to a different namespace or to the same namespace. In the former case, PVC names are preserved and all the workloads will be migrated to a different namespace. The migrated workloads will continue running as the PVC names will be preserved and simply the namespace will change. However, in the latter case, PVC names cannot be preserved as two PVCs with same names cannot co-exist in a namespace. As a result, the workloads will continue using old PVCs and the users will need to manually update the PVC references in their workloads to use the new PVCs. This enhancement proposes possible solutions to automatically update the PVC references so that the users don't require to take manual steps post migration.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]
 
## Summary

When using state migration to migrate PVCs within the same namespace, the target PVCs must have different names. As a result, the users will need to manually update their workloads to point to the new PVCs after a migration. The migration controller can automate this process. It will improve the experience especially for large scale migrations. This enhancement proposes two possible solutions to solve this problem.

## Motivation

In MTC 1.6, the users can leverage _State Migration_ to migrate their PVCs to a different storage provider. If the target PVCs are to be created in the same namespace as the source, the users are required to take manual steps to update their workloads to point to the new PVCs post a migration. For large scale migrations, this would be a cumbersome process. Also, during this time, the workloads have to be quisced so that new data is not written to the old PVCs. MTC can handle the updates to the workloads/PVCs automatically after PV migration minimizing downtime of workloads and the possibility of human errors.

### Goals

> Automate the process of re-attaching PVCs to workloads post a storage class conversion as doing it manually is not practical for bulk conversions

> Ensure that the data is not lost during the re-attaching process

## Proposal
During a state migration, the _MigPlan_ controller will determine whether any of the PVCs are being migrated within the same cluster and in the same namespace. If that's the case, it will raise a _Warning_ notifying the user about the problem. The _MigMigration_ controller will use this information to enable the automatic PVC update procedure. The default behavior would be to update the PVCs if such case arises. The users can disable the automatic updates either through a _Spec_ field, an annotation or a _MigrationController_ flag (to be decided).

The _MigMigration_ controller will choose any one of the below approaches to restore PVs:

### Approach 1: Update the user workloads to point to migrated PVCs


1. Quiesce workloads

2. Migrate data using DVM

3. Find source PVC references in the workloads and update them to new migrated PVCs instead

4. Un-quiesce the workloads

### User Stories [optional]

#### Story 1

As a user, I would like to use _State Migration_ to migrate PVCs in the same namespace so that I can leverage MTC's storage class conversion capabilities to move my PVCs to a different storage provider.   

## Design Details

### Test Plan

Storage Conversion from Portworx to OCS 4 will be tested in following scenarios:

- If workloads are associated with PVCs, PVCs will be migrated to new storage class and the workloads are automatically updated to use the new PVCs

- If workloads are not associated with PVCs, PVCs will be migrated to new storage class and the message with names of PVCs will be added to Migration to indicate the new PVCs
### Upgrade / Downgrade Strategy

The only API change needed is the switch that disables the automatic update of PVC references. Currently, I have not put much thought on where to keep it. It could be a _Spec_ field on _MigMigration_ CR or an annotation or a _MigrationController_ variable passed as environment variable through the configmap. 

If it were to be a _Spec_ field on _MigMigration_, it will be an additive _optional_ field added to _MigMigration_ API: 

```golang
type MigMigrationSpec struct {
	[...]
	
	AutoUpdatePVCRefs bool `json:"autoupdatepvcrefs,omitempty"`
}
```

The default behavior is to turn the functionality OFF. The UI will set this value when creating MigMigrations whenever PVCs are migrated within the same namespace using _State Transfer_. 

When upgrading from 1.7- versions, to 1.7 versions this field will be absent on all _MigMigration_ resources. However, _MigMigration_ resources are supposed to go to completion. The users ideally shouldn't have _Running_ migrations during an upgrade. But there could be blocked migrations waiting on perhaps a plan to become Ready. All such blocked migrations will automatically enable this functionality upon upgrade without the user noticing.

When downgrading from 1.7+ versions to an older version, this feature will simply be absent. The extra _spec_ field present in _MigMigration_ CRs wont make any difference on the behavior of the controller.

Alternatively, an annotation on _MigMigration_ CR would indicate the controller to auto update the PVC references:

```golang
apiVersion: migration.openshift.io/v1alpha1
kind: MigMigration
metadata:
  annotations:
    migration.openshift.io/auto-update-pvc-refs: "true"
```

Any existing resources from previous versions would not have the above annotation set. Thus, the functionality will be turned off for _MigMigration_ resources created in older versions of MTC.

### Alternatives considered

### Use Indirect Volume Migration

1. Quiesce the workloads

2. Backup source PVCs to replication repository and backup data using Restic

3. Delete source PVCs

4. Restore PVCs in the same namespace and restore data using Restic

### Update PVs inplace without changing the user workloads

1. Quiesce the workloads

2. Migrate PVCs using DVM

3. Update PVC references:

    a. Find PV object for migrated PVC
    b. Check if the PV Retain Policy is set to _Reclaim_. If not, update and set it to _Reclaim_.
    c. Delete _ClaimRef_ on migrated PV
    d. Delete source PVC definition
    e. Copy migrated PVC definition, change its name to source PVC name, and create it. 
    f. Add _ClaimRef_ on the PV that points to the newly create PVC object
    g. Delete migrated PVC definition

4. Un-quiesce workloads

The sub-stages _[a - g]_ in _Step 3_ above are critical and modify the original PV/PVC objects. Therefore, any failure happening during the execution will need to be handled carefully such that the original data is not lost. 

Stages _[a - c]_ update the migrated PVC objects. Any failure happened in any one of those stages will abort the rest of the stages and the reconciler will move on to the next PVC.

A failure in _[d]_ will still result in aborting the rest of stages. However, a failure here indicates that the original PVC object still exists and the workloads will continue to run without issues. 

Stages _[e - g]_ happen after the original PVC definitions are deleted. If any of these stages fail, the controller will attepmt to restore the original PVC definition. An added retry mechanism can be added to ensure that the original PVC is restored correctly.

### Pros and Cons

#### Updating user workloads

_Approach 1_ will only work on "known" workload types. It wont work on workloads managed through Custom Resources. The other two approaches work on all kinds of workloads as they do not need to change the original user workloads. They only deal with the PVC, PV and the PV data.

#### Complexity and failure handling

_Approach 1_ is easier to implement and doesn't touch original PVC object. In case of failures, the users can simply revert back the PVC references to the original ones.

_Approach 2_ is the easiest to implement (since we already have all the needed logic in the controller), but it requires deleting the original PVC object. Any failure during a restore would result in additional downtime for user workloads. The original data can still be recovered from the replication repository if needed.

_Approach 3_ is complex to implement and requires deleting the original PVC object potentially leaving the environment in a bad state. Since this approach uses direct volume migration 

#### Additional storage required

_Approach 1 and 3_ require additional storage of equal capacity created in the namespace during the time of transfer. Therefore, during the migration, the storage needs of the namespace would be double their original capacity. 

_Approach 2_ uses Restic and thus, a replication repository for intermediate storage. Therefore, during the migration, the source namespace wouldn't require additional storage. The storage needs are delegated to the replication repository instead.

For the above reasons, alternative approaches were discarded.