---
title: indirect-data-migration
authors:
  - "@aufi / maufart"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-07-21
last-updated: 2026-07-21
status: provisional
see-also:
  - "/enhancements/crane-2.0/transfer-pvc-fixes"
replaces: []
superseded-by: []
---

# Indirect Data Migration

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. ~~Should the `--cloud-storage` flag accept non-S3 rclone remotes (GCS, Azure) in the first iteration, or limit to S3-compatible only?~~ **Decision:** Start with S3-compatible only, additional backends can be added in subsequent iterations.
2. ~~Should crane automatically clean up data from cloud storage after a successful transfer, or leave that to the user?~~ **Decision:** Clean up by default after successful transfer. A `--keep-cloud-data` flag allows users to skip the cleanup.
3. ~~Should `--bandwidth-limit` be included in the first release or deferred to a follow-up?~~ **Decision:** No additional rclone flags will be added for the 0.11 release unless directly requested or found critical during testing. Detailed flag evaluation is tracked in migtools/crane#689.

## Summary

This enhancement extends `crane transfer-pvc` with indirect data migration
via S3-compatible cloud storage. A new `--cloud-storage` flag changes the
transfer mechanism from direct rsync-over-stunnel to a two-phase flow:
the source cluster uploads PVC data to cloud storage using rclone, then the
destination cluster downloads from the same location. This enables PVC data
migration between clusters that have no direct network connectivity, such as
cross-cloud migrations, air-gapped environments, or scenarios where setting
up Ingress/Route endpoints between clusters is not feasible.


## Motivation

The current `crane transfer-pvc` command transfers PVC data between two
Kubernetes clusters using rsync over a stunnel TLS tunnel. This requires
direct network connectivity between the clusters — specifically, an
Ingress or Route endpoint on the destination cluster that the source cluster
can reach. This architecture has several limitations:

1. **No air-gapped support:** Clusters without direct connectivity cannot
   transfer PVC data at all. This is common in cross-cloud migrations (AWS
   to GCP), environments with strict network policies, or classified/regulated
   environments.

2. **Complex network setup:** The stunnel-based approach requires creating
   Ingress/Route resources, copying TLS certificates between clusters, and
   running 4 Pods (rsync client/server + stunnel client/server). Firewall
   rules and DNS configuration add further complexity.

3. **Single-threaded transfer:** rsync is inherently single-threaded. For
   PVCs with millions of small files, transfer can take hours where a
   multi-threaded tool would complete in minutes.

Cloud storage (S3, MinIO, GCS, etc.) is widely available in
environments running Kubernetes. Using it as an intermediary decouples
the source and destination clusters entirely — each cluster only needs
outbound HTTPS access to the cloud storage endpoint.

### Goals

- Add a `--cloud-storage` flag to `crane transfer-pvc` that enables indirect
  transfer via S3-compatible cloud storage
- Compile rclone as a Go library into the `migtools/rsync-transfer` image
  with selective backend imports (S3 + local) to minimize image size growth
- Support credential management via Kubernetes Secrets and local config files
- Provide optional client-side encryption (`--encrypt`)
- Maintain 100% backward compatibility — no changes to existing behavior
  when `--cloud-storage` is not used

### Non-Goals

- Adding rclone as a replacement for rsync in direct cluster-to-cluster
  transfers — rsync remains the default engine for direct transfers
- Supporting block storage (`volumeMode: Block`) — neither rsync nor rclone
  work with raw block devices; this requires a separate approach (dd-based
  or CSI snapshots)
- Building a dedicated backup/restore feature (`crane backup-pvc`) with
  retention policies, versioning, or deduplication
- Supporting all 70+ rclone backends — initial scope is S3-compatible storage
  only, with GCS and Azure as optional additions
- Bandwidth limiting (`--bandwidth-limit`) — deferred to a follow-up

## Proposal

### Overview

When `--cloud-storage` is provided, `crane transfer-pvc` replaces the
rsync/stunnel flow with two sequential rclone operations:

