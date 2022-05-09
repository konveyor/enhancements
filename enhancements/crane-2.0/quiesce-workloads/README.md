---
title: quiescing-workloads
authors:
  - "@pranavgaikwad"
reviewers:
  - "@alaypatel07"
  - "@djzager"
  - "@jmontleon"
  - "@shawn-hurley"
approvers:
  - "@alaypatel07"
  - "@djzager"
  - "@jmontleon"
  - "@shawn-hurley"
creation-date: 2022-05-05
last-updated: 2022-05-09
status: implementable
see-also:
  - "N/A"  
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Quiescing Workloads

The main use case of "Import from cluster" wizard in Crane is to allow users to perform a _Cutover_ migration. In a cutover migration, users migrate workload resources and Persistent Volume data from source cluster to destination cluster. Before transferring PV data, workloads on the source cluster will be put to sleep to ensure that no new data is produced by applications during migration. Quiescing refers to the process of putting workloads to sleep. Currently, the wizard creates a _ClusterTask_ to quiesce workloads. It uses `kubectl scale` command to bring replicas of workloads to 0. Since the ClusterTask is only created by the UI, CLI users are expected to use `kubectl scale` command directly to bring down workloads. This enhancement proposes a new subcommand in Crane CLI to automate quiescing of workloads in the source namespace. The command can be used by both Crane CLI and the UI users to quiesce workloads.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

Quiesce _ClusterTask_ in Crane uses `kubectl scale` command to bring workloads down by setting replicas of workloads to 0. The _ClusterTask_ cannot be used by CLI users. To enable Cutover migrations using Crane, the CLI users need a way to quiesce down workloads in the source namespace.

## Motivation

### Goals

- Enable first class cutover migration flow for Crane CLI users

- Allow users to quiesce up/down workloads in source namespace

- Provide a single subcommand to bring down all supported workloads at once

- Enable quiescing of DaemonSet resources which is currently not possible using `kubectl`

## Proposal

A new subcommand `quiesce` will be added to Crane CLI.

```sh
> crane quiesce
```

The command will take two mandatory parameters:

```sh
> crane quiesce --context <source_cluster_context> --namespace <source_namespace>
```

When executed, it will quiesce the workloads running in `source_namespace` on the cluster accessible via `source_cluster_context`. 

### Execution 

The quiesce will discover all Deployments, DeploymentConfigs (OpenShift), StatefulSets, Jobs, CronJobs, ReplicaSets and DaemonSets. In general, the quiesce operation will work in three stages:

1. PreQuiesce
2. Quiesce
3. PostQuiesce

Depending on the workload type, each stage will work differently for different resources.

#### Deployments, DeploymentConfigs, StatefulSets, ReplicaSets

1. PreQuiesce: Set an annotation on the resource which to store the value of `spec.Replicas` field in the workload definition. This is needed to unquiesce applications back to their normal state.

2. Quiesce: Set `spec.Replicas` value to `0`.

3. PostQuiesce: Wait until all Pods owned by these workloads are terminated.

> Only ReplicaSets that do not have OwnerReferences set will be quiesced separately. 

#### Jobs

1. PreQuiesce: Set an annotation on the resource to store the value of `spec.Parallelism` field in the workload definition.

2. Quiesce: Set `spec.Parallelism` value to `0`.

3. PostQuiesce: Wait until all Pods owned by the workloads are terminated.

#### CronJobs

1. PreQuiesce: Set an annotation on the resource to store the value of `spec.Suspend` field in the workload definition.

2. Quiesce: Set `spec.Suspend` value to `true`.

3. PostQuiesce: Wait until all Pods owned by the workloads are terminated.

#### DaemonSets

For daemonsets resources, quiesce works by forcing the scheduler to schedule pods on nodes that do not exist.

1. PreQuiesce: Set an annotation on the resource to store the value of `spec.Template.Spec.NodeSelector` field in the workload definition.

2. Quiesce: Set `spec.Template.Spec.NodeSelector` value to a unique label not expected to be used by any of the nodes.

3. PostQuiesce: Wait until all Pods owned by the workloads are terminated.

### Optional Parameters

The subcommand will also have some additional optional parameters:

- `--force`: it will force the subcommand to return as soon as workload definitions are updated without waiting on the resources to be terminated. 

- `--rollback`: it will unquiesce the applications by returning them to their original state. 

### User Stories [optional]

#### Story 1

As a CLI user, I want to quiesce down applications running in the source namespace so that I can migrate the persistent data over to a new cluster and start my applications there.

### Implementation Details/Notes/Constraints [optional]

