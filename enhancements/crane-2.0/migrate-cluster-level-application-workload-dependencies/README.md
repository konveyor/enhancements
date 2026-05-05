---
title: migrate-cluster-level-application-workload-dependencies
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
creation-date: 2026-05-05
last-updated: 2026-05-05
status: implementable
see-also:
  - "https://github.com/migtools/crane/issues/184"
  - "https://github.com/migtools/crane/issues/186"
  - "https://github.com/migtools/crane/issues/189"
---

# Migrate Cluster-Level Application Workload Dependencies

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. What is the ordering strategy when applying cluster-scoped resources before namespace-scoped ones? CRDs and SCCs must exist before workloads that reference them.
2. How should `crane apply` behave when cluster-scoped resources already exist on the target with conflicting definitions (merge, skip, fail)?
3. **Role Aggregation depth:** When a ServiceAccount is bound to an aggregated ClusterRole, should Crane export only the parent ClusterRole, or the full aggregation tree (the parent plus all child ClusterRoles carrying matching `aggregationRule` labels)? Exporting only the parent means the target cluster must already have the child roles installed for aggregation to produce the same effective permissions. Exporting the full tree is more complete but may pull in shared/system ClusterRoles that are not application-specific.
4. **CRD export vs. CRD-assumed-on-target:** The current implementation exports CRDs referenced by CR instances in the namespace. If the user lacks permissions to GET the CRD, it is not exported and the user is responsible for ensuring the CRD exists on the target. Should the enhancement explicitly state this two-tier behavior (export when possible, assume-on-target when forbidden), or should a CRD that cannot be exported be treated as a hard failure?
5. **Cluster-scoped apply opt-in vs. opt-out:** The current implementation exports cluster-scoped resources by default. For apply, should `crane apply` include cluster-scoped resources by default (opt-out via `--namespace-scoped-only`), or exclude them by default (opt-in via `--cluster-scoped-only` or `--include-cluster-scoped`)? The default matters because applying cluster-scoped resources affects the entire cluster and many admins will want to review them separately.

## Summary

Crane's export -> transform -> apply pipeline currently operates exclusively on namespace-scoped resources. This means applications that depend on cluster-level objects -- ClusterRoleBindings, custom SecurityContextConstraints (SCCs), CRD definitions, RoleAggregations (under question, see above) -- cannot be fully migrated. The missing dependencies cause workloads to fail on the target cluster even when the namespace-scoped resources are migrated correctly.

This enhancement extends the entire Crane pipeline to discover, export, transform, and apply cluster-scoped resources that are referenced by namespace workloads. It brings Crane toward feature parity with mig-controller's cluster-level migration capabilities while preserving Crane's Unix-philosophy design of small, composable, non-destructive tools.

The work spans three pipeline stages (export, transform, apply) and includes a design-input phase to establish parity with mig-controller's existing behavior as the baseline.

## Motivation

When migrating applications between Kubernetes or OpenShift clusters, namespace-scoped resources alone are often insufficient. Real-world applications depend on cluster-level objects:

* **RBAC:** ServiceAccounts bound via ClusterRoleBindings to cluster-wide roles
* **Security:** Custom SCCs that pods reference in their security context
* **Extensibility:** CRDs that define the custom resource types the application uses
* **Authorization:** RoleAggregations that compose fine-grained permissions

These cluster-scoped dependencies are typically missing on the target cluster. Without them, migrated workloads fail at admission, scheduling, or runtime. mig-controller already handles this scenario -- Crane currently does not, which is a significant gap for users migrating from mig-controller to the Crane CLI workflow.

Additionally, creating cluster-scoped resources typically requires elevated permissions (cluster-admin), which namespace administrators do not have. Crane must handle the permission boundary gracefully: exporting what it can, warning when permissions are insufficient, and supporting workflows where a cluster administrator handles the cluster-scoped portion separately.

### Goals

* **Full-pipeline support:** Cluster-scoped dependencies are discovered and carried through export -> transform -> apply without breaking the existing namespace-only workflow.
* **mig-controller parity baseline:** Document which cluster-level resource types mig-controller discovers and migrates, and use that as the starting reference for Crane's implementation.
* **Graceful permission handling:** When the user lacks cluster-admin privileges, `crane export` warns about inaccessible cluster-scoped resources and completes the namespace export successfully with partial data.
* **Non-destructive pipeline preserved:** Cluster-scoped resources are written to disk alongside namespace resources, maintaining Crane's auditable, GitOps-friendly output model.
* **Composability:** A namespace admin can run export/transform and hand off the cluster-scoped artifacts to a cluster admin for review and apply -- supporting split-responsibility workflows.

### Non-Goals