1. **Upload phase (source cluster):** A mover Pod mounts the source PVC
   read-only and uses rclone to sync its contents to the specified cloud
   storage location.

2. **Download phase (destination cluster):** A mover Pod mounts the
   destination PVC read-write and uses rclone to sync the contents from
   cloud storage into the PVC.

Note: `transfer-pvc` does not quiesce source workloads — ensuring
filesystem consistency (e.g. scaling down the application before
transfer) is the responsibility of the orchestration layer that invokes
`crane transfer-pvc`. This is consistent with the existing rsync-based
direct transfer, which also assumes the caller has quiesced the source.

Both mover Pods use the existing `quay.io/konveyor/rsync-transfer`
image, which will now include rclone compiled in as a Go library. No
stunnel is needed — rclone communicates with cloud storage over native
HTTPS.

```text
Source Cluster                    Cloud Storage              Destination Cluster
┌─────────────────┐              ┌──────────┐              ┌─────────────────┐
│                 │              │          │              │                 │
│  ┌───────────┐  │   rclone    │  S3      │   rclone    │  ┌───────────┐  │
│  │Source PVC │  │   sync up   │  bucket/ │   sync down │  │ Dest PVC  │  │
│  │ (ReadOnly)│  │────────────▶│  path/   │────────────▶│  │ (ReadWrite│  │
│  └─────┬─────┘  │              │          │              │  └─────┬─────┘  │
│        │mount   │              └──────────┘              │        │mount   │
│  ┌─────┴─────┐  │                                        │  ┌─────┴─────┐  │
│  │Mover Pod  │  │                                        │  │Mover Pod  │  │
│  │(rclone    │  │                                        │  │(rclone    │  │
│  │ upload)   │  │                                        │  │ download) │  │
│  └───────────┘  │                                        │  └───────────┘  │
└─────────────────┘                                        └─────────────────┘
```

### User Stories

#### Story 1: Cross-Cloud Migration

As a platform engineer migrating workloads from AWS EKS to GCP GKE, I want
to transfer PVC data through S3 so that I don't need to set up direct
network connectivity between my AWS and GCP clusters.

```bash
crane transfer-pvc \
  --source-context=aws-eks \
  --destination-context=gcp-gke \
  --pvc-name=postgres-data \
  --pvc-namespace=myapp \
  --cloud-storage=s3:migration-bucket/postgres-data \
  --rclone-config-secret=s3-credentials
```

#### Story 2: Air-Gapped Environment

As an operator in a classified environment, I want to migrate PVC data
between clusters that share access to an internal MinIO instance but have
no direct network path to each other.

```bash
crane transfer-pvc \
  --source-context=zone-a \
  --destination-context=zone-b \
  --pvc-name=app-data \
  --pvc-namespace=production \
  --cloud-storage=s3:internal-minio/migration/app-data \
  --rclone-config-secret=minio-credentials \
  --encrypt
```

### Implementation Details/Notes/Constraints

rclone will be compiled as a Go library into the existing `migtools/rsync-transfer`
container image. No new images or external binaries are introduced. Without the
`--cloud-storage` flag, `crane transfer-pvc` behaves exactly as before — 100%
backward compatible.

#### CLI Changes

New flags on `crane transfer-pvc`:

| Flag | Type | Required | Description |
|------|------|----------|-------------|
| `--cloud-storage` | string | No | S3-compatible target (e.g. `s3:bucket/path`). Activates indirect mode. **Warning:** uses `rclone sync` which overwrites existing data at the target path |
| `--rclone-config-secret` | string | Yes* | K8s Secret containing rclone.conf (must exist in both clusters) |
| `--rclone-config-file` | string | Yes* | Path to rclone.conf on disk (crane creates temporary Secrets) |
| `--encrypt` | bool | No | Enable client-side encryption via rclone crypt overlay |
| `--keep-cloud-data` | bool | No | Skip cloud storage cleanup after successful transfer |

\* One of `--rclone-config-secret` or `--rclone-config-file` is required
when `--cloud-storage` is set.

