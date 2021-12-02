---
title: dvm-permissions-handling
authors:
  - "@pranavgaikwad"
reviewers:
  - "@alaypatel07"
  - "@jmontleon"
  - "@shawn-hurley"
  - "@sseago"
approvers:
  - "@alaypatel07"
  - "@jmontleon"
  - "@shawn-hurley"
  - "@sseago"
creation-date: 2021-11-17
last-updated: 2021-12-02
status: implementable
see-also:
  - "N/A"  
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# DVM Permissions Handling

When migrating persistent volume data from the source to the destination cluster, MTC preserves the ownership of the data. It is assumed that the applications will continue working without needing changes in file ownership information. In some cases, the users may be interested in intentionally changing file ownership during migration. This enhancement discusses scenarios where change in file ownership may be needed and proposes solutions to allow changing file ownership automatically in those scenarios.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

In State Migration, the destination PVC resources need not be created by MTC. The users can provision destination PVCs prior to migration and migrate data to them. The destination _Namespace_ may exist prior to migration. OpenShift uses uid-range annotation on the namespace to control file ownership of the data. This annotation needs to be preserved in the destination namespace for the applications to continue working without needing changes in file ownership. If the namespaces are created prior to migration, it is possible that these annotations are different in the destination cluster than the source. In such cases, the applications will not be able to access the files post migration as the files will be owned by UID of the source namespace. This enhancement proposes to add an optional method to atomatically change file ownership based on UID ranges present on the destination namespace. Note that this enhancement only proposes to change file ownership and not the file permissions. The file permissions will still be preserved as they were before. 

In addition to the above problem, there is one more minor problem that this enhancement proposes a solution to. The recommended way of accessing shared storages in OpenShift is by using Supplemental Groups. If the source or destination PVCs are backed by shared storage, Rsync Pods won't be able to access files even though it's running as root. This is because MTC does not use _Supplemental Groups_ on either of Rsync Pods.

## Motivation

The users may create _Namespace_ and _PersistentVolumeClaim_ resources prior to running a state migration. It is possible that the UID range assigned to destination namespaces are different than the source . In such cases, the migrated files will be owned by different users than that of the source. MTC does not provide an option to automatically change the ownership of files. For large-scale migrations it can be a cumbersome process to change the file ownership manually. MTC can use some of Rsync's capabilities to automatically change file ownership during data transfer. OpenShift already assigns right UID/GID to container processes based on the namespace annotations. Rsync Server process can piggyback on this ability and write files to destination volumes with the UID/GID assigned to the process. It will enable users to migrate data in such scenarios at a scale without needing manual ownership changes post migration.

### Goals

- Add a way to change ownership of files if the assigned uid-range of the destination namespaces differ from that of the source so that applications can continue accessing files in the destination cluster

- Allow users to provide a custom UID/GID pair per namespace to override the default UID/GID used on the Rsync Pod

- Allow users to provide a list of supplemental groups so that the Rsync Pods can access PVCs that are backed by a shared storage

### Non-Goals

- Change the default behavior of MTC data transfer which is to preserve file ownership

- Alter file permissions

## Proposal

When a Migration Plan is created, the controller will scan the destination cluster and check whether destination namespaces in the plan already exist in the cluster. If they exist, the controller will compare `openshift.io/sa.scc.uid-range` and `openshift.io/sa.scc.supplemental-groups` annotations with their respective source namespaces. If there exists a namespace which has a different value in any one of the two annotations, the controller will add a _Warning_ condition on the Migration Plan indicating that there may be access issues in the destination cluster. 