* **Automatic privilege escalation:** Crane will not attempt to acquire cluster-admin credentials on behalf of the user.
* **Conflict resolution for existing cluster-scoped resources:** If a CRD or SCC already exists on the target with a different definition, Crane will not merge or reconcile -- it will report the conflict. Automated merge strategies are out of scope.
* **Full cluster backup/restore:** This enhancement targets only cluster-scoped resources that are *referenced by* the namespace workloads being migrated, not arbitrary cluster-level backup.
* **Replacing Velero or mig-controller:** Crane remains a CLI toolkit; this does not replicate the operator-based orchestration of mig-controller.

## Proposal

### User Stories

#### Story 1: Namespace Admin Migrates an App with Custom SCCs

A developer runs `crane export --namespace myapp` on a source OpenShift cluster. Their application pods use a custom SCC (`myapp-scc`). Crane detects the SCC reference in the pod spec, attempts to export it, and -- since the developer has read access to SCCs -- includes `SecurityContextConstraints_clusterscoped_myapp-scc.yaml` in the export directory. The developer runs `crane transform` and `crane apply` on the target cluster (where they also have sufficient privileges), and the application starts successfully with its custom SCC in place.

#### Story 2: Namespace Admin Lacks Cluster-Level Read Permissions

A developer runs `crane export --namespace myapp`. The app references a ClusterRoleBinding, but the developer's RBAC does not grant GET on ClusterRoleBindings. Crane logs a warning: `WARNING: unable to export ClusterRoleBinding "myapp-binding": forbidden (requires cluster-level read access)`. The export completes with all namespace-scoped resources intact. The developer shares the warning output with their cluster admin, who can run a targeted export with elevated credentials.

#### Story 3: Cluster Admin Applies Cluster-Scoped Resources Separately

After the namespace admin generates the export directory, the cluster admin reviews the cluster-scoped manifests (CRDs, SCCs, ClusterRoleBindings). The cluster admin runs `crane apply --cluster-scoped-only` to generate the final cluster-level resources and then uses `kubectl apply` to deploy them. Finally, the namespace admin follows the same process for the remaining namespace resources. This split workflow strictly respects organizational permission boundaries.

#### Story 4: CI/CD Pipeline with Full Permissions

An automated pipeline runs with a service account that has both namespace and cluster-admin access. `crane export` captures everything. `crane transform` processes both scopes. `crane apply` applies cluster-scoped resources first (CRDs, SCCs), then namespace resources -- in the correct order -- with no manual intervention.

### Implementation Details/Notes/Constraints

The implementation is structured in three phases, preceded by a design-input deliverable:

#### Phase 0 -- mig-controller Parity Baseline (#184, closed)

Document which cluster-level resource types mig-controller discovers during namespace migration, how it resolves references (e.g., pod spec -> SCC, ServiceAccount -> ClusterRoleBinding), and when elevated permissions are required. This deliverable serves as the reference specification for Crane's behavior.

#### Phase 1 -- Export (#186, closed)

Extend `crane export` to:

* Traverse exported namespace resources and identify references to cluster-scoped objects (SCCs, ClusterRoleBindings, CRDs, RoleAggregations, etc.)
* Attempt to GET/list referenced cluster-scoped resources
* On success: write them to the export directory alongside namespace resources
* On forbidden: log a warning with the specific resource and required permission, and continue the export. The resource is recorded under `failures/clusterscoped/` so administrators can identify what was inaccessible.
* **CRD handling:** When a namespace contains CR instances backed by a CRD, Crane attempts to export the CRD definition. If the user lacks GET permissions on the CRD, Crane logs a warning and continues -- the user is responsible for ensuring the CRD is available on the target cluster. *(See Open Question 4.)*
* **Role Aggregation:** When a ServiceAccount is bound (via ClusterRoleBinding) to a ClusterRole, Crane exports that ClusterRole. For aggregated ClusterRoles, the current behavior exports only the parent ClusterRole -- not the child ClusterRoles whose labels match the `aggregationRule`. This means the target cluster must already have the contributing child roles for aggregation to produce the same effective permissions. *(See Open Question 3 for whether this should be expanded.)*
* Evaluate whether to replace the Velero discovery dependency with Crane-owned discovery logic for full control over cluster-scoped traversal

#### Phase 2 -- Transform & Apply (#189, depends on #188/#207)

Extend `crane transform` and `crane apply` to handle cluster-scoped resources:

* **Transform:** Plugins receive cluster-scoped manifests and can emit patches (e.g., renaming an SCC, adjusting a ClusterRoleBinding's subjects). The kustomize-based multi-stage pipeline (#207) must support cluster-scoped resources in its overlay structure.
* **Apply:** Cluster-scoped resources are applied before namespace resources (CRDs before CRs, SCCs before pods). Apply supports filtering by scope (`--cluster-scoped-only`, `--namespace-scoped-only`) to enable split-responsibility workflows. Existing namespace-only workflows remain unchanged when no cluster-scoped resources are present.

### Security, Risks, and Mitigations

**Elevated permissions exposure:** Exporting cluster-scoped resources requires read access that namespace admins typically lack. Crane mitigates this by:

* Never attempting privilege escalation
* Warning clearly when access is denied, without failing the export
* Supporting split workflows where cluster-admin actions are reviewed separately

**Accidental cluster-wide changes:** Applying cluster-scoped resources affects the entire cluster, not just a namespace. Mitigations:

* Cluster-scoped manifests are written to disk for review before apply
* `kubectl apply` with `--dry-run` shows what would be created/modified
* Scope-filtering flags prevent accidental application of cluster-wide resources

**Sensitive resources in export:** ClusterRoleBindings, SCCs, and CRDs may contain security-sensitive configurations. The exported YAML is written to the local filesystem and should be treated with the same care as kubeconfig files. Documentation will note this.

## Design Details

### Export Directory Layout

Cluster-scoped resources follow the existing convention but use `_cluster` as the namespace component:

```text
export-dir/
  resources/
    my-namespace/
      Deployment_my-namespace_frontend.yaml
      Service_my-namespace_frontend.yaml
    _cluster/
      SecurityContextConstraints_clusterscoped_myapp-scc.yaml
      ClusterRoleBinding_clusterscoped_myapp-binding.yaml
      CustomResourceDefinition_clusterscoped_widgets.example.com.yaml
  failures/
    clusterscoped/
```

### Transform Directory Layout (Kustomize)

Aligned with the multi-stage kustomize pipeline (#207):

```text
transform-dir/
  10_KubernetesPlugin/
    kustomization.yaml
    resources/
      ...
    patches/
      clusterscoped--security.openshift.io-v1--SecurityContextConstraints--myapp-scc.patch.yaml
  20_OpenShiftPlugin/
    ...
```

### Apply Ordering

1. CRDs (to register custom types)
2. Namespaces (if any are being created)
3. Cluster-scoped RBAC and security resources (ClusterRoles, ClusterRoleBindings, SCCs)
4. Namespace-scoped resources (existing behavior)

### Test Plan

* **Unit tests:** Reference-detection logic for each supported cluster-scoped type (SCC, ClusterRoleBinding, CRD, RoleAggregation). Permission-denied handling paths.
* **Integration tests:** Export with a kubeconfig that has mixed permissions (namespace-admin + partial cluster read). Verify warnings are emitted and namespace export succeeds.
* **E2E tests:** Full export -> transform -> apply cycle on a test cluster with:
    * An application using a custom SCC and a CRD-based custom resource
    * A partial-permission scenario where some cluster-scoped GETs are forbidden
    * Verification that the target application runs correctly after migration
* **Regression:** Existing namespace-only migration tests must continue to pass unchanged.

### Upgrade / Downgrade Strategy

* **Backward compatibility:** When no cluster-scoped resources are detected or the user lacks permissions, Crane behaves identically to the current version. No configuration changes are required.
* **Export format:** The `_cluster/` subdirectory is additive. Older versions of Crane that encounter it will ignore it (they only process namespace directories). Newer versions can process exports from older versions without issue.

## Subtasks

| Issue | Title | Status |
|-------|-------|--------|
| #184 | Explore and document mig-controller logic for migrating cluster-level resources | Closed |
| #186 | Update export action with cluster-level dependencies | Closed |
| #188 | Update transform and apply action with kustomize (prerequisite) | Closed |
| #189 | Update transform & apply for cluster-level resources | Open |

## Implementation History

* **Phase 0 (#184):** mig-controller cluster-level migration logic documented and reviewed. *(Completed)*
* **Phase 1 (#186):** `crane export` extended to gather cluster-scoped dependencies with permission-denied handling. *(Completed)*
* **Phase 2 (#189):** Transform & apply support -- *in progress*.

## Drawbacks

* **Increased complexity:** The pipeline now handles two resource scopes, adding branching logic to export, transform, and apply.
* **Permission model complexity:** Users must understand when cluster-admin is needed and how to use split workflows, adding cognitive overhead.
* **Velero discovery dependency decision:** Replacing Velero discovery with owned code is more work upfront but avoids coupling to Velero's roadmap.

## Alternatives

1. **Separate CLI tool for cluster-scoped resources:** A dedicated `crane-cluster-export` tool. This was considered but rejected -- it fragments the workflow and violates the principle that `crane export` should capture everything needed to migrate an application.
2. **Plugin-only approach:** Handle cluster-scoped discovery entirely via plugins rather than core logic. Rejected because reference detection (pod -> SCC, SA -> ClusterRoleBinding) requires access to the namespace resources during export, which plugins don't currently have.

## Infrastructure Needed

No new repositories required. Changes land in [migtools/crane](https://github.com/migtools/crane) and [konveyor/crane-lib](https://github.com/konveyor/crane-lib).