**Validation rules:**

- `--cloud-storage` without rclone config → error
- Both `--rclone-config-secret` and `--rclone-config-file` at once → error
- Referenced Secret does not exist in cluster → error before creating mover Pod

**Known limitation:** The same rclone config is used in both clusters.
Asymmetric credential setups (e.g. IAM roles on source, static keys on
destination) are not supported in the first iteration and may be added
later with per-cluster config flags.

#### rsync-transfer Image Changes

The `migtools/rsync-transfer` image gains rclone as a Go dependency. To
minimize image size growth, only the required backends are imported:

```go
import (
    _ "github.com/rclone/rclone/backend/s3"
    _ "github.com/rclone/rclone/backend/local"
    _ "github.com/rclone/rclone/backend/crypt"    // optional: --encrypt
)
```

This keeps the image size increase to ~20-30MB instead of ~50-60MB if all
70+ backends were included. The existing rsync and stunnel binaries remain
unchanged.

#### crane-lib Changes

A new rclone transfer engine is added alongside the existing rsync engine
in `crane-lib/state_transfer/transfer/`. The engine:

- Creates a mover Pod with PVC volume mount and rclone config Secret mount
- Invokes rclone sync via the Go library API (librclone) inside the Pod
- Reports progress programmatically via the library API
- Removes the Pod on completion (garbage collection)

#### Transfer Flow

1. Crane CLI validates flags
2. **Source phase:** Create mover Pod in source cluster
   - Image: `quay.io/konveyor/rsync-transfer:latest`
   - Mount source PVC as ReadOnly at `/mnt/pvc-data`
   - Mount rclone config Secret at `/etc/rclone/rclone.conf`
   - Sync via librclone API: `/mnt/pvc-data` → `<cloud-storage-path>`
   - Report progress via library API, garbage collect on completion
3. **Destination phase:** Create mover Pod in destination cluster
   - Mount destination PVC as ReadWrite at `/mnt/pvc-data`
   - Mount rclone config Secret
   - Sync via librclone API: `<cloud-storage-path>` → `/mnt/pvc-data`
   - Report progress via library API, garbage collect on completion

The transfer uses rclone's `sync` operation, which mirrors the source to
the target. Files in the target that are absent from the source will be
deleted — any existing data on the cloud storage path or the destination
PVC will be overwritten.

Filesystem metadata (mode bits, ownership, xattrs) is preserved using
rclone's `--metadata` flag. Symlinks are handled via `--links` (stored
as `.rclonelink` files during cloud transit and restored on download).

#### Credential Management

**Kubernetes Secret (production):** User creates a Secret containing
`rclone.conf` in both the source and destination cluster namespaces. The
mover Pod mounts the Secret at `/etc/rclone/rclone.conf`.

```bash
# Create rclone.conf
cat > rclone.conf <<EOF
[s3]
type = s3
provider = AWS
access_key_id = AKIAIOSFODNN7EXAMPLE
secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
region = us-east-1
EOF

# Create Secret in both clusters
kubectl create secret generic s3-credentials \
  --from-file=rclone.conf=rclone.conf \
  -n myapp --context=source-cluster

kubectl create secret generic s3-credentials \
  --from-file=rclone.conf=rclone.conf \
  -n myapp --context=dest-cluster
```

**Local config file (development):** Crane CLI reads the file from disk,
creates temporary Secrets in both clusters, and removes them after transfer
completion as part of garbage collection. Temporary Secrets are created
with crane-specific labels (e.g. `app.kubernetes.io/managed-by: crane`)
so they can be identified and cleaned up on subsequent runs if a previous
transfer crashed mid-flight.

#### Encryption (optional)

**Encryption (`--encrypt`):** Uses rclone's crypt overlay to encrypt data
before it leaves the cluster (client-side encryption). Technology: NaCl
SecretBox (XSalsa20 + Poly1305) with 256-bit keys. Crane dynamically
generates a crypt configuration wrapping the user's remote, so the mover
Pod transparently encrypts/decrypts during transfer. The encryption
password is sourced from the rclone config Secret.

