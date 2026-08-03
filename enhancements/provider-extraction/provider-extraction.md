---
title: provider-extraction
authors:
  - "@jmle"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-07-31
last-updated: 2026-07-31
status: provisional
see-also:
  - "/enhancements/enhancements/generic-to-specific-providers/generic-provider-to-specific-providers.md"
  - "https://github.com/konveyor/analyzer-lsp/issues/1013"
replaces: []
superseded-by: []
---

# Extract Providers from analyzer-lsp into Separate Repositories

Move each external provider out of the analyzer-lsp monorepo into its own standalone repository. Providers already communicate with the engine via gRPC and each has its own `go.mod`; this enhancement completes that separation so that each provider can be developed, versioned, released, and CI-tested independently.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

The analyzer-lsp monorepo currently contains all external providers under `external-providers/`. Each provider is already a separate Go module with its own `go.mod`, communicates with the engine via gRPC, and builds as an independent binary. However, they share the repository, the Dockerfile build context (`COPY / /analyzer-lsp`), CI workflows, and the Makefile. Changes to any provider require a PR to analyzer-lsp, and CI rebuilds everything on every PR.

This enhancement proposes moving each provider into its own GitHub repository under `konveyor/`. Providers continue to depend on `github.com/konveyor/analyzer-lsp` as a Go module for shared interfaces and types -- no separate SDK repository is needed because the dependency is one-directional (providers import from analyzer-lsp, never the reverse). Container image names remain unchanged (`quay.io/konveyor/<provider-name>`), so downstream consumers like kantra, the operator, and koncur require minimal updates.

## Motivation

### Goals

- **Independent versioning and releases:** Each provider can cut releases on its own schedule. A bug fix in the Java provider does not require a new analyzer-lsp release.
- **Faster CI:** PRs to a provider only build and test that provider, not the entire monorepo. analyzer-lsp CI focuses on the engine and built-in provider.
- **Clearer ownership:** Each repository has its own OWNERS, issue tracker, and PR queue. Contributors know exactly where to go for a given provider.
- **Smaller, focused container images:** Each provider's Dockerfile only includes what it needs, without pulling in the full monorepo as build context.
- **Alignment with existing providers:** `c-sharp-analyzer-provider` and `java-analyzer-provider` (Rust) are already separate repositories. This makes the Go-based providers consistent with that pattern.

### Non-Goals

- **Moving rules into provider repositories.** Extracting providers opens the door to co-locating each provider's rules alongside its code, but that is a separate effort with its own implications for rule discovery and packaging. This enhancement only moves provider source code and Dockerfiles.

## Proposal

### New Repositories

| New repository | Source | Container image |
|---|---|---|
| `konveyor/java-external-provider` | `external-providers/java-external-provider/` | `quay.io/konveyor/java-external-provider` |
| `konveyor/go-external-provider` | `external-providers/go-external-provider/` | `quay.io/konveyor/go-external-provider` |
| `konveyor/python-external-provider` | `external-providers/python-external-provider/` | `quay.io/konveyor/python-external-provider` |
| `konveyor/nodejs-external-provider` | `external-providers/nodejs-external-provider/` | `quay.io/konveyor/nodejs-external-provider` |
| `konveyor/yq-external-provider` | `external-providers/yq-external-provider/` | `quay.io/konveyor/yq-external-provider` |

### Dependency Model

Each extracted provider continues to `require github.com/konveyor/analyzer-lsp` in its `go.mod` and import these packages:

| Package | Used for |
|---|---|
| `provider/` | Core interfaces (`BaseClient`, `ServiceClient`), types (`Config`, `InitConfig`, `Capability`, `ProviderEvaluateResponse`), `NewServer()` gRPC wrapper, `FileSearcher`, `MultilineGrep` |
| `engine/` | `CodeSnip` interface, `Location`, `Position` types |
| `engine/labels/` | Label handling utilities |
| `lsp/protocol/` | LSP protocol type definitions |
| `lsp/base_service_client/` | Base LSP service client |
| `jsonrpc2_v2/` | JSON-RPC 2.0 implementation |
| `output/v1/konveyor/` | `Dep`, `DepDAGItem` types |
| `tracing/` | OpenTelemetry tracing |

