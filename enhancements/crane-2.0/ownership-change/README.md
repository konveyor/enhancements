---
title: changing-ownership-of-files-after-during-state-transfer
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
last-updated: 2022-04-18
status: implementable
see-also:
  - "N/A"
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Changing ownership of files after/during state transfer

'Import from cluster' wizard in Crane UI can only be used after a destination namespace is created.

On OpenShift clusters, the destination namespace will be assigned a unique UID range. The assigned range will be different than that of the source namespace from where the workload is being imported. By default, Rsync transfer in Crane preserves owners and groups of the files during transfer. The transferred files are owned by a UID belonging to range of the source namespace. As a result, workloads running with a UID belonging to destination namespace won't be able to access the files. For workloads to continue working in the destination namespace, it is necessary that either the namespace UID range is same as that of the source or the files are owned by a UID belonging to the destination namespace. The destination namespace cannot use the same UID range as the source because the range may collide with another namespace in the cluster. The only possible solution left is to change the file ownership based on UID range of the destination namespace. 

On Kubernetes clusters, kubelet updates file ownership automatically whenever `fsGroup` field is specified in the PodSecurityContext. It makes sure that the GID specified in `fsGroup` is added to Supplemental Groups of the container process. Also, it runs `chown` recursively on the volume mount before exposing it in the Pod. However, this is not true for _all_ storage types. Automatic `chown` does not work in following cases:
- [Kubernetes >= 1.19] whenever a CSI driver opts out recursive modifications by setting `fsGroupPolicy` to `None`
- [Kubernetes < 1.19] whenever a CSI volume does not specify `fsType` OR when `fsType` is present but volume is not ReadWriteOnce
- for shared storages such as NFS or in-tree storages that do not explicitely change the permissions

This document proposes an approach to change owners/groups of the files transferred using Crane. 
## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

This enhancement discusses a possible solution to fix owners and groups of files migrated using Crane state transfer. The solution works for both OpenShift and Kubernetes environments. It proposes to change the owners/groups using Rsync options. It will work for non-admin users as well. 

On OpenShift clusters, changing the owners/groups will ensure that - 

* migrated workloads will continue running in the destination namespace even when there's mismatch in UID ranges
* there's no collision of UID ranges in the destination cluster otherwise needed if we were to maintain the UID range

On Kubernetes clusters, changing the owners/groups will ensure that the users migrating into storages that do not automatically change the ownership still have a manual way to fix ownership using Crane.
## Motivation

Pre-requisite for using 'Import from cluster' wizard of Crane UI is that the destination namespace already exists. On OpenShift clusters, UID range assigned to the destination namespace will not be same as its source counterpart. The migrated workloads will not be able to access the migrated files because of mismatch in UID. MTC solves this problem by preserving the UID range annotations on the namespace resource. Crane cannot use the same approach because it does not migrate namespace resource. Therefore, the only possible option is to change the ownership of files as per UID range assigned to the destination namespace. Similarly on Kubernetes clusters, in some storages, kubelet does not perform automatic `chown`. Therefore, users need a manual way to fix ownership as needed. 

### Goals

- Propose solutions to fix owners/groups of files transferred using state transfer in Crane
  - Design a solution that can be used by both OpenShift & Kubernetes users
- Design a common solution such that Crane CLI and UI users both can leverage it
- Design a solution that works for non-admin users as well
- Discuss pros and cons of proposed solutions
### Non-Goals

- Change permissions of files
- Propose a solution for admin users only
## Proposal

Currently, the Rsync Daemon & Rsync Client both run as `root` user. It is achieved by adding a custom SCC `rsync-anyuid` which allows running the Rsync Pod as any UID. In the proposed solution, the Rsync Pods will run as less privileged user by default. This ensures that non-admin users can run Rsync Pods without issues. Optionally, admin users can choose to run Rsync as `root` user. 
### Default Operation

By default, the Rsync Pods will run with minimum capabilties. The Rsync Pod definition will be updated to drop certain capabilities:

