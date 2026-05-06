---
title: validate-api-compatibility
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
  - "https://github.com/migtools/crane/issues/185"
  - "https://github.com/migtools/crane/issues/319"
replaces: []
superseded-by: []
---

# Validate API Compatibility

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. Should crane validate also check that CRDs backing custom resources are installed on the target, or only check built-in API discovery rows?
2. For the next phase (post Konveyor 0.10 release), should the command support a `--fix` or `--rewrite` mode that rewrites manifests to a compatible apiVersion when one exists, or should that remain a transform plugin concern?
3. Should the `validate` command be limited to a local schema check, or should it include a server-side "can-i" check to verify user permissions and cluster policies?
    * **Option 1: Local Schema Validation** -- Validates manifests against Kubernetes resource definitions without connecting to a cluster.
    * **Option 2: Server-Side Validation (The "can-i" approach)** -- Uses `kubectl apply --dry-run=server` to simulate the actual import process against the live API. Requires an active network connection and valid credentials at the time of validation.
    * **Option 3: Hybrid Approach** -- Implement Local Validation as the default while providing an optional `--server-side` flag.
4. Do we want to have both console output as well as a JSON report (detailed under the Design Details section below)?
5. In the case where the target is GitOps, is the crane validate workflow valid here? If yes, whose responsibility is it?

## Summary

Crane's pipeline currently has no target-aware preflight step. Resources are exported from the source cluster, transformed via plugins, and rendered into final manifests by `crane apply` -- but if the target cluster does not serve the same group/version for a resource, applying those manifests fails with an opaque API error. Users discover incompatibilities only at `kubectl apply` time, after they have already invested effort in the export, transform, and apply phases.

This enhancement adds a new **`crane validate`** command -- a read-only preflight check that scans the final rendered manifests from `crane apply`'s output directory, queries the **target** cluster's discovery API, and reports which apiVersion+kind combinations are not available on the target under strict group/version matching. When an incompatible resource's kind exists under a different apiVersion on the target, validate suggests the alternative. This brings Crane toward feature parity with mig-controller's GVK compatibility warnings, giving operators early visibility into API skew between clusters.

The enhancement is delivered in phases:

* **Phase 1 (implemented):** Live-cluster validation -- `crane validate` queries the target cluster's discovery API directly via kubeconfig.
* **Phase 2 (planned):** Offline/disconnected validation -- `crane validate --api-resources api-resources.json` accepts the JSON output of `kubectl api-resources -o json` for air-gapped environments, pre-provisioning checks, and CI/CD pipelines without cluster access.
* **Phase 3:** GitOps target (to be considered).

## Motivation

Migration between Kubernetes or OpenShift versions frequently involves API changes:

* **Removed APIs:** `extensions/v1beta1` Deployments removed in Kubernetes 1.16+, `policy/v1beta1` PodSecurityPolicies removed in 1.25+.
* **Moved groups:** Resources migrating from `authorization.openshift.io` to `rbac.authorization.k8s.io`, or from `extensions` to `apps`/`networking.k8s.io`.
* **CRD-backed types:** Custom resources whose CRDs may not be installed on the target cluster.

Today, Crane exports using the source cluster's preferred versions (via `discoverPreferredResources()` in `cmd/export/discover.go`). The export-group-mapping enhancement improved version selection at export time, but this does not cover all cases -- particularly when source and target clusters are different Kubernetes/OpenShift major versions.

mig-controller provides GVK compatibility warnings by comparing source and destination discovery. Crane lacks an equivalent. Without it, users must either manually cross-reference API versions or discover failures during apply, which is too late for CI pipelines and disruptive for manual workflows.

Additionally, many real-world migrations involve **disconnected or air-gapped environments** where the migration workstation has no direct network access to the target cluster. Live-only validation cannot serve these scenarios. Offline validation using a captured API surface snapshot closes this gap.

### Goals

