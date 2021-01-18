# Filesystem ownership reconciliation with direct volume migrations

While working the [direct volume migration](https://issues.redhat.com/browse/MIG-284)
feature, it became apparent that it can be complex to manage the owner and
group permissions of volumes between
the source, replication pod (rsync pod), and target destination pods. This is
especially true when considering the different provisioners that potentially
behave differently with regard to security contexts (some respect the fsGroup,
while some unusual ones do not).

## Related issues:

Bugzilla: https://bugzilla.redhat.com/show_bug.cgi?id=1908214

## Observed issue

With the original implementation of DVM, the rsync transfer pod was run
as a privileged pod. A privileged pod is different from root. 

By default, a container is not allowed to access any devices on the host, but
a "privileged" container is given access to all devices on the host. This 
allows the container nearly all the same access as processes running on the 
host. This is useful for containers that want to use linux capabilities like
manipulating the network stack and accessing devices. [Source](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#privileged). 

In DVM controller the replication pod (running in the privileged mode and with
user ID 0, i.e. root) would mount the source PVCs, this presented two
problems: 

1. Block storages like gp2 were not meant for being shared. When two pods
 mount the same PVC, kubelet essentially re-writes the ownership
 permissions. Because the rsync pod ran as root on the source, it mutated
 the source volume's filesystem permission bits to be owned by root. The user
 running the actual application can no longer read/write to a filesystem
 owned by root and it would crashloop. This can be solved by using
  source applications fsGroup, more info later in the doc. 

2. Because the Pod is run as privileged in the user namespace, this is a
  significant security hole. Any user (non-admin) with exec access to the Pod
  would have the ability to exec into it and access the node's filesystem, 
  therefore gaining access to sensitive credentials that could be used for
  privilege escalation.

## BZ Root Cause Analysis

In order to root cause this bug the following test set up was used:

1. A dummy pod was setup to replace the rsync client pod. This pod just sleeps
2. On gp2 even with a dummy pod, the application broke. This strongly suggests
   that mounting gp2 volumes twice is the problem, nothing to do with rsync.
3. There is a SCC field called fsGroup. More info [here](https://docs.openshift.com/container-platform/4.6/storage/persistent_storage/persistent-storage-nfs.html#storage-persistent-storage-nfs-group-ids_persistent-storage-nfs).
 It was observed that the application pod has fsGroup of 1000570000, 
 whereas the rsync client pod(and the dummy pod) did not. The fsGroup is used 
 to isolate block storages from one container to another and the host. It was
  immediately clear that we need the same fsGroup as the application pod in
  rsync pod and dummy pod.
4. The dummy pod was then started with the same scc fsgroup as the application pod.
5. The application did not break.

Observations:

When the application is broken (this was taken from the node where the volume is mounted.

```
sh-4.2# getfacl vol-0d96f2165bbe22a64/
# file: vol-0d96f2165bbe22a64/
# owner: root
# group: 1000570000
# flags: -s-
user::rwx
group::rwx
other::r-x
```

```
sh-4.2# cd vol-0d96f2165bbe22a64/
sh-4.2# getfacl userdata/
# file: userdata/
# owner: 1000570000
# group: 1000570000
# flags: -s-
user::rwx     ← note the permission is only to the user
group::---
other::---
```

The rsync client pod is running as root user and the application pod is running
as 1000570000 user. The application pod cannot see contents owned by the root user

When the application is not broken:

```
sh-4.2# getfacl vol-0d96f2165bbe22a64/
# file: vol-0d96f2165bbe22a64/
# owner: root
# group: 1000570000
# flags: -s-
user::rwx
group::rwx
other::r-x
```

```
sh-4.2# getfacl userdata/
# file: userdata/
# owner: 1000570000
# group: 1000570000
# flags: -s-
user::rwx                     ← Note both the user and the group has the permissions
group::rwx
other::---
```

The group, in this case, is the same as the application user, hence the 
application pod does not break.)

## Proposed solution

In order to solve both problems, we added two things to the controller:
1. Use the same fsGroup, supplemental-groups and seLinuxContext as the
   application pod.
2. Run the rsync transfer pod with anyuid SCC and removed the container's
   privileged: true flag. This would make sure that we continue running as
   user 0 i.e. root, but dont have access to the root devices for privilege
   escalations. Note, the root user is required to carry over the file
   permissions exactly as is from the source PVC to destination.

Before starting the rsync client pod, we check if the pvc’s are attached to
any pods in the namespace.

I'm going to use `securityContext` in the following as a shorthand for
`securityContext.{fsGroup,supplementalGroups,seLinuxOptions}`

* If any pvc is attached to one or more pods:
  * Attached to one pod, get its `securityContext` and use the same `securityContext` for the rsync
  client pods and explicitly set the node to match the current pod that the
  volume is attached to.
  * Attached to more than one pod, raise a warning because this could lead to
  indeterminate behavior. Pick any of the pods’s `securityContext` at random and use it
  for the rsync client pod while explicitly setting the node to match the picked pod’s.
  * If pvc’s are attached to pods and none of them have a `securityContext`.
  This will adopt the namespaces default security context.

* If not attached to any pod:
  * Run without explicit `securityContext`, adopting default `securityContext`.

### privileged vs. anyuid

Initially we ran the source transfer Pod as privileged, which provides a number
of extra "capabilities", inluding access to host devices. It became clear this
could be used to escalate privleges beyond what an admin over a namespace should
have. Instead, `anyuid` should provide the ability to run with `uid` 0, giving
us enough privileges to satisfy our requirement for having sufficient privileges
to exactly clone all filesystem permission attributes to the destination without
suffering privilege issues.

Notable differences between `anyuid` and `privileged` are:

1) The capabilities that are afforded to `privileged` that will mount the problematic
dev from the node is not present with `anyuid`.

2) `anyuid` is "constrained" by SELinux, while `privileged` is "unconstrained".
This adds the an additional layer of restriction to prevent unauthorized file access.

As an additional improvement beyond `anyuid`, we can define a custom SCC that
drops all capabilities and only whitelists the specific privileges we require.

## Edge cases

In the event that the transfer pod running as the anyuid is not sufficient,
We'd like to build an "escape hatch" into the operator
that allows admins to **explicitly opt in** to running the transfer pod as
privileged. Proper caution should be taken in this case by the admin to ensure
unauthorized access to the transfer pod is not possible while the transfer is underway.
