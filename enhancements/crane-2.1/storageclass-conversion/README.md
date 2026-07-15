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
approvers:
  - "@istein1"
  - "@stillalearner"
creation-date: 2026-07-14
last-updated: 2026-07-14
status: implementable
see-also:
  - "https://github.com/migtools/crane/issues/655"
  - "https://github.com/migtools/crane/issues/319"
replaces: []
superseded-by: []
---

# Intra-Cluster StorageClass Conversion

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. Should the command support access mode changes (e.g., RWO -> RWX) as part of conversion?
    * **Decision:** Yes, via optional `--target-access-mode` flag and the plan file's `accessModes` field. Not enforced — user is responsible for ensuring target SC supports the requested mode.

2. Should the `plan` subcommand auto-detect StatefulSet PVC naming and generate correct target names automatically?
    * **Decision:** Yes. Both the **plan generation** and **swap phase** auto-detect StatefulSet volumeClaimTemplate PVCs using regex `^<templateName>-<stsName>-(\d+)$`. The plan subcommand generates correct target names (`<templateName>-mig-<suffix>-<stsName>-<ordinal>`), and the swap phase performs the delete+recreate dance automatically.

3. Should old PVCs be cleaned up automatically or left for the user?
    * **Decision:** Old PVCs are labeled (`crane.konveyor.io/migrated-to`) but never deleted. The user verifies data integrity and deletes manually.

## Summary

Crane currently has no mechanism for converting PVCs from one StorageClass to another within the same cluster. The only existing option is the `--dest-storage-class` flag on `crane transfer-pvc`, which is designed for cross-cluster migration — not intra-cluster StorageClass changes.

This enhancement adds a new **`crane convert-storage`** command that provides MTC-equivalent `StorageConversionPlan` functionality as a CLI tool. The command creates a new PVC with the target StorageClass, transfers data via rsync (reusing the pvc-transfer library), automatically swaps workload references (Deployments, StatefulSets, DaemonSets, CronJobs, Jobs, ReplicaSets), and preserves the old PVC for rollback safety.

The command supports two modes:
* **Single-PVC mode:** Convert one PVC via CLI flags
* **Batch mode:** Generate an editable YAML plan file, then execute it to convert multiple PVCs

## Motivation

StorageClass changes within a single cluster are a common operational need:

* **Cloud provider upgrades:** AWS `gp2` -> `gp3` (better performance, lower cost), Azure `managed-standard` -> `managed-premium`
* **Storage system migration:** GlusterFS decommissioned, replaced by Ceph/OCS
* **Performance tiering:** Moving workloads between fast/slow storage tiers
* **Compliance/policy:** Organization mandates encryption-at-rest via a specific StorageClass

MTC provides this capability through `StorageConversionPlan` in `mig-controller`, but it requires the full MTC operator stack (MigPlan CR, MigMigration CR, MigCluster, controllers, UI). Crane users need a lightweight CLI alternative that fits the Unix-philosophy pipeline.

Since Kubernetes PVC `spec.storageClassName` is immutable after creation, converting a PVC's StorageClass requires creating a new PVC, copying data, and updating all workload references — a multi-step process that is error-prone when done manually.

### Goals

* **Single-command conversion:** Convert a PVC from one StorageClass to another with one command, including data transfer and workload reference swap.
* **Batch support:** Convert multiple PVCs in a namespace via an editable YAML plan file.
* **Non-admin support:** Work with namespace-admin RBAC — no cluster-admin required.
* **StatefulSet support:** Handle immutable `volumeClaimTemplates` via delete+recreate dance, matching MTC's approach.
* **Data integrity:** Transfer data via rsync with optional checksum verification (`--verify`).
* **Rollback safety:** Preserve old PVCs (labeled, not deleted) so the user can verify and roll back if needed.
* **OCP and K8s:** Work on both OpenShift (Route endpoint, SCC UID detection) and vanilla Kubernetes (Ingress endpoint).
* **MTC feature parity:** Cover the core `StorageConversionPlan` functionality from `mig-controller`.

### Non-Goals