* **Early detection:** Surface all API incompatibilities between final rendered manifests and the target cluster *before* `kubectl apply`, in a single report.
* **CI-friendly:** Provide deterministic exit codes so `crane validate` can gate a migration pipeline -- non-zero exit when incompatible GVKs are found.
* **Read-only:** Validate never modifies manifests or cluster state. It is a pure reporting command.
* **Low privilege:** Validate only requires basic cluster authentication -- it uses the Kubernetes discovery API (`/api`, `/apis`) which is accessible to all authenticated users via the default `system:discovery` ClusterRole. No special RBAC is needed.
* **Pipeline composability:** Validate fits naturally into the export -> transform -> apply -> validate flow, reading from `crane apply`'s output directory without requiring changes to other commands.
* **Suggestion engine:** When an incompatible resource's kind is available under a different apiVersion on the target, suggest the alternative to guide the user.
* **Offline/disconnected support (Phase 2):** Support validation against air-gapped clusters by accepting `kubectl api-resources -o json` output as input, using the same matching logic as live mode.

### Non-Goals

* **Automatic manifest rewriting:** Validate reports problems; it does not fix them. Rewriting apiVersions is the domain of transform plugins.
* **Schema validation:** This is not an OpenAPI schema validator. It checks whether the API *exists* on the target, not whether the manifest fields are valid for that API version.
* **Source cluster access:** Validate operates on filesystem YAML only; it does not require a connection to the source cluster.
* **Version negotiation or upgrade path recommendation:** Validate suggests alternative apiVersions when available (e.g., "available as apps/v1"), but does not provide upgrade path guidance or migration recipes.
* **GitOps without a known target cluster:** When crane outputs manifests into a GitOps repo where the target cluster is not yet decided, GVK validation is not applicable -- crane's job stops at producing valid Kubernetes YAML.

## Proposal

### User Stories

#### Story 1: Operator Validates Before Applying to Production Target (Live Mode)

An operator has exported resources from an OpenShift cluster, transformed them, and rendered final manifests with `crane apply`. Before running `kubectl apply` on the target, they run `crane validate`. The output reports incompatible resources. The operator knows to go back and run transform plugins that handle the version mapping, then re-run apply and validate.

#### Story 2: CI Pipeline Gates on Validation

A CI/CD pipeline runs `crane validate` after `crane apply`. The non-zero exit code from `crane validate` halts the pipeline before any cluster modification occurs.

#### Story 3: Post-Transform Validation Confirms Fixes

After running `crane transform` with plugins that remap deprecated APIs, the operator runs the full pipeline and validates the final output. All entries report OK; exit code 0. The operator proceeds to `kubectl apply` with confidence.

#### Story 4: CRD-Backed Custom Resources

An application uses a CRD-backed type `widgets.example.com/v1`. The CRD is not installed on the target cluster. `crane validate` reports the incompatibility. The operator installs the CRD on the target before applying.

#### Story 5: Air-Gapped / Disconnected Environment (Phase 2 -- Offline Mode)

The target cluster is on a restricted network. A team member with cluster access captures the API surface with `kubectl api-resources -o json`. The migration team validates without cluster access using `crane validate --api-resources api-resources.json`. The matching logic, report format, and exit codes are identical to live mode.

#### Story 6: Pre-Provisioning Check (Phase 2 -- Offline Mode)

The target cluster doesn't exist yet, but the team knows it will be OCP 4.14. Someone captures the API surface from a reference cluster and uses it for validation.

#### Story 7: CI/CD Pipeline Without Cluster Access (Phase 2 -- Offline Mode)

For multi-cluster targeting, the pipeline runs validate once per cluster using the corresponding `api-resources.json` file.

### Implementation Details/Notes/Constraints

#### Command Structure

New command at `cmd/validate/validate.go`, following the existing Cobra pattern (self-contained package, options struct with Complete/Validate/Run, PreRun with viper binding).

**Flags (Phase 1 -- implemented):**

