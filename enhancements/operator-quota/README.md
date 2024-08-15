---
title: neat-enhancement-idea
authors:
  - "@jortel"
reviewers:
  - TBD
  - "@alicedoe"
approvers:
  - TBD
  - "@jmontleo"
creation-date: 2024-08-15
last-updated: 2024-08-15
status: implementable
---

# Operator Managed Resource Quota

Enhance the operator to optionally manage a ResourceQuota in the namespace.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

Enhance the operator to optionally manage a ResourceQuota in the namespace.
The user may optionally specify a percentage of cluster resources to be utilized by konveyor.
This mainly applies to task pods.

## Motivation

Tasks may be created in batches of thousands. When this happens, the task manager will create
thousands of pods and relies on the node scheduler to run them in the most efficient manner.
This has the following drawbacks:
- Potentially _hogs_ all of the node resources.
- The Task Manager no longer has control over the order in which tasks/pods are executed (other than the order it created all of the pods).
- The `oc get pod` in the namespace produces thousands of entries.
- Puts needless pressure on the k8s controllers and etcd.

Quotas are the Kubernetes mechanism to achieve this.

### Goals

Maximize task manger scheduling features.
Be a better citizen on the cluster.

### Non-Goals

## Proposal

Enhance the operator to optionally manage a Resource Quota in the namespace.
The user would specify a percentage of cluster resources to be utilized. The operator
would list the allocatable resources on cluster nodes and dynamically deploy a 
quota.  The percentage would be a property on the Tackle CR with a default of 85% (open
to suggestions here). A percentage is used because it's a value that users and define
without knowing anything about node configurations. A percentage will automatically
adjust as nodes are added/removed or reconfigured.

Quotas are routinely use by Development and QE and have been very effective at managing
cluster resources and maximizing the Task manager scheduling features.

## Design Details

### Test Plan

### Upgrade / Downgrade Strategy