* **Cross-cluster StorageClass change:** Already handled by `crane transfer-pvc --dest-storage-class`. This command is intra-cluster only.
* **Snapshot-based copy:** Only rsync-based data transfer. CSI snapshot copy is out of scope.
* **PV move/re-bind:** Only copy action. NFS PV re-bind (MTC's `move` action) is out of scope.
* **Automatic rollback:** The command labels old PVCs but does not provide an automated rollback mechanism.
* **VirtualMachine (KubeVirt) swap:** Not in initial scope.
* **Cross-namespace conversion:** Source and destination PVC must be in the same namespace.

## Proposal

### User Stories

#### Story 1: Single PVC Conversion

A developer needs to convert one specific PVC to a different StorageClass. They run a single command specifying the PVC name and target StorageClass. The command creates the new PVC, transfers data, swaps the workload reference, and labels the old PVC.

#### Story 2: Batch Conversion of Multiple PVCs

An operator needs to convert all PVCs in a namespace from one StorageClass to another. They generate a plan, review and edit it, and execute. All PVCs are converted, workload references swapped, old PVCs preserved.

#### Story 3: StatefulSet Storage Upgrade

A team runs a multi-replica StatefulSet with volumeClaimTemplates. They generate a plan, edit target names to follow the StatefulSet naming convention, and execute. The swap phase auto-detects the StatefulSet volumeClaimTemplate match and performs the delete+recreate dance to update the immutable template field.

#### Story 4: Non-Admin User

A namespace-admin (not cluster-admin) converts PVCs. The command gracefully handles Forbidden responses for cluster-scoped operations (StorageClass listing) and falls back to user-provided values.

#### Story 5: Selective Conversion via Plan

A namespace has multiple PVCs but only some need conversion. The user generates a plan, sets `action: skip` on PVCs that should remain unchanged, and executes. Only the non-skipped PVCs are converted.

### Implementation Details/Notes/Constraints

#### Command Structure

New command at `cmd/convert-storage/`, following the existing Cobra pattern (Complete/Validate/Run).

**Root command (`crane convert-storage`):**

| Flag | Type | Required | Description |
|------|------|----------|-------------|
| `--context` | string | yes* | Cluster kubeconfig context |
| `--pvc-name` | string | yes* | Source PVC name |
| `--pvc-namespace` | string | yes* | PVC namespace |
| `--target-storage-class` | string | yes* | Target StorageClass name |
| `--target-pvc-name` | string | no | Override auto-generated target PVC name (default: `<name>-mig-<suffix>`) |
| `--target-access-mode` | string | no | Override access mode |
| `--target-capacity` | string | no | Override storage capacity |
| `--endpoint` | string | no | `route` or `nginx-ingress` (auto-detected) |
| `--subdomain` | string | no | Subdomain for nginx-ingress |
| `--ingress-class` | string | no | IngressClass for nginx-ingress |
| `--plan` | string | no | Path to plan YAML (batch mode) |
| `--skip-swap` | bool | no | Skip workload reference swap |
| `--verify` | bool | no | Enable checksum verification |
| `--image` | string | no | Container image for rsync pods |

*Not required when `--plan` is provided.

**Plan subcommand (`crane convert-storage plan`):**

| Flag | Type | Required | Description |
|------|------|----------|-------------|
| `--context` | string | yes | Cluster kubeconfig context |
| `--namespace` | string | yes | Namespace to discover PVCs in |
| `--output` | string | yes | Path to write plan YAML |
| `--label-selector` | string | no | Filter PVCs by label |

#### Plan File Format

```yaml
context: mycluster
namespace: myapp
suffix: a1b2
endpoint: route
pvcs:
  - name: mysql-data
    sourceStorageClass: gp2-csi
    targetStorageClass: gp3-csi
    targetName: mysql-data-mig-a1b2
    capacity: 10Gi
    accessModes: [ReadWriteOnce]
    action: convert
  - name: cache-vol
    sourceStorageClass: gp2-csi
    targetStorageClass: ""
    targetName: ""
    action: skip
```

#### Auto-Naming Convention

New PVC names follow MTC's naming pattern: `<original-name>-mig-<suffix>` where suffix is a random 4-character alphanumeric string, consistent across all PVCs in a plan.

For StatefulSet PVCs, the user must edit target names in the plan to follow the Kubernetes convention: `<newTemplateName>-<statefulSetName>-<ordinal>`. Example: `data-redis-0` -> `data-mig-a1b2-redis-0`.

#### Per-PVC Conversion Flow

```
 1. Pre-flight validation:
    - Source PVC exists and is Bound
    - Target StorageClass exists (Get by name; Forbidden = warn + proceed)
    - Target PVC name doesn't already exist (collision check)
 2. Create destination PVC with target StorageClass
 3. Detect UID requirements (OCP namespace annotation / workload spec fallback)
 4. Create rsync server pod (mounts destination PVC)
 5. Create stunnel TLS tunnel + endpoint (Route on OCP / Ingress on K8s)
 6. Copy server cert secret under client's expected name (intra-cluster cert sharing)
 7. Create rsync client pod (mounts source PVC)
 8. Monitor progress via rsync log parsing
 9. Wait for completion
10. Garbage collect rsync infrastructure (pods, stunnel, endpoint, secrets)
11. Label old PVC with migration metadata
```

After all PVCs are transferred:
```
12. Swap workload references (Deployments, DaemonSets, ReplicaSets, CronJobs, Jobs)
13. Auto-detect StatefulSet volumeClaimTemplates via regex and perform delete+recreate
14. Report summary
```

#### Workload Reference Swap

| Workload | Method |
|----------|--------|
| Deployment | Patch volume references |
| DaemonSet | Patch volume references |
| ReplicaSet | Patch (standalone only — skip owned-by-Deployment) |
| CronJob | Patch jobTemplate volume references |
| Job | Delete + recreate (immutable) |
| StatefulSet | Delete + recreate dance (volumeClaimTemplates immutable) |

**StatefulSet handling** (matching MTC's `swapStatefulSetPVCRefs()`):

1. Auto-detect: regex `^<templateName>-<stsName>-(\d+)$` matches PVC names to templates
2. Scale to 0 replicas
3. Create temporary StatefulSet (holds label selector to prevent PVC GC)
4. Rename template name + update volumeMounts in containers and initContainers
5. Delete original StatefulSet
6. Recreate with modified spec + original replica count
7. Delete temporary StatefulSet

#### Intra-Cluster Stunnel Architecture

Since both rsync server and client pods run in the same cluster/namespace, the pvc-transfer library's cross-cluster design requires adaptations:

* **Separate labels:** Server-side pods use `role: server` label, client-side pods use `role: client` label. This prevents the log reader from finding 2 pods when expecting 1.
* **Cert sharing:** The stunnel server generates TLS certs in a secret named after the destination PVC. The client expects certs in a secret named after the source PVC. The command copies the server's cert data into a second secret with the client's expected name, ensuring both use the same CA for mutual TLS authentication.
* **Garbage collection:** Both server-label and client-label resources are cleaned up separately after transfer.

#### Old PVC Handling

After successful conversion:
* Old PVC is **NOT deleted**
* Old PVC receives labels: `crane.konveyor.io/migrated-by: <run-id>` and `crane.konveyor.io/migrated-to: <new-pvc-name>`
* User can manually delete after verification

#### Non-Admin RBAC

The command works with namespace-admin permissions:

| Operation | Scope | Handling |
|-----------|-------|----------|
| List/Get PVCs | Namespace | Allowed for namespace-admin |
| Create PVCs | Namespace | Allowed for namespace-admin |
| Create/Delete Pods | Namespace | Allowed for namespace-admin |
| Create/Delete Secrets, ConfigMaps | Namespace | Allowed for namespace-admin |
| Create/Delete Routes/Ingresses | Namespace | Allowed for namespace-admin |
| List/Update Deployments, StatefulSets, etc. | Namespace | Allowed for namespace-admin |
| Get StorageClass by name | Cluster | Gracefully handles Forbidden — warns and proceeds |
| List StorageClasses | Cluster | Plan subcommand: best-effort — warns if Forbidden, user sets target SC manually |

### Security, Risks, and Mitigations

**Data in transit:** All data transfer goes through stunnel (TLS 1.3) with mutual certificate verification. Certificates are auto-generated per conversion run and cleaned up afterward.

**UID handling:** On OCP, the command reads the namespace's UID range annotation and runs rsync pods with the correct UID. On vanilla K8s, it reads the workload's security context. This ensures rsync can read/write files with the correct ownership.

**Old PVCs:** Preserved for rollback. If the conversion fails mid-way, the old PVC still has the original data. The user can manually revert the workload reference.

**StatefulSet delete+recreate:** During the brief window between delete and recreate, the StatefulSet doesn't exist. The temporary StatefulSet holds the label selector to prevent PVC garbage collection. Pods are already scaled to 0 before the dance.

**No cluster-admin required:** The command operates within namespace RBAC boundaries. Cluster-scoped operations (SC validation) are best-effort with graceful fallback.

## Design Details

### Internal Package Structure

| File | Purpose |
|------|---------|
| `cmd/convert-storage/convert_storage.go` | Root command: Complete/Validate/Run, single-PVC + batch dispatch, data transfer orchestration |
| `cmd/convert-storage/plan.go` | `plan` subcommand: PVC discovery, SC listing, auto-suggestion, YAML output |
| `cmd/convert-storage/swap.go` | Workload reference swap: Deployments, DaemonSets, ReplicaSets, CronJobs, Jobs, StatefulSets |
| `cmd/convert-storage/types.go` | ConversionPlan, PVCEntry structs, YAML I/O, suffix generation, auto-naming |
| `cmd/convert-storage/types_test.go` | Unit tests for types, naming, plan validation, plan round-trip, SC suggestion |
| `cmd/convert-storage/swap_test.go` | Unit tests for workload swap with controller-runtime fake client |

### Dependencies

Reuses existing libraries — no new dependencies:
* `github.com/backube/pvc-transfer` — rsync, stunnel, endpoint (Route/Ingress)
* `github.com/konveyor/crane/cmd/transfer-pvc` — exported `Verify`, `RestrictedContainers`, `Verbose`, `FollowClientLogs` types
* `sigs.k8s.io/controller-runtime/pkg/client` — K8s client
* `k8s.io/cli-runtime/pkg/genericclioptions` — kubeconfig handling

### Test Plan

**Unit tests:**

* Plan types: suffix generation, auto-naming, plan YAML round-trip, validation of required fields and action values
* SC suggestion: provisioner matching, GlusterFS/NFS -> Ceph mapping, default SC fallback
* Workload swap: patch-based swap for Deployments, DaemonSets, ReplicaSets, CronJobs; delete+recreate for Jobs and StatefulSets (including initContainer volumeMount rename)

**Integration / E2E tests:**

* Single PVC conversion on minikube and OCP: data integrity via checksum, workload swap, old PVC labeling
* Batch conversion via plan file: multiple PVCs with skip action, per-PVC checksum verification
* StatefulSet conversion: volumeClaimTemplate rename, delete+recreate dance, replicas restored, data intact
* Non-admin RBAC: all tests run as namespace-admin (not cluster-admin), verify graceful Forbidden handling
* OCP-specific: Route endpoint, UID detection via namespace annotation, file ownership preservation

### Upgrade / Downgrade Strategy

* **Additive command:** `crane convert-storage` is a new command. No existing behavior changes.
* **Exported symbols in transfer-pvc:** Four types were exported (`Verify`, `RestrictedContainers`, `Verbose`, `FollowClientLogs`) for reuse. Internal callers updated. No API break.
* **progress.go nil pointer fix:** Added nil guard on `TransferredData` in `Status()` — pre-existing bug #178.

## Implementation History

* [#655](https://github.com/migtools/crane/issues/655) — Parent feature issue
* [#656](https://github.com/migtools/crane/issues/656) — Convert a single PVC to a different StorageClass
* [#657](https://github.com/migtools/crane/issues/657) — Convert multiple PVCs via a plan file
* [#658](https://github.com/migtools/crane/issues/658) — Automatically update workload references after conversion
* [#659](https://github.com/migtools/crane/issues/659) — Handle StatefulSet volumeClaimTemplate conversion
* [#660](https://github.com/migtools/crane/issues/660) — Support non-admin users for StorageClass conversion
* [#661](https://github.com/migtools/crane/issues/661) — E2E test coverage for StorageClass conversion

## Drawbacks

* **Requires rsync infrastructure:** Even though source and destination are on the same cluster, the command still creates rsync server/client pods, stunnel TLS tunnel, and a Route/Ingress endpoint. This overhead is inherited from the pvc-transfer library which was designed for cross-cluster transfer. A future optimization could use a simpler copy mechanism for intra-cluster (e.g., a single pod mounting both PVCs).
* **Sequential PVC processing:** In batch mode, PVCs are processed one at a time. Parallel conversion could speed up large-scale operations but adds complexity.

## Alternatives

1. **Direct `kubectl` scripting:** Users could manually create PVCs, run rsync pods, and patch workloads. Rejected because it's error-prone, doesn't handle StatefulSets, and requires significant Kubernetes expertise.
2. **Transform plugin approach:** Add `--storage-class-map` flag to `crane transform` to rewrite `storageClassName` in manifests. Rejected as insufficient — transform only changes YAML files, it doesn't handle actual PVC data transfer or workload swapping at runtime.
3. **Use MTC operator:** Install the full MTC stack and use `StorageConversionPlan`. Rejected for crane users who want a lightweight CLI tool without operator dependencies.
4. **Single pod mounting both PVCs:** Instead of rsync over stunnel, create one pod that mounts both old and new PVCs and copies data with `cp -a`. Simpler but doesn't work with `ReadWriteOnce` PVCs that are already mounted by application pods. The rsync approach handles this by using the stunnel/endpoint networking layer.

## Infrastructure Needed

No new repositories required. Changes land in [migtools/crane](https://github.com/migtools/crane).
