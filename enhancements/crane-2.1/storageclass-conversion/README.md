---
title: storageclass-conversion
authors:
  - "@stillalearner"
reviewers:
  - "@rromannissen"
  - "@dymurray"
  - "@jwmatthews"
  - "@shawn-hurley"
  - "@pranavgaikwad"
  - "@aufi"
approvers:
  - "@istein1"
  - "@stillalearner"
creation-date: 2026-07-16
last-updated: 2026-07-16
status: implementable
see-also:
  - "https://github.com/migtools/crane/issues/655"

superseded-by: []
---

# Intra-Cluster StorageClass Conversion

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. Should workload reference updates be handled by a new plugin or existing optional flags?
    * **Decision:** Use existing `pvc-rename-map` optional flag in KubernetesPlugin. The flag mechanism may evolve as part of the plugins system enhancement (parametrized custom stages), but the underlying plugin logic stays the same.

2. Should the command support batch conversion of multiple PVCs in a single run?
    * **Decision:** Not as a new command. The user runs `transfer-pvc` once per PVC, then a single export/transform/apply pass handles all workload reference updates in the manifests.

3. Should old PVCs be cleaned up automatically?
    * **Decision:** No. Old PVCs remain on the cluster. The user verifies data integrity and deletes manually.

4. How should StatefulSet immutable volumeClaimTemplates be handled?
    * **Decision:** The transform pipeline generates the correct manifest with renamed templates. The user handles the delete+recreate of the StatefulSet manually. Crane does not do live workload patching.

## Summary

Crane currently has no mechanism for converting PVCs from one StorageClass to another within the same cluster. The only existing option is `--dest-storage-class` on `crane transfer-pvc`, but the command rejects same-cluster transfers.

This enhancement extends the existing crane pipeline to support intra-cluster StorageClass conversion by:

1. **Extending `transfer-pvc`** to allow same-cluster transfers — creating a new PVC with the target StorageClass and copying data via rsync within the same cluster
2. **Using the existing `pvc-rename-map`** in the KubernetesPlugin to generate updated manifests with renamed PVC references across all workload types

No new commands are introduced. The feature composes existing crane primitives (`transfer-pvc`, `export`, `transform`, `apply`) following crane's Unix-philosophy pipeline.

## Motivation

StorageClass changes within a single cluster are a common operational need:

* **Cloud provider upgrades:** `gp2` → `gp3`, `managed-standard` → `managed-premium`
* **Storage system migration:** GlusterFS decommissioned, replaced by Ceph/OCS
* **Performance tiering:** Moving workloads between fast/slow storage tiers
* **Compliance/policy:** Organization mandates encryption-at-rest via a specific StorageClass

Since Kubernetes PVC `spec.storageClassName` is immutable after creation, converting requires: creating a new PVC with the target StorageClass, copying data, and updating workload references — a multi-step process that is error-prone when done manually.

### Goals

* **No new commands:** Extend existing `transfer-pvc` and leverage existing `pvc-rename-map` in transform
* **Composable pipeline:** Data transfer and manifest generation are separate steps that the user composes
* **Non-destructive:** Crane generates updated manifests on disk — the user reviews and applies
* **No live workload patching:** Crane does not modify running workloads directly
* **Data integrity:** Transfer data via rsync with optional checksum verification
* **OCP and K8s:** Work on both OpenShift (Route endpoint) and vanilla Kubernetes (Ingress endpoint)
* **Non-admin support:** Work with namespace-admin RBAC

### Non-Goals

* **Automatic workload patching:** Crane generates manifests; the user applies them
* **Automatic quiesce/restore:** User scales down workloads before transfer and scales up after applying
* **StatefulSet delete+recreate automation:** Crane generates the correct manifest; the user handles the immutable field update
* **Cross-cluster StorageClass change:** Already handled by the existing `transfer-pvc` cross-cluster flow
* **Snapshot-based copy:** Only rsync-based data transfer

## Proposal

### User Stories

#### Story 1: Single PVC Conversion

A developer needs to convert a PVC from one StorageClass to another within the same cluster. They scale down the workload, run `transfer-pvc` with same source and destination context to copy data to a new PVC, then use the export/transform/apply pipeline to generate updated workload manifests and apply them.

#### Story 2: Multiple PVC Conversion

An operator needs to convert multiple PVCs in a namespace. They run `transfer-pvc` once per PVC, then run a single export/transform/apply pass with all rename mappings to generate updated manifests for all workloads at once.

#### Story 3: Non-Admin User

A namespace-admin converts PVCs using `transfer-pvc` with same-cluster contexts. The command gracefully handles Forbidden responses for cluster-scoped operations.

### Workflow

#### Simple case: Deployment with one PVC

```bash
# 1. Quiesce workload
kubectl scale deploy webapp --replicas=0 -n myapp

# 2. Transfer data to new PVC with new StorageClass
crane transfer-pvc \
  --source-context mycluster --destination-context mycluster \
  --pvc-name "mysql-data:mysql-data-new" \
  --pvc-namespace myapp \
  --dest-storage-class gp3 \
  --endpoint route

# 3. Generate updated manifests
crane export --context mycluster --namespace myapp --export-dir ./export
crane transform --export-dir ./export --transform-dir ./transform \
  --optional-flags '{"pvc-rename-map": "mysql-data=mysql-data-new"}'
crane apply --export-dir ./export --transform-dir ./transform --output-dir ./output

# 4. Review and apply
kubectl apply -f ./output/output.yaml -n myapp
```

#### StatefulSet case

