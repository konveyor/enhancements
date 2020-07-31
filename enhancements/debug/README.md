# MTC Debug Experience Improvements

The objective of this document is to capture the current debug experience,
catalog the high level types of common issues that can occur, and collectively
identify opportunities for improvement, with a goal of implementing tractable
improvements in the 1.3.0 release (and beyond).

## Related issues:

https://github.com/konveyor/mig-controller/issues/612

## Example Debug Story

I'll use a recent debug session to illustrate issues in the current debug process.
I recognize this is just one example and different problems will have different
steps, but I think it's useful to describe a typical process.

I have a 3.11 source cluster and a 4.4.12 cluster with MTC 1.2.4-rc deployed into it
(this is from staging since it has not been released yet). On my 3.11 cluster,
I have a simple nginx-example toy app that I use for testing. It has a pv that
it stores its logs in. Generally in my flow of testing, I migrate it, and the
source workload gets quiesced. To reset, I delete the target's namespace to make
room to do the migration again, and I scale back up the source side. This was all
using filesystem copy. However, it had been a week or so and I forgot to delete
the target namespace, didn't realize it, scaled up the source side, and ran a
stage migration. After an unexpectedly long amount of time,
the UI status appeared to be stuck in `StageRestoreCreated`, so something "felt wrong".

Based on the phase, I knew the backup phase had completed and I'm in stage
restore, so I should look to the target cluster and its restores for more
information about what's potentially wrong. The place to start is to get the
relevant `MigMigration`, but when the UI creates the `MigMigration`, it uses
a UUID to name it and doesn't display that name in the UI [1].

To get the relevant `MigMigration` CRs related to the `MigPlan` in question,
you need to run a cli cmd to filter out irrelevant CRs (there could be hundreds
of `MigMigration` CRs in the cluster). Currently, the `MigPlan` is referenced
on a `MigMigration` via `spec.migPlanRef.name`, which is the authoratative
source. We need to add a command that returns all `MigMigrations` for that plan [2].

> NOTE: It's **very** important that these `oc` commands are run on the correct
cluster, whether that's the source or the target cluster. We think that introducing
features into the UI to automate a lot of this will simply *ensure* the commands
are run against the correct clusters so the user doesn't even have an opportunity
to mistake it.

Assuming you've gotten the relevant `MigMigration` name via the footnote's
improvement, now we know we need to look at the restores to see if anything
is wrong. This must be run on the target cluster of course, because we're looking
for the restores. Confusingly, the restore has a label called `migmigration`
on it, and **the value is actually the uid from the migmigration, NOT the name**.
Both appear to be UUIDs, which can be a significant source of confusion. [3]

```sh
# NOTE: MigMigration objects have a migmigration label on them, but confusingly,
the value is the uid of the MigMigration, not the name. See the 3rd footnote.
oc get restores -l migmigration=<MigMigrationUid> -o yaml`
```

```yaml
apiVersion: v1
items:
- apiVersion: velero.io/v1
  kind: Restore
  metadata:
    annotations:
      openshift.io/migrate-copy-phase: stage
      openshift.io/migration-registry: 172.30.129.158:5000
      openshift.io/migration-registry-dir: /test-registry-a5554d3e-5ad9-4c1e-9978-7e4aecf24adb
    creationTimestamp: "2020-07-28T19:03:28Z"
    generateName: e26a19a0-d104-11ea-a7c7-a386498433aa-
    generation: 2
    labels:
      app.kubernetes.io/part-of: openshift-migration
      migmigration: 85d81222-f2fb-4457-817f-077139210b18
      migration-stage-restore: 85d81222-f2fb-4457-817f-077139210b18
    name: e26a19a0-d104-11ea-a7c7-a386498433aa-rw2w6
    namespace: openshift-migration
    resourceVersion: "4876371"
    selfLink: /apis/velero.io/v1/namespaces/openshift-migration/restores/e26a19a0-d104-11ea-a7c7-a386498433aa-rw2w6
    uid: 002b222d-730f-4345-805a-6e88ca6775c5
  spec:
    backupName: e26a19a0-d104-11ea-a7c7-a386498433aa-xg6ph
    excludedResources:
    - nodes
    - events
    - events.events.k8s.io
    - backups.velero.io
    - restores.velero.io
    - resticrepositories.velero.io
    restorePVs: true
  status:
    phase: InProgress
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

Above is the response to the restore query. Checking the status, it is stuck
`InProgress`. At this point, I wanted to check the velero logs to see if there
were any errors, but was met with an error from their cli:

```sh
velero -n openshift-migration restore logs e26a19a0-d104-11ea-a7c7-a386498433aa-rw2w6
An error occurred: unable to retrieve logs because restore is not complete
```

Checking back in the UI, the `MigMigration` was finally marked failed. This is
because velero, and more specifically restic, timed out, and pushed the `Restore`
into a `PartiallyFailed` state. The status continues to read the last phase
written to the CR [4].

```yaml
# Snipping a bunch of this, the failed Restore:
apiVersion: v1
specificallyitems:
- apiVersion: velero.io/v1
  kind: Restore
    name: e26a19a0-d104-11ea-a7c7-a386498433aa-rw2w6
#...
  status:
    errors: 1
    phase: PartiallyFailed
    warnings: 1
```

Note the status (and lack of information there). In order to actually elaborate
on these errors and warnings, you need to run a `velero describe`:

