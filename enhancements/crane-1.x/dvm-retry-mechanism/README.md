---
title: dvm-retry-mechanism
authors:
  - "@pranavgaikwad"
reviewers:
  - "@alaypatel07"
  - "@djwhatle"
  - "@shawn-hurley"
approvers:
  - "@alaypatel07"
  - "@djwhatle"
  - "@shawn-hurley"
creation-date: 2021-03-08
last-updated: 2021-03-15
status: implementable
see-also:
  - "N/A"  
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# DVM Retry Mechanism

Direct Volume Migration (DVM) in MTC allows migration of Persistent Volume data using Rsync through a direct connection between the source and the target cluster. This enhancement proposes a retry mechanism for DVM to guarantee a successful transfer when Rsync fails due to transient errors. 

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

When running Rsync to transfer Persistent Volume data from the source to the target cluster, it is possible that DVM encounters a transient/intermittent error causing Rsync to fail. Transient failures are more likely to happen for large scale Persistent Volumes.  Such failures can be overcome by simply running Rsync again. Rsync is capable of handling partially transferred data from previous attempts. However, DVM doesn't have a retry mechanism in place as of MTC 1.4.2. DVM simply marks the transfer as failed when the first attempt of Rsync fails. 

This enhancement proposes a retry mechanism for DVM. When implemented, it will allow setting maximum number of retries for Rsync. The DVM controller will retry Rsync until it succeeds or the number of retries has reached the maximum value, whichever comes first.

## Motivation

DVM controller creates an OpenShift Route in the target cluster to establish direct connection between the source and the target. An OpenShift Route is handled by _ingress-controller_. On an AWS backed OpenShift cluster, _ingress-controller_ typically uses an AWS ELB to load balance incoming requests. As a result, the incoming Rsync requests go through the ELB. When running Rsync for large scale Persistent Volumes, we have observed that the AWS ELB may drop connections intermittently during the transfer. These network errors cannot be solved deterministically. However, it is possible to retry Rsync repeatedly to workaround such intermittent issues.

### Goals

1. Add ability to retry Rsync until transfer is successful or the maximum number of attempts are reached.

### Non-Goals

1. Guarantee successful transfer when non-transient errors make the Rsync operation fail repeatedly.

2. Allow users to manually retry Rsync transfer.

## Proposal

The proposed retry mechanism will be implemented in DVM controller. DVM CR will create a Rsync Pod per PVC as previously done. However, instead of marking the migration as failed upon first failure, it will re-create the Rsync Pods until the specified number of retries is met or the most recently created Rsync Pod exits with 0 return code. The number of failures will be kept track of in the DVM CR's _Status_.