Encryption is opt-in. Without this flag, data transfers unencrypted.

#### Comparison with Direct Transfer

| Aspect | Direct (rsync/stunnel) | Indirect (cloud storage) |
|--------|----------------------|------------------------|
| Network path | Source → stunnel → Ingress/Route → stunnel → Dest | Source → HTTPS → S3 → HTTPS → Dest |
| Network requirements | Ingress/Route between clusters | Both clusters need S3 access |
| TLS | Custom CA + mTLS (stunnel) | Native HTTPS (rclone) |
| Pod count | 4 (rsync + stunnel, client + server) | 2 (rclone upload + download, sequential) |
| Air-gapped clusters | Does not work | Works (S3 as intermediary) |
| Transfer parallelism | Single-threaded (rsync) | Multi-threaded (rclone, configurable) |
| Additional cost | None | Cloud storage + egress fees |

### Security, Risks, and Mitigations

**Security considerations:**

- Cloud storage credentials are stored in Kubernetes Secrets with standard
  RBAC controls. Documentation will include recommended Role definitions
  with least-privilege access (only `get` on the specific Secret).
- When using `--rclone-config-file`, crane creates temporary Secrets and
  removes them after transfer. If crane crashes mid-transfer, orphaned
  Secrets may remain — documented as a known limitation with manual
  cleanup instructions.
- The `--encrypt` flag ensures data is encrypted before leaving the cluster.
  Without it, data in cloud storage is protected only by the cloud
  provider's server-side encryption (if configured).

**Risks and mitigations:**

| Risk | Impact | Mitigation |
|------|--------|------------|
| Image size increase ~20-30MB | Longer pull on first deploy | Selective backend imports (S3 + local only) |
| librclone API is experimental | May break on rclone version upgrade | Pin specific version in go.mod, upgrade deliberately |
| Secret must exist in both clusters | User may forget one cluster | Pre-flight validation with clear error message |
| Partial upload on source failure | Inconsistent data in cloud storage | rclone is idempotent — re-run completes the transfer |
| Cloud storage and egress costs | Unexpected costs for users | Document cost estimates in user guide |
| Orphaned Secrets on crash | Credentials left in cluster | Garbage collection on next run; documented manual cleanup |

## Design Details

### Test Plan

**Unit tests:**

- Flag validation: `--cloud-storage` requires rclone config, mutual
  exclusion of `--rclone-config-secret` and `--rclone-config-file`
- rclone configuration generation: crypt overlay config for `--encrypt`
- Mover Pod spec construction: correct volume mounts, image reference,
  command arguments

**Integration tests:**

- Deploy MinIO (S3-compatible) in a kind cluster
- Create a PVC with test data (files of varying sizes and types)
- Run indirect transfer via MinIO
- Verify destination PVC contents match source
- Test with `--encrypt` flag
- Test symlink handling: relative symlinks, absolute symlinks, symlink loops
- Test error cases: missing Secret, unreachable cloud storage, partial
  transfer recovery

**E2E tests:**

- Two kind clusters with a shared MinIO instance
- Complete indirect transfer flow: source PVC → MinIO → destination PVC
- Verify data integrity (checksums)
- Verify filesystem metadata round-trip fidelity: create files with specific
  ownership (UID/GID), permissions, and symlinks on the source PVC, transfer
  through S3, and verify they are preserved on the destination. If metadata
  round-trip proves unreliable, fallback to `tar | rclone rcat` (archiving
  PVC contents into a single object that preserves POSIX metadata natively)
  will be evaluated. Document any limitations found.
- Verify backward compatibility: direct rsync/stunnel transfer still works
  with the updated image

### Upgrade / Downgrade Strategy

**Upgrade:** No action required. The `--cloud-storage` flag is additive.
Existing `crane transfer-pvc` invocations without the flag continue to
use the rsync/stunnel path. The updated `rsync-transfer` image is backward
compatible — it contains rsync and stunnel alongside rclone.