This is the existing pattern. The committed `go.mod` files already have versioned dependencies on analyzer-lsp (e.g., `v0.9.0-alpha.1.0.20251211173054-36ef5519576d`). The `replace` directives are only injected by the Makefile and Dockerfiles for local development and will no longer be needed after extraction.

### Per-Provider Repository Structure

Each provider repo follows a standard layout:

```text
konveyor/<provider-name>/
  go.mod           # module github.com/konveyor/<provider-name>
                   # requires github.com/konveyor/analyzer-lsp
  main.go          # gRPC server entry point
  pkg/             # provider implementation
  Dockerfile       # standalone build (standard Go module, no monorepo COPY)
  .github/workflows/
    image-build.yaml   # multi-arch image build (reuse release-tools)
    pr-testing.yml     # unit tests + e2e
    pr-closed.yaml     # cherry-pick automation
  e2e-tests/       # provider-specific e2e tests
```

Key changes per provider:
- **go.mod:** Module path changes from `github.com/konveyor/analyzer-lsp/external-providers/<name>` to `github.com/konveyor/<name>`.
- **Dockerfile:** No longer copies the entire monorepo. Standard Go module build that fetches dependencies via `go mod download`.
- **CI workflows:** Each repo gets its own image build, PR testing, and cherry-pick workflows modeled after existing release-tools patterns.

### Special Cases

- **java-external-provider:** Continues to use `quay.io/konveyor/jdtls-server-base` (from `java-analyzer-bundle` repo) as its Dockerfile base image. The `repository_dispatch` trigger for rebuilding when `jdtls-server-base` updates moves from analyzer-lsp to this repo.

### User Stories

#### Story 1: Provider Developer

A developer working on the Python provider can clone `konveyor/python-external-provider`, make changes, open a PR, and get CI feedback specific to that provider in minutes. They do not need to wait for Java, Go, or YAML provider tests to pass, and their PR does not appear in the analyzer-lsp review queue.

#### Story 2: Core Engine Developer

A developer making changes to analyzer-lsp's rule engine can focus on engine tests without rebuilding all provider images. If their change affects the provider interface (rare -- the gRPC protocol is stable), they update the protobuf definitions in analyzer-lsp and then open coordinated PRs in the affected provider repos.

#### Story 3: Release Manager

When a critical bug is found in the Java provider, the team can fix, test, and release a new `java-external-provider` image without cutting a new analyzer-lsp release. Downstream consumers (kantra, operator) update only the Java provider image tag.

### Security, Risks, and Mitigations

No new security concerns are introduced. The gRPC protocol, authentication (JWT), and TLS configuration remain unchanged. Each provider repo inherits the same CI security scanning and OWNERS-based review that analyzer-lsp uses today.

## Design Details

### Changes to analyzer-lsp

- Delete `external-providers/` directory.
- Remove Makefile targets: `external-go-provider`, `external-python-provider`, `external-nodejs-provider`, `yq-external-provider`, `java-external-provider`, and the `run-external-providers-*` targets.
- **`image-build.yaml`:** Remove provider matrix entries. Keep only `analyzer-lsp` and `analyzer-lsp-windows` image builds.
- **`demo-testing.yml`:** Remove provider build/test phases (build-bases, build-all-providers, provider-tests, demo test). Analyzer-lsp PR testing focuses on the engine and built-in provider. The full integration test (all providers in a pod) moves to the CI repo.
- **`pr-testing.yml`:** Remove the java-external-provider test step.
- **`java-provider-image-build.yaml`:** Delete (moves to the java-external-provider repo).

### Changes to CI Repository (`konveyor/ci`)

#### `nightly-matrix-config.yaml`

Provider entries change from pointing into analyzer-lsp's subdirectories to pointing at standalone repos:

**Before:**
```yaml
- repo: "konveyor/analyzer-lsp"
  image: "quay.io/konveyor/go-external-provider"
  dockerfile_path: "./external-providers/go-external-provider/Dockerfile"
  context_path: "."
  go_mod_update:
    - "github.com/konveyor/analyzer-lsp@BRANCH_PLACEHOLDER"
  go_mod_directory: "./external-providers/go-external-provider"
```

**After:**
```yaml
- repo: "konveyor/go-external-provider"
  image: "quay.io/konveyor/go-external-provider"
  dockerfile_path: "Dockerfile"
  context_path: "."
```

The `analyzer-lsp` entry in the matrix config loses its provider-related `dependent_jobs`. It keeps `tackle2-addon-analyzer` and `c-sharp-analyzer-provider` as dependents.

The `kantra` entry's `go_mod_update` for `github.com/konveyor/analyzer-lsp/external-providers/java-external-provider@BRANCH_PLACEHOLDER` is removed (kantra does not actually import from that module).

#### `nightly-matrix-config-0.9.yaml`

Same structural changes for the release-0.9 config (which still uses `generic-external-provider` + `golang-dependency-provider` per that branch's provider layout).

#### Koncur test scripts

`koncur-kantra/check_images.sh` and `koncur-tackle-hub/check_images.sh` check for provider images and download nightly builds if missing. Image names stay the same, so changes are minimal -- only the nightly download URL construction needs to account for the new source repos if it uses repo names in the URL.

#### Integration test workflow

Create a new workflow (or extend `nightly-koncur.yaml`) that runs the "all providers in a pod" demo test currently in analyzer-lsp's `demo-testing.yml`. This test pulls pre-built provider images and validates they work together end-to-end.

### Changes to Downstream Consumers

#### kantra

**No Go import changes needed.** kantra imports `github.com/konveyor/analyzer-lsp/provider` and `github.com/konveyor/analyzer-lsp/output/v1/konveyor` -- these packages stay in analyzer-lsp.

**Dockerfile:** No changes. It references provider images by `quay.io/konveyor/` names in `FROM` stages; the source repo is transparent.

#### koncur

**No Go import changes needed.** koncur imports the same two analyzer-lsp packages, which stay in analyzer-lsp.

**Config/Makefile:** Image names stay the same. No changes.

#### tackle2-addon-analyzer

**No changes needed.** Depends on analyzer-lsp for engine/core. Providers are loaded at runtime via Hub extensions.

#### operator

**Helm values:** Image names unchanged. No changes needed.

#### java-analyzer-bundle

**No changes needed.** Builds `jdtls-server-base` independently.

### Execution Order

The extraction must be sequenced to avoid breaking CI across repos.

**Step 1: Extract providers one at a time**, starting with the simplest:
1. `yq-external-provider` -- standalone, no other provider depends on it.
2. `go-external-provider` -- standalone.
3. `python-external-provider` -- standalone.
4. `nodejs-external-provider` -- standalone.
5. `java-external-provider` -- most complex; depends on `jdtls-server-base`, has `repository_dispatch` trigger.

For each provider:
1. Create the new repository with code, Dockerfile, and CI workflows.
2. Verify the provider image builds and publishes to `quay.io/konveyor/`.
3. Update CI `nightly-matrix-config.yaml` to point to the new repo.
4. Run koncur tests to validate end-to-end.
5. Remove the provider from analyzer-lsp's `external-providers/`.
6. Update analyzer-lsp CI workflows to remove that provider's build/test steps.

**Step 2: Clean up analyzer-lsp.**
Remove the now-empty `external-providers/` directory, strip remaining provider CI workflow phases, and clean up the Makefile.

**Step 3: Finalize the CI repo.**
Update `nightly-matrix-config.yaml` with all providers as independent repos, add the integration test workflow, update cross-repo testing scripts and PR override mechanisms, and validate the full nightly pipeline.

### Test Plan

**Per-provider (on each extraction):**
- Unit tests: `go test ./...` in the new provider repo.
- Image builds: verify the provider image builds and pushes to `quay.io/konveyor/`.
- Provider e2e tests: run the provider's own e2e tests (podman pod + analyzer), migrated from analyzer-lsp's `demo-testing.yml`.

**Cross-repo integration (after each extraction):**
- Koncur kantra tests: run the full koncur test suite with the kantra target.
- Koncur hub tests: run the full koncur test suite with the tackle-hub target.
- Nightly pipeline: validate the CI repo's nightly build-and-test pipeline still passes.
- Cross-repo PR testing: verify `/ci test-with` overrides work with the new repo.

**Final validation (after all providers extracted):**
- Full nightly pipeline with all providers building from their own repos.
- Integration test workflow in the CI repo (all providers in a pod) passes.
- kantra Dockerfile builds successfully, pulling all provider images.
- Operator deployment with Helm values resolves all provider images.

### Upgrade / Downgrade Strategy

This is a development infrastructure change, not a runtime feature. There is no upgrade/downgrade path for end users.

For integrators (kantra, operator, CI):
- Container image names remain unchanged, so existing deployments work without modification.
- Go module import paths for kantra and koncur remain unchanged (they import from analyzer-lsp, not from provider modules).
- The only changes are in CI configuration (which repo builds which image) and are handled as part of the extraction steps.

## Implementation History

- 2026-03-13: [Generic-to-specific-providers enhancement](/enhancements/enhancements/generic-to-specific-providers/generic-provider-to-specific-providers.md) proposed.
- 2026-07: Generic provider split completed -- `generic-external-provider` replaced by `go-external-provider`, `python-external-provider`, `nodejs-external-provider`. `golang-dependency-provider` removed.
- 2026-07-31: This enhancement proposed.

## Drawbacks

- **More repositories to manage.** Five new repos means more OWNERS files, CI workflows, branch policies, and release processes. Mitigated by reusing `konveyor/release-tools` patterns for CI and cherry-pick automation.
- **Cross-repo coordination for interface changes.** If the provider interfaces in analyzer-lsp change, all provider repos need coordinated updates. Mitigated by the gRPC protobuf providing a stable wire protocol -- interface changes are rare.
- **Harder local development.** Developers who need to test a provider change against an unreleased analyzer-lsp change must use `go mod replace` directives locally. This is the same workflow used by `c-sharp-analyzer-provider` and `java-analyzer-provider` (Rust) today.
- **Loss of monorepo-level testing.** The "all providers in a pod" integration test can no longer run on every analyzer-lsp PR. Mitigated by moving it to the CI repo as a nightly/on-demand workflow.

## Alternatives

1. **Extract a separate "provider SDK" module.** Move shared interfaces, types, protobuf definitions, and utilities into a new `konveyor/provider-sdk` repo. Both analyzer-lsp and providers depend on it. This was considered and rejected: the dependency is already one-directional (providers → analyzer-lsp), so a separate SDK adds coordination overhead without technical necessity. Providers do import several analyzer-lsp packages (`provider/`, `engine/`, `lsp/protocol/`, etc.), but these are the shared interfaces and types that define the provider contract -- extracting them into a third repo would not reduce coupling, only add a coordination layer.

2. **Use a Go workspace (multi-module monorepo).** Keep all providers in analyzer-lsp but make each a fully independent Go module with workspace-level `go.work` for local development. This preserves monorepo benefits (atomic cross-cutting changes, shared CI) but does not achieve independent versioning, CI, or ownership. It also does not reduce the review queue or CI time.

3. **Keep everything in the monorepo.** The generic-to-specific split is complete. Leave all providers under `external-providers/` as they are. This preserves monorepo testing and atomic changes but does not achieve the organizational and CI velocity goals.

## Infrastructure Needed

- Five new GitHub repositories under the `konveyor` organization.
- Quay.io image repositories (already exist -- the image names are unchanged).
- CI runner access for the new repos (same pool used by other Konveyor repos).
- Branch protection and OWNERS configuration per repo.