This approach will work on all supported versions of MTC. It also leaves room for enhancements in the retry logic itself. See [alternative approaches](#alternatives) considered and the problems with each of them. 

### Implementation Details/Notes/Constraints

Throughout this document, a single _Rsync Pod_ launched by the DVM controller will be referred to as an _Rsync Attempt_, while the collection of all attempts will be referred to as _Rsync Operation_, interchangeably.

#### Specifying retries

The number of retries Rsync performs will be configured through _DirectVolumeMigration_ CR. A sane default value will be chosen in case the user doesn't specify their own retry value:

```golang
type DirectVolumeMigrationSpec struct {
	[...]
	BackOffLimit int `json:"backOffLimit,omitempty"`
}
```

#### Creating Pods

In _VolumeMigration_ itinerary of the DVM controller, _[CreateRsyncClientPods](https://github.com/konveyor/mig-controller/blob/523d3a8ef70e89296535d2d0b94173669e92007c/pkg/controller/directvolumemigration/task.go#L126)_ phase is responsible for creating Rsync Pod. This phase will create pods as usual and add a unique label on each pod indicating which PVC it belongs to. 

#### Direct Volume Migration Progress

Most of the logic in determining the state of the ongoing Rsync operation depends on _Status_ field of _DirectVolumeMigrationProgress_ CR which is created per Rsync client Pod. It takes a reference to the MigCluster on which Rsync Pods are running and the reference to the Pod itself: 

```golang
type DirectVolumeMigrationProgressSpec struct {
	ClusterRef *kapi.ObjectReference `json:"clusterRef,omitempty"`
	PodRef     *kapi.ObjectReference `json:"podRef,omitempty"`
}
```

DVMP will be modified to watch all attempts of Rsync instead of watching just one Pod. A new field `PodSelector` will be introduced to input unique label added by the DVM controller on the Pods of each Rsync attempt.

```golang
type DirectVolumeMigrationProgressSpec struct {
	[...]
	// PodSelector selector to find all attempts of Rsync
	PodSelector     map[string]string `json:"podSelector,omitempty"`
	// BackOffLimit retries limit set by DVM
	BackOffLimit    int               `json:"backOffLimit,omitempty"`
}
```

A new struct `RsyncPodStatus` will be introduced which will encapsulate the _Status_ information of an individual Rsync attempt. `RsyncPodStatuses` in the DVMP _Status_ will hold the historical information of all Rsync attempts. The `RsyncPodStatus` of the most recent Rsync attempt will also be included as top level field in DVMP status to avoid repeated searches through the list of attempts. Field `RsyncElapsedTime` will record the elapsed time of the ongoing Rsync operation as a whole (including all retries). `Succeded`, `Failed` and `Completed` fields in the _Status_ will be introduced to hold the derived status of the Rsync operation. `BackOffLimit` will be used to verify whether the derived status of the Rsync operation. This simplifies logic wherever DVMP is being used in other places.

```golang
type DirectVolumeMigrationProgressStatus struct {
	[...]
	// RsyncPodStatus describes information about most recent Rsync attempt
	RsyncPodStatus
	// RsyncPodStatuses is a list of status information of all Rsync attempts
	RsyncPodStatuses            []RsyncPodStatus `json:"rsyncPodStatuses,omitempty"`
	// RsyncElapsedTime is the total time spent in Rsync operation including all retries
	RsyncElapsedTime            *metav1.Duration `json:"rsyncElapsedTime,omitempty"`
	// TotalProgressPercentage is the sum of progress percentage of all Rsync attempts
	TotalProgressPercentage     int              `json:"totalProgressPercentage,omitempty"`
	// Succeded whether the Rsync operation succeded
	Succeded                    bool             `json:"succeded,omitempty"`
	// Failed whether the Rsync operation failed
	Failed                      bool             `json:"failed,omitempty"`
	// Completed whether the Rsync operation has finished - failed or succeded
	Completed                   bool             `json:"completed,omitempty"`
}

type RsyncPodStatus struct {
	// PodName name of the Rsync Pod
	PodName                     string           `json:"podName,omitempty"`
	// PodPhase phase of most recent Rsync Pod
	PodPhase                    kapi.PodPhase    `json:"phase,omitempty"`
	// ExitCode exitCode of the Rsync Pod
	ExitCode                    *int32           `json:"exitCode,omitempty"`
	// ContainerElapsedTime elapsed time of Rsync container
	ContainerElapsedTime        *metav1.Duration `json:"containerElapsedTime,omitempty"`
	// LogMessage log lines from the Rsync container
	LogMessage                  string           `json:"logMessage,omitempty"`
	// LastObservedProgressPercent progress percentage observed in Rsync container
	LastObservedProgressPercent string           `json:"lastObservedProgressPercent,omitempty"`
	// LastObservedTransferRate transfer rate observed in Rsync container
	LastObservedTransferRate    string           `json:"lastObservedTransferRate,omitempty"`
	// CreationTimestamp is the Pod creation timestamp
	CreationTimestamp           *metav1.Duration `json:"creationTimestamp,omitempty"`
}
```

Field `TotalProgressPercentage` is explained in more detail further down in [this section](#deriving-rsync-progress). 

#### Watching Rsync Operation

In _VolumeMigration_ itinerary, _[WaitForRsyncClientPodsCompleted](https://github.com/konveyor/mig-controller/blob/523d3a8ef70e89296535d2d0b94173669e92007c/pkg/controller/directvolumemigration/task.go#L127)_ phase is responsible for waiting for all Rsync Client pods to finish. It lists all _DirectVolumeMigrationProgress_ (DVMP) CRs and waits until all of them indicate a terminal status (failed/succeeded). This logic will be modified to not only watch the completion of Rsync Pods but also re-create the failed Pods until success. The watching logic will depend on _DirectVolumeMigrationProgress_ CR as previously done. The difference here will be that the :

```golang
// when the operation is running
if (!dvmp.Status.Completed) {
  ...
}
// when the operation is failed
if (dvmp.Status.Failed) {
  ...
}
// when the operation is succeded
if (dvmp.Status.Succeded) {
  ...
}
```

The existing check that determines whether any of the Rsync Pods are in pending state will still work without any change as DVMP _Status_ has the top level _RsyncPodStatus_ which shows the status of the most recent Rsync attempt:

```golang
if (dvmp.Status.PodPhase == corev1.PodPending) {
  ...
}
```

Similarly, the methods `hasAllRsyncClientPodsTimedOut()` and `isAllRsyncClientPodsNoRouteToHost()` will work by looking at the most recent Rsync Pod's status. The only difference is that these methods will only be triggered when the Rsync Operation is in its terminal state - failure/success.

#### Deriving Rsync progress

If the Rsync process fails at 10% in the first attempt, the second attempt starts from 0% and goes upto 90%. If the second attempt fails at 30%, the third attempt starts at 0% and goes upto 60% as rest of the 40% is already transferred in the previous attempts. The total progress percentage can be calculated by simply addding `lastObservedProgressPercent` of all Rsync attempts. The field `totalProgressPercentage` in DVMP _Status_ stores the calculated total percentage of all attempts. If an operation is determined to be successful, this value will be set to 100 without looking at individual pod percentages. In all other cases, the sum of all progress percentages will be used.

#### Integration with mig-log-reader

The _Status_ field in DVMP CR contains history of all Rsync attempts. The `LogMessage` field in each `PodStatus` only contains last 5 lines of the logs from Rsync Pod. In case of a failure even after all Rsync attempts, this information may not be sufficient to determine the cause. On the other hand, searching through logs of Pods of all Rsync attempts can be a daunting task. Therefore, the Rsync pod logs will be integrated with `mig-log-reader`. Currently, `mig-log-reader` only watches `openshift-migration` namespace. But there is a way to supply `--all-namespaces` along with a `--selector` argument which will be used to filter the Pods we need. The exact details of this implementation are not certain yet. But after discussing with [@djwhatle](https://github.com/djwhatle), it is clear that the `mig-log-reader` is best suited for this use case. New labels will be introduced across all the migration pods to make it easier to find pods across namespaces. Upon complete failure, users will simply be asked to look at logs collected by Stern in the source namespace. The relevant logs can be found by grepping for Rsync Pod names present in the DVMP CR _Status_. 

#### Cleaning up Pods

DVM creates one Rsync Pod for every _Persistent Volume_. In case of multiple failures, the number of Rsync Pods launched by the DVM controller in a namespace can grow significantly possibly causing nuisance for users. The only purpose we leave the Rsync Pods behind is because we want to let users give a chance to debug issues in case of failures. However, if we are able to integrate the logs with `mig-log-reader`, we no longer need to keep the Rsync Pods. Therefore, all Rsync Pods will be deleted even when they fail. The history of Pods can still be found in DVMP _Status_.

The _DirectVolumeMigrationProgress_ CR will add a label on Rsync Pod whenever they are done reading the status information of the Pod. This label will act as a cue to the _DVM_ controller indicating that the pod can now be successfully cleaned up. Upon failure, _DVM_ controller will only delete the Pod if the label is present on the Pod leaving time for DVMP to collect status information. This rolling deletion will make sure that we don't leave failed pods running in the source namespace. 

## Design Details

### Test Plan

- An automation will be written to simulate transient Network failures by creating and deleting _Network Policy_ in source namespace intermittently. It will be verified that the Rsync Operation as a whole works as expected in such a scenario. 

- Various aspects of progress reporting system will be verified. Mainly, it will be verified whether the progress bar picks up from where it left in the previous failed attempt. 

- Stern integration will be verified.

- Unit tests will be written wherever necessary.

### Upgrade / Downgrade Strategy

#### Upgrading from 1.4.2-

As of MTC 1.4.2, the _DirectVolumeMigrationProgress_ CR doesn't have `PodSelector` and `BackOffLimit` fields. All the existing CRs will instead have a `PodRef` field. Also, the _DVMP_ CR did not have a _Completed_ condition. As a result, old CRs will be reconciled even when we introduce the new `JobRef` field. Old CRs will need to be marked _Completed_ at least once to avoid breakage going forwards. New logic will be introduced to add _Completed_ condition. The reconciler will be updated to check for _Completed_ condition and not reconcile CRs which are already completed. Old CRs with `PodRef` will be marked _Completed_ regardless of whether they actually completed or not. In case of new CRs with JobRef field, the _Completed_ condition will be added once the Rsync Operation is deemed completed. Perhaps, this will also make the dedicated _Status_ field _Completed_ redundant.

Any existing Rsync Pods running in the source namespaces will still be cleaned as the cleanup logic of Rsync Pods will not be removed in the version in which this enhancement is implemented. It will be removed in the later versions.

#### Downgrading to 1.4.2-

As of MTC 1.4.2, _DirectVolumeMigrationProgress_ CRs are reconciled irrespective of whether they completed or not. Existing CRs that have `PodRef` field will be marked _Completed_ when an upgrade is made to a version in which this enhancement is implemented. On rollback to an earlier version of MTC, however, this condition will have no effect. Instead, `PodRef` will be continued to be used by the controller. The old controller will attempt to find the Pod referenced in the CR, and add a _NotFound_ condition on the CR without breakage.

## Alternatives

As of MTC 1.4.2, DVM controller creates a Pod to run Rsync. It creates as many Pods as there are volumes in the source cluster. Rsync command is injected as startup command in the Pod. When Rsync fails, the Rsync client container in the Pod never restarts as the _restartPolicy_ is set to _Never_ by default. It is possible to simply change the _restartPolicy_ of the Rsync client Pod to _OnFailure_ which will cause Kubelet to restart the Rsync container whenever failure takes place. However, it is not desired for our use case because Kube uses exponential backoff to restart the container. In case of multiple failures, the wait time between two consecutive retries can go upto 5 minutes. The time between consecutive launchs of a container cannot be configured. For our use-case, we want the pods to be re-launched immediately after a failure. 

[Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/) are ideal for our use case and provide the retry feature we need. A Job uses `.spec.template` to specify a Pod template. The field `.spec.backoffLimit` allows setting maximum number of retries on the Job. The _Status_ fields `.status.succeeded` and `.status.failed` tell the number of successful and failed attempts respectively. The Rsync client Pods can be replaced with Jobs to leverage their inherent retry mechanism. The controller can watch the status of the Job instead of watching individual Pods. However, this approach is undesireable for two reasons:

1. Kubernetes Job API introduced `.spec.backOffLimit` field in Kubernetes 1.11. The field was then backported to 1.10. It doesn't exist in 1.7, 1.8 and 1.9 which are supported versions of MTC.

2. Pods are created by the Job controller with its own identity and we want the Pods to be created with migration-controller's identity instead.