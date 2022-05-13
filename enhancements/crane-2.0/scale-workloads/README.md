---
title: scaling-workloads
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
creation-date: 2022-05-13
last-updated: 2022-05-13
status: provisional
see-also:
  - "N/A"  
replaces:
  - "/quiesce-workloads/README.md"
superseded-by:
  - "N/A"
---

# Scaling Workloads

This is a follow up enhancement superseding an enhancement previously submitted [here](../quiesce-workloads/README.md). This enhancement proposes a new `scale` subcommand in Crane.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

For cutover migrations in Crane, users are expected to quiesce their workloads. It is to ensure that new data is not written to Persistent Volumes during migration. This enhancement proposes a new subcommand `scale` in Crane CLI. It will help users automate scaling down/up resources in the source namespaces. The subcommand will work on known resources in Kubernetes.

## Motivation

Crane UI provides a _ClusterTask_ that internally runs `kubectl scale` command to bring down applications in the source namespace. CLI users are expected to use `kubectl scale` themselves to scale down individual resources. `kubectl scale` does not work with resources like Jobs, CronJobs and DaemonSets. The user experience can be improved by providing a similar `scale` command in Crane which will work on all resources in source namespaces at once.

### Goals

- Provide an automated way to scale standard Kubernetes workload resources in source namespaces 

- Compare the proposed subcommand with equivalent bash scripts to evaluate whether the subcommand will improve UX significantly

### Non-Goals

- Quiesce user workloads intelligently 

## Proposal

A new subcommand `scale` will be introduced in Crane CLI. The `scale` command can be used to scale down Deployments, DeploymentConfigs, Statefulsets, ReplicaSets, Jobs, CronJobs and Daemonsets by running:

```bash
> crane scale down --context <source_context> --namespace <namespace>
```

To scale the resources back to original state:

```bash
> crane scale up --context <source_context> --namespace <namespace>
```

The subcommand will wait until the resources reach their desired state.

If the user were use `kubectl` to achieve the same result as the `crane scale down` command, they could use a bash script that looks like: 

```bash
#!/bin/bash

resourceKinds=('deployment' 'deploymentconfig' 'statefulset' 'replicaset')
# Scaling down deployments, deploymentconfigs, statefulsets and replicasets 
for resourceKind in ${resourceKinds[@]}
do
  for resource in $(kubectl get ${resourceKind} --no-headers -o custom-columns=":metadata.name")
  do
  	# store current replicas
	currentReplicas=$(kubectl get ${resourceKind} ${resource} -o jsonpath='{.spec.replicas}')
	kubectl annotate ${resourceKind} ${resource} preQuiesceReplicas=${currentReplicas}
	# scale down
	kubectl scale ${resourceKind} --all --replicas=0
  done
done

# Scaling down jobs & cronjobs
resourceKinds=('job' 'cronjob')
for resourceKind in ${resourceKinds[@]}
do
  if [ ${resourceKind} == "job" ]; then
    replicaListPath='{.spec.parallelism}'
    replicaPatchPath="/spec/parallelism"
  else
    replicaListPath='{.spec.jobTemplate.parallelism}'
    replicaPatchPath=/spec/jobTemplate/parallelism
  fi

  for job in $(kubectl get jobs --no-headers -o custom-columns=":metadata.name")
  do
    # store current replicas
    currentReplicas=$(kubectl get job ${job} -o jsonpath=${replicaListPath})
    currentSuspended=$(kubectl get job ${job} -o jsonpath='{.spec.suspend}')
    kubectl annotate job ${job} preSuspendReplicas=${currentReplicas}
    kubectl annotate job ${job} preSuspendState=${currentSuspended}
  
    # scale down job
    kubectl patch job ${job} --type json -p '[ \
        {"op":"replace", "path": "/spec/suspend", "value": true}, \
        {"op": "replace", "path": '\"${replicaPatchPath}\"', "value": 0}]'
    echo "${job}"
  done
done

# Scaling down daemonsets
for ds in $(kubectl get daemonsets --no-headers -o jsonpath=":metadata.name")
do
  # store current nodeSelector
  currentNodeSelector=$(kubectl get daemonset ${ds} -o jsonpath='{.spec.template.spec.nodeSelector}')
  kubectl annotate daemonset ${ds} preSuspendSelector=${currentNodeSelector}

  # force un-schedule ds
  kubectl path daemonset ${ds} --type json -p '[{"op":"replace", "path":"/spec/template/spec/nodeSelector", "value": {"node":"doesNotExist"}}]'
done
```

To scale up resources back to their original state:

```bash
#!/bin/bash

resourceKinds=('deployment' 'deploymentconfig' 'statefulset' 'replicaset')
# Scaling up deployments, deploymentconfigs, statefulsets and replicasets 
for resourceKind in ${resourceKinds[@]}
do
  for resource in $(kubectl get ${resourceKind} --no-headers -o custom-columns=":metadata.name")
  do
  	# store current replicas
	preQuiesceReplicas=$(kubectl get ${resourceKind} ${resource} -o jsonpath='{.metadata.annotations.preQuiesceReplicas}')
	# scale up
	kubectl scale ${resourceKind} --all --replicas=${preQuiesceReplicas}
    # remove annotation
	kubectl annotate ${resourceKind} ${resource} preQuiesceReplicas-
  done
done

# Scaling up jobs & cronjobs
resourceKinds=('job' 'cronjob')
for resourceKind in ${resourceKinds[@]}
do
  if [ ${resourceKind} == "job" ]; then
    replicaListPath='{.spec.parallelism}'
    replicaPatchPath="/spec/parallelism"
  else
    replicaListPath='{.spec.jobTemplate.parallelism}'
    replicaPatchPath=/spec/jobTemplate/parallelism
  fi

  for job in $(kubectl get jobs --no-headers -o custom-columns=":metadata.name")
  do
    # store current replicas
    preSuspendReplicas=$(kubectl get job ${job} -o jsonpath='{.metadata.annotations.preSuspendReplicas}')
    preSuspendState=$(kubectl get job ${job} -o jsonpath='{.metadata.annotations.preSuspendState}')
    # scale up job
    kubectl patch job ${job} --type json -p '[ \
        {"op":"replace", "path": "/spec/suspend", "value": '$(boolean "${preSuspendState}")'}, \
        {"op": "replace", "path": '\"${replicaPatchPath}\"', "value": '${preSuspendReplicas}'}]'
    # remove annotation
    kubectl annotate job ${job} preSuspendReplicas-
    kubectl annotate job ${job} preSuspendState-
  done
done
```

### Implementation Details/Notes/Constraints [optional]

Scaling up daemonsets with the bash script is a little tricky. The temporary annotation that stores the pre-scaling value of annotation needs to be converted from string back to JSON before patching the DaemonSet. 

The bash script also doesn't handle errors well. Some extra logic is needed to incorporate error handling in the above script.