For the controller to take any further action, the feature needs to be enabled through a _MigrationController_ flag. By default, the controller will not change file ownership and will continue to work as before i.e. by preserving file ownership and permissions. See [Upgrade/Downgrade Section](#upgrade-downgrade-strategy) for more details on the flag.

If the feature is enabled, DVM will not preserve file ownership information. During the transfer, DVM will use Rsync Options and Rsync Pod's Security Context in conjuction to write the files with the correct UID in the destination cluster.

### Changing ownership

- The Rsync Server Pod will be forced to use the UID of the destination namespace. To enable that, we will add a new SCC which will be a clone of existing `rsync-anyuid` SCC with some changes. The changes are explained in [SCC changes section](#scc).

- Rsync options `--owner`, `--group` and `--devices` will be removed from the default Rsync command options. `--owner` & `--group` options preserve UID/GID of the files. By removing these options, UID/GID of the files will not be preserved. Since Rsync Server Pod will use the UID/GID assigned to the process, the files will be written using those values. `--devices` option transfers device files, but since the Rsync Pod will not be running as _root_ anymore, that option will be removed. All other options will remain as they existed before. 

- No changes will be made in security context of Rsync Client Pod (source side). It will continue to use `rsync-anyuid` SCC as it before. 

#### SCC

A new scc `rsync-restricted-uid` will be added to address the needs of this use-case. For the most part, this is a clone of `rsync-anyuid` SCC except following changes:

1. _fsGroup_ field of the SCC will be set to _MustRunAs_. It controls the GID of the mounted volume. It is only applicable for certain block storages. See [fsGroup](#fsGroup) for more details.

2. _runAsUser_ field of the SCC will be set to _MustRunAsRange_. This will force Rsync process to run as UID from the namespace's UID range.

3. Capability _SETGID_ will be added to _defaultAddCapabilities_ to allow Rsync to write the GID of the files. This is needed as Rsync will not run as root. 

#### Rsync Server Pod SecurityContext

For `rsync-restricted-uid` SCC to be correctly picked up, Rsync Server Pod's SecurityContext will changed as follows:

1. _SETGID_ will be added to _ADD_ Capabilities. This will allow Rsync Pod to set GID of the files.

2. _runAsUser_ field will be removed from the security context. This will force OpenShift to automatically use the UID from the namespace range as per `rsync-restricted-uid` SCC.


#### fsGroup

_fsGroup_ is only respected when the volume has `fsType` specified and its mode is set to RWO. In k8s 1.19+, there is an additional field introduced for CSI drivers which defines a specific policy for setting the GID. If the _fsGroup_ is respected, OpenShift will automatically apply the GID to the mounted volume and set the SETGID bit so that all future files created in the volume are owned by that GID. 

If its not respected and if the applications require a specific GID on the file, then the `runAsGroup` field on the Rsync Server container must be set. We won't need this in most cases as the Rsync process will start as the GID assigned by OpenShift (usually 0). The UID is a part of that group. But if the users are interested in owning the files with a specific UID/GID, we will provide an optional variable in _MigrationController_ which will take a list of namespaces along with their custom UID/GID and we will add them on the Rsync Pod's _SecurityContext_. In that case, the Pods will be assigned `rsync-anyuid` SCC instead of its restricted counterpart. See [Override GID/UID](#override-uid-gid). This will be only required in special cases when applications care about setting a specific group ID on files.
#### Shared Storage

For shared storages, OpenShift controls access through _supplementalGroups_ field on Pod spec. If the PVCs in the destination cluster are backed by a shared storage, the users will provide a list of _supplementalGroups_ values through _MigrationController_ CR. This list will be added to Rsync Server Pod's _securityContext_ field. As a result, Rsync server pod will be able to access the destination volume as the GID from this list.
  
## Design Details

### Test Plan


| Source Storage 	| Destination Storage 	| Test Cases                                                                                                                                                                                                                                                  	|
|:--------------:	|:-------------------:	|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------	|
|      block     	|        block        	| - Migrate to a namespace that has different UID/GID range with the feature enabled - Migrate to a namespace that has same UID/GID range with the feature disabled                                                                                           	|
|     shared     	|        block        	| - Migrate to a namespace that has different UID/GID range with the feature enabled & source supplemental groups set - Migrate to a namespace that has same UID/GID range with the feature disabled & source supplemental groups set                         	|
|      block     	|        shared       	| - Migrate to a namespace that has different UID/GID range with the feature enabled & destination supplemental groups set - Migrate to a namespace that has same UID/GID range with the feature disabled & destination supplemental groups set               	|
|     shared     	|        shared       	| - Migrate to a namespace that has different UID/GID range with the feature enabled & both source/target supplemental groups set - Migrate to a namespace that has same UID/GID range with the feature disabled & both source/target supplemental groups set 	|

### Upgrade / Downgrade Strategy

We will introduce 3 new flags in _MigrationController_ CR: 

|                               	|  Type 	| Required? 	| Default 	|                Description               	|
|:-----------------------------:	|:-----:	|:---------:	|:-------:	|:----------------------------------------:	|
| pv_transfer_auto_owner_change 	|  bool 	|     N     	|  false  	|     Enable automatic ownership change    	|
|   source_supplemental_groups  	| []int 	|     N     	|  empty  	| Supplemental groups for source Rsync Pod 	|
|   target_supplemental_groups  	| []int 	|     N     	|  empty  	| Supplemental groups for target Rsync Pod 	|

By default, none of the above flags will be set. The values will only be needed in special cases either when permission change is needed or when one of the source/target side is using a shared storage.

The supplemental groups can be set even when permission change is not needed. It will preserve the file permissions but allow Rsync Pod to access the volume.

### Override GID/UID

Additionally, a custom UID/GID can be provided per namespace:

```
target_namespace_custom_id_mapping:
- ns-A/<UID>:<GID>
```

If a mapping is present for a target namespace, the custom ids will be added to the Rsync Server Pod.

## Alternatives

Instead of changing ownership through Rsync, the ownership can be altered post migration by running `chown` recursively on the volume data. This would be an additional step in the migration either achieved through a Hook or a new task in the itinerary itself. Some of the main challenges that discourage from adopting this alternative are:

- If hook is used, additional data needs to be passed to the hook (essentially coordinates of destination volumes along with ownership information of each of them). Irrespective of whether to do it through Hooks or Itinerary task, additional Pods are needed to mount the target volumes and run `chown` command. There will be added burden of handling failures. Existing Rsync Server pod cannot be used to run `chown` for main reason that there it cannot differentiate when a particular Rsync transfer is complete.

- Recursive traversing of volume data is a compute and time intensive process. It will increase the overall migration time significantly. On the other hand, when Rsync changes ownership, that time and compute is overlapped with data transfer itself.
## Infrastructure Needed [optional]

For block storages, any of the EBS or Ceph volumes can be used to test the feature.

For shared storages, Gluster volumes in RWX mode or NFS volumes are needed. The shared volume approach in this enhancement is verified on NFS volumes.
