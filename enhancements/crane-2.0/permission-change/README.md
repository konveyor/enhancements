---
title: changing-ownership-of-files-after-state-transfer
authors:
  - "@pranavgaikwad"
reviewers:
  - "@djzager"
  - "@shawn-hurley"
  - "@jmontleon"
  - "@alaypatel07"
approvers:
  - "@djzager"
  - "@shawn-hurley"
  - "@jmontleon"
  - "@alaypatel07"
creation-date: 2022-03-24
last-updated: 2022-03-28
status: implementable
see-also:
  - "N/A"
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Changing ownership of files after state transfer

'Smart Import' wizard in Crane UI can only be used after a destination namespace is created. The destination namespace will be assigned a unique UID range. The assigned range will be different than that of the source namespace from where the workload is being imported. By default, Rsync transfer in Crane preserves owners and permissions of the files during transfer. The transferred files are owned by a UID belonging to range of the source namespace. As a result, workloads running with a UID belonging to destination namespace won't be able to access the files. For workloads to continue working in the destination namespace, it is necessary that either the namespace UID range is same as that of the source or the files are owned by a UID belonging to the destination namespace. The destination namespace cannot use the same UID range as the source because the range may collide with another namespace in the cluster. The only possible solution left is to change the file ownership based on UID range of the destination namespace. This document describes a solution to change file ownership using a ClusterTask which can be run after PVC is transferred.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

This enhancement proposes a new ClusterTask that can be run to change ownership of files after a state transfer. It will ensure that the workloads will continue running in the destination namespace even though the UID range assigned to the namespace is not same as that of the source namespace. It also prevents collision of UID ranges in the destination cluster otherwise needed if we were to maintain the UID ranges. 

## Motivation

Pre-requisite for using Smart Import wizard of Crane UI is that the destination namespace already exists. In most cases, UID range assigned to the destination namespace will not be same as its source counterpart. The migrated workloads will not be able to access the migrated files because of mismatch in UID. MTC solves this problem by preserving the UID range annotations on the namespace resource. Crane cannot use the same approach because it does not migrate namespace resource. Therefore, the only possible option is to change the ownership of files as per UID range assigned to the destination namespace.

### Goals

- Propose a ClusterTask to change file ownership after state transfer

### Non-Goals

- Change file ownership during state transfer

## Proposal

A ClusterTask will run an Ansible Playbook to change ownership of files. The job of the ClusterTask is to simply run the playbook and pass variables to it. The actual ownership change will be handled by the Ansible Playbook.

### Ansible Playbook

The playbook will run following tasks:

- Read namespace annotations and figure out UID
  - In this step, the playbook will read value of `openshift.io/sa.scc.uid-range` annotation on the destination namespace.
  - By default, OpenShift uses first UID in the range. The playbook will store the first UID in a variable. See [this section](#determining-uid) for details.

- Discover PVCs in the namespace
  - Once the UID is determined, the playbook will discover all PVCs in the namespace.

- Launch a temporary Pod to run `chown` 
  - Finally, the playbook will launch a temporary Pod in the namespace. The Pod will attach all PVCs previously discovered to a known path.
  - Pod will run `chown` recursively with the previously determined UID on all PVC paths.

### User Stories [optional]

#### Story 1

As a user, I would like to change the ownership of files post state transfer such that the workloads continue running in the destination namespace despite their UIDs being different than the source.

### Implementation Details/Notes/Constraints [optional]

#### Determining UID

OpenShift namespaces are assigned a range of UIDs. If workloads in a namespace do not explicitely specify a UID, OpenShift uses first UID from the assigned range to launch the workload. Therefore, the ClusterTask will use the first UID from the range by default. If the workloads use an explicit UID, it needs to belong to the namespace range. If it does belong to the range, the workload can run. In such cases, the users can provide a custom UID for the workloads using extra variable passed to the playbook. 

## Design Details

### Test Plan

Mainly two flows will be tested:

- Workloads that do not specify a specific UID will use first UID from the range

- Workloads using a specific UID will need to pass UID as extra variable

## Drawbacks

The users are required to provide an extra variable in case the workloads use a specific UID. The variable is at namespace level and not at workload level.