```bash
# 1. Scale StatefulSet to 0
kubectl scale sts redis --replicas=0 -n myapp

# 2. Transfer each PVC
crane transfer-pvc --source-context ctx --destination-context ctx \
  --pvc-name "data-redis-0:data-new-redis-0" --pvc-namespace myapp \
  --dest-storage-class gp3 --endpoint route
crane transfer-pvc --source-context ctx --destination-context ctx \
  --pvc-name "data-redis-1:data-new-redis-1" --pvc-namespace myapp \
  --dest-storage-class gp3 --endpoint route

# 3. Generate manifests with renamed volumeClaimTemplates
crane export --context ctx --namespace myapp --export-dir ./export
crane transform --export-dir ./export --transform-dir ./transform \
  --optional-flags '{"pvc-rename-map": "data=data-new"}'
crane apply --export-dir ./export --transform-dir ./transform --output-dir ./output

# 4. Delete old StatefulSet (preserve PVCs) and apply new manifest
kubectl delete sts redis --cascade=orphan -n myapp
kubectl apply -f ./output/output.yaml -n myapp
```

### Implementation Details

#### Part 1: Extend `transfer-pvc` for same-cluster

The current `transfer-pvc` rejects same-cluster transfers. Four changes are needed:

1. **Remove same-cluster rejection** — allow `sourceContext.Cluster == destinationContext.Cluster`
2. **Fix cert secret naming for same-namespace** — when source and destination are in the same namespace, copy the server's TLS cert secret under the name the client expects (different PVC names produce different secret names)
3. **Split pod labels for same-namespace** — add a `role` label to distinguish server and client pods, so the log reader finds the correct pod
4. **Dual garbage collection** — clean up both server-side and client-side resources when using split labels

#### Part 2: Workload manifest updates via existing `pvc-rename-map`

The KubernetesPlugin in crane-lib already supports `pvc-rename-map` which generates JSONPatch operations to rename PVC references. It covers all workload types:

* Deployments, DaemonSets, ReplicaSets, ReplicationControllers, Jobs, CronJobs, Pods — volume claim name references
* StatefulSets — both pod spec volumes and `volumeClaimTemplates` names

No plugin changes needed. The user passes the rename mapping via `--optional-flags` (or the future parametrized stage flag when the plugins enhancement lands).

Note: `pvc-rename-map` renames the volumeClaimTemplate name but does not update its `storageClassName`. This is acceptable because the new PVC already has the correct StorageClass from `transfer-pvc`. However, for StatefulSets, new replicas scaled up in the future would use the old StorageClass from the template. A `storage-class-map` optional flag could be added to KubernetesPlugin in a future iteration to address this.

### Security, Risks, and Mitigations

**Data in transit:** All data transfer goes through stunnel (TLS 1.3) with mutual certificate verification, even for same-cluster transfers.

**UID handling:** On OCP, the command reads the namespace's UID range annotation and runs rsync pods with the correct UID. On vanilla K8s, it reads the workload's security context.

**Non-destructive:** Crane generates manifests on disk. The user reviews before applying. No live workload mutations by crane.

**Old PVCs preserved:** The source PVC is not deleted. If the conversion fails or the user is unhappy with the result, the original data is still available.

## Design Details

### Test Plan

Unit tests for the transfer-pvc changes and integration tests covering the full pipeline on both minikube and OCP clusters.

### Upgrade / Downgrade Strategy

* **Additive change:** Removing the same-cluster check and adding intra-cluster support does not affect existing cross-cluster transfer behavior.
* **No new commands or flags:** The feature uses existing flags (`--source-context`, `--destination-context`, `--dest-storage-class`, `--optional-flags`).
* **Backwards compatible:** Existing scripts and CI pipelines that use `transfer-pvc` for cross-cluster transfers are unaffected.

## Implementation History

* [#655](https://github.com/migtools/crane/issues/655) — Parent feature issue
* [#656](https://github.com/migtools/crane/issues/656) — Convert a single PVC to a different StorageClass
* [#657](https://github.com/migtools/crane/issues/657) — Convert multiple PVCs via a plan file
* [#658](https://github.com/migtools/crane/issues/658) — Automatically update workload references after conversion
* [#659](https://github.com/migtools/crane/issues/659) — Handle StatefulSet volumeClaimTemplate conversion
* [#660](https://github.com/migtools/crane/issues/660) — Support non-admin users for StorageClass conversion
* [#661](https://github.com/migtools/crane/issues/661) — E2E test coverage for StorageClass conversion

## Drawbacks

* **Multiple manual steps:** The user must run `transfer-pvc` per PVC, then the export/transform/apply pipeline, then `kubectl apply`. This is by design (composability and reviewability) but requires more user effort than a single-command approach.
* **StatefulSet requires manual delete+recreate:** Since `kubectl apply` cannot change immutable `volumeClaimTemplates`, the user must delete and recreate the StatefulSet manually. The generated manifest shows the correct target state.
* **No automatic quiesce:** The user must manually scale down workloads before transfer to prevent data loss from writes during rsync.
* **`pvc-rename-map` does not update StorageClass in templates:** For StatefulSets, new replicas scaled up after conversion would use the old StorageClass from the template unless a future `storage-class-map` flag is added.

## Alternatives

1. **New `crane convert-storage` command:** A monolithic command that handles data transfer, workload patching, quiesce, and restore in one step. Rejected because live workload patching is not consistent with crane's non-destructive, file-based pipeline philosophy.
2. **New transform plugin for StorageClass mapping:** A dedicated plugin that remaps `storageClassName` in PVC manifests and volumeClaimTemplates. Not needed for the core use case — `pvc-rename-map` covers workload reference updates, and the new PVC's StorageClass is set by `transfer-pvc` at creation time.
3. **Operator-based approach:** Deploy a controller that orchestrates the full conversion lifecycle. Rejected for crane users who want a lightweight CLI tool without operator dependencies.

## Infrastructure Needed

No new repositories required. Changes land in [migtools/crane](https://github.com/migtools/crane).
