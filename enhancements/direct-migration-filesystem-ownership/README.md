# Filesystem ownership reconciliation with direct volume migrations

While working the [direct volume migration] feature, it became apparent that
is can be complex to manage the owner and group permissions of volumes between
the source, replication pod (rsync pod), and target destination pods. This is
especially true when considering the many different provisioners that potentially
behave differently with regard to security contexts (some respect the fsGroup,
while some unusual ones do not).

## Related issues:

Bugzilla: https://bugzilla.redhat.com/show_bug.cgi?id=1908214

## Observed issue

With the original implementation of DVM, the rsync transfer pod was run
as a privilieged pod, meaning it executed as root. By launching the pod with
these properties, it mutated the source volumes's filesystem permission bits
to **no longer match those of the original** (in this case, obviously it would
not be able to access root permissioned files). This caused the source workload
to crashloop with permission errors while trying to access its volume's data.

## BZ Root Cause Analysis

In order to root cause this bug the following test set up was used:

1. A dummy pod was setup to replace the rsync client pod. This pod just sleeps
2. On gp2 even with a dummy pod, the application broke. This strongly suggests that mounting gp2 volumes twice is the problem, nothing to do with rsync.
3. There is a SCC field called fsGroup. More info here https://docs.openshift.com/container-platform/4.6/storage/persistent_storage/persistent-storage-nfs.html#storage-persistent-storage-nfs-group-ids_persistent-storage-nfs. It was observed that the application pod has fsGroup of 1000570000, whereas the rsync client pod(and the dummy pod) did not. The fsGroup is used to isolate block storages from one container to another and the host. It was immediately clear that we need the same fsGroup as the application pod in rsync pod and dummy pod.
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

The group, in this case, is the same as the application user, hence the application pod does not break.)

## Proposed solution

Before starting the rsync client pod, we check if the pvc’s are attached to
any pods in the namespace.

* If any pvc is attached to one or more pods:
  * Attached to one pod, get its fsGroup and use the same fsGroup for the rsync
  client pods and explicitly set the node to match the current pod that the
  volume is attached to.
  * Attached to more than one pod, raise a warning because this could lead to
  indeterminate behavior. Pick any of the pods’s fsGroup at random and use it
  for the rsync client pod while explicitly setting the node to match the picked pod’s.
  * If pvc’s are attached to pods and none of them have a scc with fsgroup,
  don't use any fsgroup. This will adopt the namespaces default security context.

* If not attached to any pod:
  * Run without explicit fsgroup, adopting default security context.

Along with the fsgroup, we will also have to work with supplemental group ids,
which is the same concept as fsgroup but for shared storages like NFS and glusterfs.

## Edge cases

We expect that there are edge cases where the fsgroup value will not be discoverable
and the default security context is not sufficient for access, which would cause
the rsync transfer pod to throw file permission errors when attempting to transfer
the data. For this reason, we'd like to build an "escape hatch" into the operator
that allows admins to **explicitly opt in** to running the transfer pod as
privileged. This option should only be used in the event that correct group
permissions are not posssible as a method to unblock a migration. Proper caution
should be taken in this case by the admin to ensure unauthorized access to the
transfer pod is not possible while the transfer is underway.
