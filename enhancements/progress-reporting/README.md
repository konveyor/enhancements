---
title: MTC Progress Reporting Improvements
authors:
  - "@pranavgaikwad"
  - "@JaydipGabani"
reviewers:
  - "@jortel"
  - "@djwhatle"
  - "@alaypatel07"
  - "@dymurray"
approvers:
  - "@jortel"
  - "@djwhatle"
  - "@alaypatel07"
  - "@dymurray"
creation-date: 2020-09-14
last-updated: 2020-09-16
status: implementable
see-also:
  - "N/A" 
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# MTC Progress Reporting Improvements

The goal of this document is to highlight user experience issues in the current implementation of progress reporting in MTC, and propose improvements in some of the areas for a better user experience. The driving goal of the effort is to ensure that the user has enough information in their hands to understand what exactly is happening in the background of an ongoing migration. 

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

With the proposed changes, the migration controller will expose useful data pertaining to the progress of ongoing migration, likely in the status field of _MigMigration_ CR. The MTC UI will be re-designed to incorporate the enhancements, possibly adopting a pipeline like view for showing progress. Additionally, the current migration phases will be divided into high level groups / steps of relevant phases. The new steps will abstract out some of the details of the migration from the end user. The existing phases will only be _hidden_ from the end user, and will still be available in the _MigMigration_ CR for debugging. 

## Motivation

The motivation behind the said changes stems from some of the user experience issues observed in MTC.

### How progress reporting looks like in MTC (Until 1.3)?

The Status field in the _MigMigration_ CR contains the information about the progress of ongoing migration. Upon creation of _MigMigration_ CR, the migration transitions from one _Phase_ to another until it reaches the _Completed_ phase. For a user, a _Phase_ simply means a one of the _Steps_ in the migration. 

Here is an example of _MigMigration_ status field during an ongoing migration:

```yml
  conditions:
  - category: Advisory
    lastTransitionTime: "2020-09-15T14:54:46Z"
    message: 'Step: 7/33'
    reason: InitialBackupCreated
    status: "True"
    type: Running
  - category: Required
    lastTransitionTime: "2020-09-15T14:53:15Z"
    message: The migration is ready.
    status: "True"
    type: Ready
  itenerary: Final
  observedDigest: 9834d071975562d5e2c2eb855bca6950711ded8a0e45af5307fa56cd0f5ba3c7
  phase: InitialBackupCreated
  startTimestamp: "2020-09-15T14:53:15Z"
```

The `message` field shows how many steps/phases have been completed so far in the migration. The total number of steps change based on whether it's a stage migration or a full migration. The MTC UI uses this field to derive progress in percentages and depicts it in the form a progress bar:

![MTC Progress Bar](./mtc-progress-bar.png)

### Issues with the current implementation

* In a full migration, there are over 30 steps/phases involved. The migration transitions from one phase to another until completion. All these phases are posted to the Status field of the _MigMigration_ CR as and when encountered. We cannot expect an end user to know the meaning of each of these phases. For instance, one of the migration phases is `EnsureCloudSecretPropagated`. While, from a debugging point of view, knowing that the migration is attempting to propagate cloud secret is an important piece of information, it is questionable how much value it adds to the end user experience. 

* Some phases in migration run for a longer period of time than other phases. For such long running phases, MTC currently does not have a way to inform the user of the progress of the ongoing phase itself. For instance, during an initial Backup, the migration controller enters `EnsureInitialBackup` phase where it creates a Velero Backup object and transitions to `InitialBackupCreated` phase upon successful creation of Backup. It then waits until the underlying Backup object has a Completed / PartiallyFailed / Failed condition. While the backup runs in the background, the end user only sees `InitialBackupCreated` in the progress bar. The problem aggrevates when the Backup contains a huge number of resources. The progress bar is stuck in `InitialBackupCreated` phase with a certain percentage value for a very long period of time. The user is left with no reason to believe that there's actually something happening in the background.

## Proposal

To address the problems discussed in the previous section, we propose to make two modifications:

1. Provide detailed progress information of ongoing migration phase in _MigMigration_ CR
2. Group existing phases into relevant steps 

### Enhancement 1: Detailed progress of phase

With this enhancement, we propose to modify _MigMigration_ CR to incorporate an additional field that shows progress of the ongoing phase. This is particularly useful for long running phases. This will require a significant change in _migration_ controller. 

Here are some examples of proposed change:

1. _StageBackupCreated_ Phase:

```yml
  status:
    conditions:
    - category: Advisory
      lastTransitionTime: "2020-09-16T15:39:44Z"
      reason: StageBackupCreated
      status: "True"
      type: Running
      progress:
      - [Backup migmigration-rpltn] 6 out of estimated total of 9 items backed up
```

Velero 1.4 onwards, we have the progress information reported in the Backup CR:

```yml
  status:
    completionTimestamp: "2020-09-16T15:38:58Z"
    expiration: "2020-10-16T15:38:52Z"
    formatVersion: 1.1.0
    phase: Completed
    progress:
      itemsBackedUp: 26
      totalItems: 26
```

For a _Backup_ with associated _PodVolumeBackup_ resources, the `progress` field would look like:

```yml
  progress:
  - [Backup migmigration-xxdmb] 0 out of estimated total of 6 items backed up
  - [VolumeBackup migmigration-xxdmb-6xmmb] 1001231 out of 2001230 bytes backed up
  - [VolumeBackup migmigration-xxdmb-gg9ll] 0 out of 1000123 bytes backed up
```

Similar changes will take place for _FinalBackupCreated_ phase too.

2. _StageRestoreCreated_ phase:

```yml
 status:
    conditions:
    - category: Advisory
      lastTransitionTime: "2020-09-16T15:41:41Z"
      reason: StageRestoreCreated
      status: "True"
      type: Running
      progress: InProgress
```

Detailed progress of _Restore_ objects is currently unavailable in Velero.  

Apart from the above phases, we propose to incorporate similar progress reporting for other phases of migration, wherever possible. For example, in Stage Pod phases, We can relay the status of the pods to _MigMigration_ CR. This is useful since we have been repeatedly observing issues in Stage pod phases.

 
### Enhancement 2: Grouping migration phases

With the progress information relayed in the _MigMigration_ CR, we believe that the user will have enough information in their hands to understand what exactly is happening in the background. We can then simplify the migration by dividing existing phases into broader steps of migration.  

All _Phases_ in a migration can be grouped into following high level steps that abstract out the details from the end user:

* Prepare
* VolumeBackup
* Backup
* VolumeRestore
* Restore
* Final

Having a fixed number of steps will make it possible for us to implement a pipeline type progress view in the UI. 

This can be implemented by introducing additional attribute `step` in the `status` field of the _MigMigration_ CR. Here is an example of _MigMigration_ CR with the proposed `step` field:

```yml
status:
  conditions:
  - category: Advisory
    lastTransitionTime: "2020-09-15T14:54:46Z"
    message: 'Step: 7/33'
    reason: InitialBackupCreated
    status: "True"
    type: Running
  itenerary: Final
  observedDigest: 9834d071975562d5e2c2eb855bca6950711ded8a0e45af5307fa56cd0f5ba3c7
  step: Backup
```

#### Relationship between Step and Phase

Each phase in the migration will belong to some _Step_ based on the the _Itinerary_. The existing itinerary struct will be updated to form a nested view of Phases through Steps. See the exact nested structs in [implementation details](#grouping-phases)


##### Final Itinerary 

- Prepare: Created, Started, Prepare, EnsureCloudSecretPropagated
- Backup: PreBackupHooks, EnsureInitialBackup, InitialBackupCreated
- VolumeBackup: EnsureStagePodsFromRunning, EnsureStagePodsFromTemplates, EnsureStagePodsFromOrphanedPVCs, StagePodsCreated, AnnotateResources, RestartRestic, ResticRestarted, QuiesceApplications, EnsureQuiesced, EnsureStageBackup, StageBackupCreated, EnsureStageBackupReplicated
- VolumeRestore: EnsureStageRestore, StageRestoreCreated, EnsureStagePodsDeleted, EnsureStagePodsTerminated
- ResourceRestore: EnsureAnnotationsDeleted, EnsureInitialBackupReplicated, PostBackupHooks, PreRestoreHooks, EnsureFinalRestore, FinalRestoreCreated, EnsureLabelsDeleted, PostRestoreHooks
- Final: Verification, Completed

##### Stage Itinerary

- Prepare: Created, Started, Prepare, EnsureCloudSecretPropagated
- VolumeBackup: EnsureStagePodsFromRunning, EnsureStagePodsFromTemplates, EnsureStagePodsFromOrphanedPVCs, StagePodsCreated, AnnotateResources, RestartRestic, ResticRestarted, QuiesceApplications, EnsureQuiesced, EnsureStageBackup, StageBackupCreated, EnsureStageBackupReplicated
- VolumeRestore: EnsureStageRestore, StageRestoreCreated, EnsureStagePodsDeleted, EnsureStagePodsTerminated
- Final: EnsureAnnotationsDeleted, EnsureLabelsDeleted, Completed

##### Failed Itinerary

- Final: MigrationFailed, EnsureStagePodsDeleted, EnsureAnnotationsDeleted, DeleteMigrated, EnsureMigratedDeleted, UnQuiesceApplications, Completed

##### Cancel Itinerary

- Final: Canceling, DeleteBackups, DeleteRestores, EnsureStagePodsDeleted, EnsureAnnotationsDeleted, DeleteMigrated, EnsureMigratedDeleted, UnQuiesceApplications, Canceled

#### How will migration progress through different steps?

In this section, we will discuss how a journey of a migration will look like to an end user.

##### Scenario 1: Migration completed without issues

Migration enters `Prepare` step, moves forward to `Backup` step, and so on until reaching the `Final` step and returning successfully.

```
Prepare -> Backup -> VolumeBackup -> VolumeRestore -> Restore -> Final
```

##### Scenario 2: Migration fails during StageBackupCreated phase

Migration enters `Prepare` step, moves forward to until `VolumeBackup` step and fails. Then, it skips the remaining steps and jumps to `Final` step. 

```
Prepare -> Backup -> VolumeBackup (Failed Here) -> VolumeRestore (Skipped) -> Restore (Skipped) -> Final
```

A user now knows where exactly the migration failed, they can inspect the _MigMigration_ CR and report the actual phase it failed at. 


### Implementation Details/Notes/Constraints

#### Velero Restore Progress

As of Velero 1.4, _Restore_ objects do not have Progress reported in the Status field like _Backup_ objects. In the first iteration, we will only show progress of _Restore_ objects in the form of `InProgress, Completed, Failed, PartiallyFailed`. Next, we will implement the Progress for Restore objects in Konveyor Velero fork and eventually, work to get the features submitted upstream.

#### Grouping Phases

Here's the new  _Itinerary_ struct with nested _Steps_ and _Phases_.

```go
// Itinerary
type Itinerary struct {
	Name  string
	Steps []Step
}
// Step
type Step struct {
	Name string
        Phases []Phase
}
// Phases
type Phase struct {
	// A phase name.
	Name string
	// Phases included when ALL flags evaluate true.
	all uint8
	// Phases included when ANY flag evaluates true.
	any uint8
}
```