**Downgrade:** Removing the `--cloud-storage` flag from CLI invocations
reverts to direct transfer behavior. If the rsync-transfer image is
downgraded to a version without rclone, any `--cloud-storage` usage will
fail with a clear error (rclone library not available in image).

## Implementation History

- `2026-07-21`: Enhancement proposed as `provisional`

## Drawbacks

- **Additional image size:** The rsync-transfer image grows by ~20-30MB
  due to the rclone Go library. This increases initial pull time, though
  subsequent pulls benefit from layer caching.

- **Cloud storage costs:** Indirect transfer incurs cloud storage fees
  (storage + egress). For a one-time 100GB migration through S3 Standard,
  this is approximately $2.30 in storage (for one month) plus $9.00 in
  egress — modest but non-zero. Users must be aware of these costs.

- **Two-phase transfer is slower for co-located clusters:** When clusters
  have direct connectivity, the indirect path (upload + download) takes
  roughly twice the network time of a direct transfer. The `--cloud-storage`
  flag is opt-in precisely for this reason — direct transfer remains the
  default.

- **Dependency on external service:** The transfer depends on cloud storage
  availability. If S3 is down or unreachable, the transfer fails. rclone's
  built-in retry and resume mitigate transient failures, but sustained
  outages block the transfer entirely.

## Alternatives

### kopia

[kopia](https://kopia.io/) is a backup tool (2015) that provides
deduplicated, encrypted snapshots stored in a repository format. It was
considered as an alternative to rclone for the cloud storage transfer
engine.

| Criterion | rclone (chosen) | kopia |
|-----------|----------------|-------|
| Tool type | Sync/transfer | Backup |
| Output format | Direct file copy | Deduplicated repository (requires `kopia restore`) |
| Migration workflow | 1 step (sync) | 2 steps (snapshot + restore) |
| Parallelism | Multi-threaded (files + within-file) | File-level only |
| Cloud backends | 70+ | ~10 (as repository backends, not direct transfer) |
| Go library | Yes (librclone) | Yes, but requires repository management |
| Encryption | Optional (crypt overlay) | Mandatory (end-to-end) |
| Deduplication | No | Yes (content-addressable) |
| Ecosystem usage | VolSync (data mover), pv-migrate | Velero (backup engine) |

**Why rclone was chosen:** `crane transfer-pvc` is a migration tool that
needs direct file transfer between PVC and cloud storage. rclone produces
a direct mirror of the source files — what the source uploads, the
destination downloads, with no intermediate format. kopia's deduplicated
repository format would require the destination to understand and restore
from the kopia repository, adding unnecessary complexity for one-time
migration.

**When kopia would be appropriate:** If crane added a dedicated backup
feature (`crane backup-pvc`) with retention policies, versioning, and
deduplication, kopia would be the right tool. That use case is out of
scope for this enhancement.

### Separate rclone binary instead of Go library

Instead of compiling rclone as a Go library, include a pre-built rclone
binary in the container image and invoke it via `exec`.

**Rejected because:** This requires managing two processes in the container,
parsing stdout for progress reporting (brittle), and results in a larger
image (~60MB for the standalone binary vs ~20-30MB for selective Go imports).
The Go library approach provides direct programmatic control and is
consistent with how VolSync and pv-migrate integrate rclone.

### Separate container image for rclone

Create a new `crane-rclone-transfer` image alongside the existing
`rsync-transfer` image.

**Rejected because:** This doubles the image maintenance burden and
complicates the deployment model. The rsync-transfer image already serves
multiple purposes (rsync client, rsync server, stunnel) — adding rclone
as another capability keeps the single-image model.

## Infrastructure Needed

- **MinIO deployment for CI:** Integration and E2E tests require an
  S3-compatible object store. MinIO can run as a Pod in the test kind
  cluster — no external infrastructure needed.

- **No new repositories:** All changes go into existing repositories
  (`migtools/rsync-transfer`, `migtools/crane-lib`, `migtools/crane`).