| Flag | Description |
|------|-------------|
| `--input-dir` / `-i` | Root directory of final rendered manifests to scan (default: `"output"`, matching crane apply's `--output-dir` default) |
| `--validate-dir` | Directory where validation report and failure artifacts are saved (default: `"validate"`) |
| `--kubeconfig` | Path to target cluster kubeconfig |
| `--context` | Kubeconfig context for target cluster |
| `--output` / `-o` | Report file format: `json` (default) or `yaml`. Controls the format of the persisted report file. Table output is always printed to the terminal. |

**Additional flag (Phase 2 -- planned):**

| Flag | Description |
|------|-------------|
| `--api-resources` | Path to `kubectl api-resources -o json` output file. Enables offline validation. Mutually exclusive with `--kubeconfig`/`--context`. |

The command also inherits all standard kubeconfig flags from `genericclioptions.ConfigFlags` (`--cluster`, `--server`, `--token`, etc.) and global crane flags (`--debug`, `--flags-file`).

The command enforces `cobra.NoArgs` -- positional arguments are rejected to prevent typos like `crane validate ./manifests` from silently validating the default directory.

**Mutual exclusion (Phase 2):**

When `--api-resources` is NOT set, existing behavior is unchanged -- `crane validate` uses the default kubeconfig as it does today. No breaking changes.

#### Pipeline Position

Validate operates on the **final rendered manifests** from `crane apply`'s output directory, not the raw export or intermediate transform output. This is deliberate:

* **Export** produces raw resources from the source cluster
* **Transform** runs plugins that may modify apiVersions, add/remove fields
* **Apply** renders final YAML via `kubectl kustomize` into the output directory
* **Validate** checks those final manifests against the target cluster's API surface (live or offline)
* **`kubectl apply`** deploys to the target cluster

Validating after apply (rendering) ensures that transform plugins' changes are reflected in the check. Validating raw exports would produce false positives for resources that transforms will fix.

#### Scanner

Walk the input directory tree recursively using `filepath.WalkDir`. For each `.yaml`/`.yml`/`.json` file:

* Split multi-document YAML using `k8s.io/apimachinery/pkg/util/yaml.NewDocumentDecoder` (proper YAML-spec-aware document splitting)
* Decode each document individually -- one bad document logs a warning and continues to the next, without aborting remaining documents in the file
* Extract `apiVersion`, `kind`, `metadata.namespace` from each document
* Skip documents missing `apiVersion` or `kind` (e.g., empty documents, comments-only documents)
* Skip the `failures/` subdirectory to avoid re-scanning previous validation output
* Collect distinct (group, version, kind, namespace) tuples, tracking source files for each
* Deduplicate by GVK+namespace and sort results by group/version/kind/namespace

Input filenames are sanitized when writing failure artifacts to prevent path traversal from malicious manifest content.

#### Target Discovery

**Live mode (Phase 1):** Use client-go discovery client:

* Call `ServerGroupsAndResources()` on the target cluster
* Handle partial discovery failures gracefully (continue with available groups, log warning)
* Call `discoveryClient.Invalidate()` before querying to ensure fresh results (no stale cache)
* Build a two-level lookup map: groupVersion -> kind -> APIResource

**Offline mode (Phase 2):** Parse `kubectl api-resources -o json` output:

* Deserialize the native Kubernetes `APIResourceList` JSON with `json.Unmarshal`
* Build the same two-level lookup map as live mode
* Zero custom parsing -- the JSON output uses the same Go types (`metav1.APIResource`) the discovery client returns internally

The JSON output from `kubectl api-resources -o json` provides everything needed:

| JSON field | Maps to | Used for |
|------------|---------|----------|
| `group` + `version` | `groupVersion` (e.g., `apps/v1`; core resources omit `group`, so groupVersion = `v1`) | GVK matching |
| `kind` | `kind` | GVK matching |
| `name` | `resourcePlural` | Report output |
| `namespaced` | `namespaced` | Future: scope validation |

**Permissions (live mode):** Only requires the Kubernetes discovery API (`/api`, `/apis`), which is accessible to all authenticated users via the default `system:discovery` ClusterRoleBinding -> `system:authenticated` group. No namespace-level RBAC or resource-read permissions are needed.

**Permissions (offline mode):** No cluster access needed from the migration workstation. The person capturing the API resources file needs only basic authentication on the target cluster.

#### Matching Logic

For each (group, version, kind) from the scanned manifests:

1. Look up the groupVersion in the discovery index
2. If groupVersion not found: mark Incompatible with reason "API version X not available on target cluster"
3. If groupVersion found but kind not present: mark Incompatible with reason "kind X not found in API version Y on target cluster"
4. If both match: mark OK and populate the resource plural name
5. For incompatible resources: check if the same kind is available under a different apiVersion on the target (suggestion engine). If so, populate the Suggestion field (e.g., "available as apps/v1") and append it to the Reason for visibility in both table and JSON output.

The matching logic is identical for live and offline modes -- only the source of the discovery index differs.

#### Output

**Terminal (always):** Human-readable table with columns: APIVERSION, KIND, NAMESPACE, RESOURCE, STATUS, REASON, SUGGESTION. Followed by a summary line and a clear result line:

* `Result: PASSED -- all resources compatible with target cluster`
* `Result: FAILED -- N resource(s) incompatible with target cluster`

**Report file (persisted to `--validate-dir`):** Written as `report.json` (default) or `report.yaml` (with `-o yaml`). Contains the full ValidationReport struct including:

* `mode`: `"live"` or `"offline"` -- indicates how the target API surface was sourced
* `clusterContext`: kubeconfig context name (live mode)
* `apiResourcesSource`: file path (offline mode)
* `results`: array of ValidationResult entries
* `totalScanned`, `compatible`, `incompatible`: summary counts

**Failure artifacts:** When incompatibilities are found, individual YAML files are written to `--validate-dir/failures/`, one per incompatible GVK+namespace. Files are named `Kind_group_version_namespace.yaml` (matching export's naming pattern). All filename components are sanitized to prevent path traversal.

#### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | All GVKs compatible |
| 1 | One or more incompatible GVKs found |
| 2 | Command/runtime error (I/O failure, discovery failure, invalid input, etc.) |

### Security, Risks, and Mitigations

**Target cluster access (live mode):** Validate requires read access to the target cluster's discovery API. This is a low-privilege operation -- discovery is available to all authenticated users via the default `system:discovery` ClusterRole. No special RBAC setup is needed. The kubeconfig must be handled securely.

**No write operations:** Validate is strictly read-only on the cluster. It only writes to the local filesystem (report and failure artifacts under `--validate-dir`).

**Path traversal prevention:** Failure artifact filenames are derived from manifest content (kind, apiVersion, namespace). All components are sanitized through `safeFilePart()` which strips path separators and non-alphanumeric characters, preventing directory escape.

**False positives with CRDs:** If a CRD is installed on the target but its API group hasn't been aggregated yet (e.g., during CRD installation), validate may report a false incompatibility. Documentation should note this timing consideration.

**Stale discovery cache (live mode):** The command calls `discoveryClient.Invalidate()` before querying to ensure fresh results from the target cluster's API server.

**Stale data (offline mode):** The `kubectl api-resources -o json` file is a point-in-time snapshot. Users should re-capture after cluster upgrades or CRD changes. The report includes the source file path and mode for traceability.

## Design Details

### Architecture

#### Phase 1: Live Mode (implemented)

The command reads manifests from `crane apply`'s output directory, connects to the target cluster via kubeconfig, queries the discovery API, and produces a compatibility report.

#### Phase 2: Live + Offline Mode (planned)

Key refactoring: decouple index building from matching. Currently `MatchResults` takes a `DiscoveryClient` and builds the index internally. Phase 2 requires:

1. Extract the discovery index as a public type (`DiscoveryIndex`)
2. Add `ParseAPIResourcesJSON(path)` that deserializes the kubectl JSON output and builds the same index type
3. Create `MatchResultsFromIndex(entries, index)` that takes a pre-built index
4. Keep `MatchResults` as a convenience wrapper (calls `buildDiscoveryIndex` then `MatchResultsFromIndex`)

Matching logic (`matchEntry`, `addSuggestion`, `buildKindIndex`) stays untouched.

### Internal Package Structure

| File | Purpose |
|------|---------|
| `cmd/validate/validate.go` | Cobra command: flags, Complete/Validate/Run, PreRun with viper |
| `cmd/validate/validate_test.go` | Flag registration, validation, and NoArgs tests |
| `internal/validate/types.go` | Data types: ManifestEntry, ValidationResult, ValidationReport, ErrValidationFailed |
| `internal/validate/scanner.go` | Recursive dir walker + NewDocumentDecoder-based multi-doc YAML parser + dedup |
| `internal/validate/scanner_test.go` | 10 tests: single-doc, multi-doc, nested dirs, dedup, core/named group parsing, cluster-scoped, skip failures dir, skip non-YAML |
| `internal/validate/matcher.go` | Discovery index builder, GVK matching, MatchResultsFromIndex, suggestion engine |
| `internal/validate/matcher_test.go` | 8 tests: all OK, missing GV, missing kind, mixed, suggestions, no suggestions, empty, resource plural |
| `internal/validate/report.go` | Table/JSON/YAML formatters, WriteFailures with filename sanitization |
| `internal/validate/report_test.go` | 4 tests: table formatting, JSON round-trip, empty report |
| `internal/validate/api_resources.go` | *(Phase 2)* ParseAPIResourcesJSON -- deserialize kubectl JSON into DiscoveryIndex |
| `internal/validate/api_resources_test.go` | *(Phase 2)* JSON parsing tests: valid, empty, malformed, core vs non-core groups |

### Test Plan

**Phase 1 (implemented):**

* **Unit tests (23 tests):**
    * YAML scanner: single-doc, multi-doc, nested directories, malformed YAML resilience, empty files, non-YAML files, failures dir skipping, deduplication, core group parsing, named group parsing, cluster-scoped resources
    * Matcher (with mock discovery client): exact match, missing group/version, missing kind, mixed results, suggestion when alternative exists, no suggestion when kind not on target, empty entries, resource plural populated
    * Reporter: table output formatting and column headers, JSON encoding and round-trip, empty report
    * Command: flag registration and defaults, flag validation (missing dir, not-a-directory, invalid output format, valid formats), positional argument rejection (`cobra.NoArgs`)
* **Integration tests:**
    * Mock discovery client returning controlled API group lists
    * Validate against a directory with known compatible and incompatible resources
    * Verify exit codes for all-compatible, some-incompatible, and error scenarios
* **E2E tests:**
    * Full pipeline: export from source -> transform -> apply -> validate against target with known API differences -> confirm report accuracy
    * Confirm that resources reported as Incompatible actually fail on `kubectl apply`, and those reported OK succeed
* **Regression:** Existing export/transform/apply tests unaffected (validate is additive)

**Phase 2 (planned):**

* **Unit tests:** JSON parsing -- valid output, empty resources, malformed JSON, core resources (no group), non-core resources (with group), duplicate kinds across groups
* **Unit tests:** Offline matching -- same scenarios as live matcher tests but with a parsed index from JSON instead of a discovery client
* **Unit tests:** Mutual exclusion -- `--api-resources` with `--context` or `--kubeconfig` returns error
* **Integration tests:** Capture `kubectl api-resources -o json` from a test cluster, run `crane validate --api-resources` with it -- ensure results match live validate against the same cluster
* **E2E tests:** Cross-platform scenarios (EKS api-resources validated against OCP manifests)
* **Regression:** Live-mode validate behavior unchanged

### Offline Mode: Edge Cases (Phase 2)

| Concern | Handling |
|---------|----------|
| **Stale data** | Document that the file is a point-in-time snapshot. Users should re-capture after cluster upgrades or CRD changes. |
| **CRDs installed after capture** | Same as above -- re-run the kubectl command. |
| **Empty resources array** | Error: "api-resources file contains no resources" |
| **Malformed JSON** | Error from `json.Unmarshal` with clear message. |
| **Missing `kind` field at top level** | Warn if `kind` is not `APIResourceList`, but still attempt to parse. |
| **Core resources (no `group` field)** | `group` is omitted for core API resources. Parser uses `version` alone as `groupVersion` (e.g., `"v1"`). |
| **Duplicate resources across groups** | Multiple entries with the same kind but different group/version are valid (e.g., `Event` in `v1` and `events.k8s.io/v1`). Index handles this naturally since it's keyed by `groupVersion`. |
| **Verbs / RBAC** | Out of scope. `verbs` field is present in JSON but ignored. |
| **Multi-cluster** | `--api-resources` takes one file. For N clusters, run validate N times with the corresponding file. |
| **kubectl version compatibility** | `kubectl api-resources -o json` is available since K8s 1.11+. The `APIResourceList` schema is stable. |
| **GitOps without target cluster** | When crane outputs to a GitOps repo and no target cluster is known yet, offline validate is not applicable. GVK validation requires a target cluster's API surface. |

### Implementation Plan (Phase 2)

Single phase -- everything ships together.

| Task | Files | Effort |
|------|-------|--------|
| `ParseAPIResourcesJSON` -- deserialize kubectl JSON into discovery index | `internal/validate/api_resources.go` | S |
| Refactor `MatchResults` to extract `MatchResultsFromIndex` | `internal/validate/matcher.go` | S |
| Add `--api-resources` flag + mutual exclusion logic | `cmd/validate/validate.go` | S |
| Add mode info to `ValidationReport` | `internal/validate/types.go`, `report.go` | S |
| Unit tests for JSON parsing (valid, empty, malformed, core vs non-core groups) | `internal/validate/api_resources_test.go` | M |
| Unit tests for offline matching | `internal/validate/matcher_test.go` | S |
| Update validate command tests | `cmd/validate/validate_test.go` | S |

**Estimated effort:** ~2 days

### Upgrade / Downgrade Strategy

* **Additive command:** `crane validate` is a new command. No existing behavior changes. Users who don't invoke it see no difference.
* **No data format changes:** Validate reads the existing apply output directory format. It does not modify it.
* **Offline mode (Phase 2):** Adding `--api-resources` is backwards-compatible. When the flag is not set, behavior is identical to Phase 1.

## Implementation History

* **#185:** Exploration of mig-controller's resource versioning logic -- design input for this enhancement. *(Open)*
* **#230:** Feature story for this enhancement. *(Open)*
* **#302:** Phase 1 implementation PR -- crane validate command with live-cluster validation, 23 unit tests. *(Merged)*
* **#319:** Phase 2 feature request -- offline/disconnected cluster support. *(Open)*

## Drawbacks

* **Additional step in the pipeline:** Users must remember to run `crane validate` -- it is not automatically invoked by `crane apply`. This is deliberate (composability), but means users can still skip it.
* **Strict matching may over-report:** Some resources may work on the target under a different version of the same group (e.g., v1beta1 -> v1), but strict matching flags them as incompatible. The suggestion engine mitigates this by showing available alternatives.
* **Discovery limitations (live mode):** The Kubernetes discovery API may not surface all available API versions in all configurations (e.g., aggregated API servers with intermittent availability). The command handles partial discovery failures gracefully.
* **Stale data (offline mode):** The api-resources JSON file is a point-in-time snapshot that may drift from the actual cluster state. Documentation should emphasize re-capturing after changes.

## Alternatives

1. **Integrate validation into crane apply as a preflight step:** Rejected because it couples a read-only check to a write command, violating Crane's Unix-philosophy composability. Users may want to validate without being ready to apply, or validate in a different environment than where apply runs.
2. **Validate at export time against both source and target:** Rejected because export is source-only by design, and requiring target access during export adds a dependency that doesn't exist today. Validate as a separate command keeps concerns cleanly separated.
3. **Transform plugin that rewrites apiVersions:** Complementary, not a replacement. A transform plugin can fix known version mappings, but the user still needs a way to discover *which* mappings are needed. Validate provides that discovery.
4. **Static compatibility database:** Ship a hardcoded map of "version X removed in Kubernetes Y." Rejected because it cannot cover CRDs, custom API servers, or OpenShift-specific APIs, and requires constant maintenance.
5. **Custom offline format (Phase 2 alternative):** Define a crane-specific YAML format for target API surface. Rejected in favor of `kubectl api-resources -o json` because it requires zero custom parsing, uses native Kubernetes types, and needs no crane tooling on the target cluster side.
6. **Using `kubectl apply --dry-run=server`** (or client for offline, without schema validation in this client case) to check if manifests are valid for target cluster.

## Infrastructure Needed

No new repositories required. Changes land in [migtools/crane](https://github.com/migtools/crane).