```
# $ velero -n openshift-migration restore describe e26a19a0-d104-11ea-a7c7-a386498433aa-rw2w6

# ...snipping

Restore PVs:  true

Phase:  PartiallyFailed

Validation errors:  <none>

Warnings:  <error getting warnings: DownloadRequest.velero.io "e26a19a0-d104-11ea-a7c7-a386498433aa-rw2w6-20200728162351" is invalid: status.phase: Unsupported value: "": supported values: "New", "Processed">

Errors:  <error getting errors: DownloadRequest.velero.io "e26a19a0-d104-11ea-a7c7-a386498433aa-rw2w6-20200728162351" is invalid: status.phase: Unsupported value: "": supported values: "New", "Processed">

Restic Restores (specify --details for more information):
  New:  1
```

That's not a very helpful error, so I grepped the velero pod logs directly for
errors related to the restore:

```
# oc logs velero-7478d86474-tzxr4 | grep e26a19a0-d104-11ea-a7c7-a386498433aa-rw2w6 | grep error
time="2020-07-28T20:03:28Z" level=error msg="unable to successfully complete restic restores of pod's volumes" error="timed out waiting for all PodVolumeRestores to complete" logSource="pkg/restore/restore.go:1287" restore=openshift-migration/e26a19a0-d104-11ea-a7c7-a386498433aa-rw2w6
```

Here we finally see the root issue for the Restore failure: velero time out
waiting for the `PodVolumeRestore` to report completion.

In order to get the pod volume restore, you need to know the name of the `Restore`
resource, which is used as a prefix for `generateName` [5]. Grepping for that
gives me the full name of the relevant PVR:

```sh
# oc get podvolumerestores | grep e26a19a0-d104-11ea-a7c7-a386498433aa
e26a19a0-d104-11ea-a7c7-a386498433aa-rw2w6-h5k5l   90m
```

Getting this PVR shows a status with empty progress:

```yaml
status:
  progress: {}
```

Looking at this status, I know the PVR controller never was able to actually
begin the data transfer; it's an empty object. Based on experience [6], that's
probably an issue with a stage pod, and sure enough the stage pod is stuck in
an init state:

```sh
# oc get pods
NAME                                            READY   STATUS     RESTARTS   AGE
nginx-deployment-559df9786b-6ngmb               1/1     Running    0          7d21h
stage-nginx-deployment-6c7958f7b7-4zh7t-gflm6   0/1     Init:0/1   0          92m
```

Describing the pod to investigate the events reveals the ultimate problem:

```sh
o describe pod stage-nginx-deployment-6c7958f7b7-4zh7t-gflm6
Warning  FailedAttachVolume  96m                 attachdetach-controller                             Multi-Attach error for volume "pvc-9bbcd87d-b4d6-493a-8813-db81ee176d66" Volume is already used by pod(s) nginx-deployment-559df9786b-6ngmb
```

As it turned out, my nginx pod was already happily deployed and running with the
pvc attached, so the stage pod was stuck while trying to mount a pvc it never
would be able to double mount.

## Proposed Backend Improvements

[1] It is not clear from the UI what the name of a migration under
a plan is. The UI gives them UUID names, so it's not immediately useful to GUI
users, however, if you're going to begin the debug process if something goes
wrong, it is important to get this `MigMigration`. Suggestion is to add the
name in a tooltip.

[2] Add `MigPlan` name as a label to the `MigMigration`. Apart from adding this
info to the UI, cli users should be able to easily filter out irrelevant `MigMigrations`
to list those that are associated with the `MigPlan` A couple options are available
to us:

The controller can add the `MigPlan` name as a label so the user can use
`oc get migmigrations -o yaml -l migrations.openshift.io/migPlanName=<name>`.
~~Alternatively, we may be able to mark the `spec.migPlanRef.name` as indexed so
that users can use a field selector in their oc command to retrieve it instead
of a label. This is preferable because a) it's the authoratative primary key,
and 2) it doesn't require the controller to write any other data that could
potentially get out of sync.~~ The only field selectors available for custom
resources right now are [name and namespace](https://github.com/kubernetes/kubernetes/issues/53459),
so we'll need to go with the controller applied label selector.


[3] `Backup` and `Restore` objects have the label `migmigration=<uid>` on it.
It was very confusing to me that this value was actually the uid from the
migmigration and not the name of the `MigMigration`, because they are BOTH UUIDs.
I didn't understand why they did not match until I realized it was the uid field.
I would propose we 1) make that label very explicit which field it's referencing,
and 2) add a label that contains the name of the related `MigMigration` so it
can be queried via label selector.

[4] It's not clear what the "phase" actually represents. Trying to be specific,
is this the "last started phase", and when an error occurs, the phase is simply
left alone? So is it fair to say the phase seen here really is the "failed" phase?
Is there anything that can be done here to help? There is some perception that this
is the last successfully completed phase, but I don't think this phase
actually completed in my case, based on the itinerary:
https://github.com/konveyor/mig-controller/blob/master/pkg/controller/migmigration/task.go#L111-L112

[5] This is not good; we should explore if it's possible to label PVR/PVBs
with the `MigMigration` name so they can also be retrieved by label selector;
preferably by name and NOT uid.

[6] Information about stage pods and the problems that can be introduced should
be noted in the debug guide.

## Generic Debug Algorithm

**<TODO>**

## Proposed UI debug experience

**<TODO>** Link to second doc, POC?