```yaml
capabilities:
  drop:
    - KILL
    - MKNOD
    - SETGID
    - SETUID
```

By dropping these capabilities, we ensure that
- the Pods don't require special privileges
- we do not need a separate SCC to admit pods. they will be admitted via existing admission policies of the cluster (SCC/PodSecurityPolicy/PodSecurityAdmission) e.g. `restricted` SCC. 

On OpenShift clusters, `restricted` SCC allows Pods that drop above mentioned capabilities. Therefore, Rsync Pods will use `restricted` SCC on OpenShift clusters. On Kubernetes clusters, an equivalent of `restricted` PodSecurityPolicy or PodSecurityAdmission will determine the admission of the Rsync Pods. It is hard to tell whether all Kubernetes clusters will have an equivalent of `restricted` PodSecurityPolicy or PodSecurityAdmission. However, we will ensure that the capabilities of Rsync Pods are kept to the minimum such that they do not require a privileged user.

Rsync Daemon Pod will not run as `root` user by default. Therefore, `uid`, `gid` fields from Rsync Daemon Pod's configuration will be removed:

```diff
-uid=root
-gid=root
```

`restricted` SCC on OpenShift sets `runAsUser` to `MustRunAsRange`. This ensures that the Rsync Pods cannot violate the UID range assigned to the namespace in which they are running. When a specific UID is not specified in the Pod definition, OpenShift automatically assigns first UID from the range to the Pod. As a result, Rsync Pod will automatically run as the first UID from the range of the namespace. See [Custom UIDs/GIDs](#providing-custom-uidgid) which can be used in case users want UIDs/GIDs different than the first UID.

`restricted` PodSecurityPolicy/PodSecurityAdmission on Kubernetes set `runAsUser` field to `MustRunAsNonRoot`. It ensures that the containers cannot run as root users. Since we have already dropped the capabilities, the Rsync Pods will be admitted via this policy. Unlike OpenShift environments, it is hard to tell what UID/GID will be used by default on Kubernetes. It is because the default behavior SCC provides does not exist in Kubernetes. Therefore, users will be required to provide a UID/GID as explained in [Custom UIDs/GIDs](#providing-custom-uidgid).

> See [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) for details on `restricted` policy discussed above

Since `SETGID` and `SETUID` capabilities are removed from the Rsync Pods, the `--group` & `--owner` options cannot be used anymore. This is where Rsync will fix the ownership of files. By removing these options, the ownership will not be preserved. It means that all files will not be have same UID/GID in the target cluster. The files will instead have UID/GID assigned via default behavior of SCC/PodSecurityPolicy or the ones specified by the user via custom UID/GID fields. On OpenShift, by default (unless overridden), the files will be owned by first UID/GID from namespace range. On Kubernetes, the users will provide custom values. This is because Rsync process will write the files using the assigned UID/GID instead of writing them as root. Since `--owner` and `--group` options are removed, Rsync will not run `chown`/`chgrp` which requires escalated privileges. 

#### Providing custom UID/GID

Rsync Command will expose two new options to provide UID/GID values:

```go
type CommandOptions struct {
  [...]

  Gid *int64
  Uid *int64
}
```

These options will be available in the Crane `transfer-pvc` subcommand: 

```sh
crane transfer-pvc <...> --uid=1000680000 --gid=1000680000
```

These UIDs/GIDs will be passed to the SecurityContext of Rsync Containers:

```yaml
securityContext:
  runAsUser: 1000680000
  runAsGroup: 1000680000
```

Although, the above IDs are custom, it is necessary that they do not violate the SCC or PodSecurityPolicy. These options only exist in case the users require to override the defaults. For example, OpenShift assigns first UID by default. In some cases, the applications may be using another valid UID from the Namespace range.

### User Stories [optional]

#### Story 1 [OpenShift]

As an OpenShift user, I would like to change ownership of files migrated with state transfer such that the workloads continue running in the destination namespace despite their UIDs being different than the source.

#### Story 2 [Kubernetes]

As a Kubernetes user, I need a way to fix ownership of files migrated with state transfer such that I can migrate existing volumes into a new storage that does not automatically update file ownership using `chown`.

### Implementation Details/Notes/Constraints [optional]

#### Exceptions to SCC

We are only relying on SecurityContext fields to ensure right SCC / PodSecurityPolicies are attached. It is possible that the environments may have multiple conflicting SCC / PodSecurityPolicies and the Rsync Pods may end up using something other than `restricted`. It is likely that the SCC/PodSecurityPolicies admitting the Rsync Pods doesnt have the same `fsGroup`/`runAsUser` policies as `restricted`. In such cases, we cannot guarantee that the Rsync Pods will be admitted at all. The solution to such problems cannot be generalized, and it will have to be solved on a case-by-case basis.

### Test Plan

Mainly two flows will be tested:

- Workloads that do not specify a specific UID will use first UID from the range

- Workloads using a specific UID will need to pass UID as extra variable

## Alternatives

### Change ownership of files after state transfer

In this solution, the ownership of files will be updated after files are transferred to the destination namespace. 

A new subcommand `chown` will be added to the `crane` CLI:

```sh
[user@envision]$ crane help
[...]

Available Commands:
  [...]
  chown   Change ownership of transferred files in a namespace 
```

The subcommand will take following arguments:

```sh
[user@envision]$ crane chown help
Usage:
  crane chown [flags]

Flags:
      --context string     Name of the context in current kubeconfig
      --gid int            Custom GID to own files with
      --namespace string   Namespace in which ownership needs to be fixed
      --timeout int        Timeout in seconds for runner pod (default 300)
      --uid int            Custom UID to own files with
```

The `--namespace` and `--context` arguments will be set as required. All other options will be optional. 

The subcommand will work in following steps:

- At first, all PVC resources present in the namespace will be discovered

- UID and GID values be discovered:
  - If `--gid` and `--uid` options are NOT set, it will assume that it's running in OpenShift environment and it will parse the namespace annotations to find out UID and GID values. If a Container does not explicitely set `runAsUser` or `runAsGroup` field in its SecurityContext, OpenShift uses first UID and GID values from the range assigned to the Namespace. The subcommand will use the first UID and GID values as well.
  - If `--gid` and `--uid` options are set, the provided values will be used as UID and GIDs

- A temporary Pod will be created which will attach all PVCs in the Namespace at a known path. The starting command of the Pod will be:

	```yaml
  - command:
    - "/bin/bash",
    - "-c",
    - fmt.Sprintf("/usr/bin/timeout --signal=SIGINT %d /usr/bin/chown -R %s:%s /mnt/", c.Timeout, uid, gid)
	```
	The `uid`, `gid` values will be the ones discovered in the previous step or provided by the user.

- Process will wait until the Pod reaches completion. If the Pod succeeds, the temporary pod will be deleted. Otherwise, it will be left behind for debugging purposes. 

`--timeout` argument allows setting a timeout value on the chown process in the Pod. If only one of the `--gid` and `--uid` options are specified, the respective ID of the files will be changed keeping the other one as-is. If both values are NOT specified, OpenShift environment will be assumed and the values will be read from the annotations. If the values are not found in the namespace, command will fail.

Following Pros/Cons were considered before ruling out the alternative:
#### Complexity

Adding a new subcommand takes more efforts than simply adding a new option to `rsync` subcommand. Also, creating a new temporary Pod implies added surface for debugging that Pod. Therefore, final solution is better than the alternative in terms of complexity.

#### Speed/Performance

By fixing ownership during Rsync transfers ensures that the total time it takes to fix ownership is overlapped with rsync transfer time. In case of Staging/Cutover setting, this time is also spread across multiple runs making it even faster during the final Cutover. This is similar to the [performance issue](https://github.com/kubernetes/kubernetes/issues/69699) in `kubelet` due to automatic `chown` of volumes at mount time.

#### Quiescing

Separate Pod for running `chown` requires workloads in the destination namespace be quiesced down. Therefore, it must be run in a specific order during migration. When running as a standalone command, the workloads need to be quiesced down before running the chown subcommand.