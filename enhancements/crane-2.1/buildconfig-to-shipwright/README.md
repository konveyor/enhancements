---
title: buildconfig-to-shipwright
authors:
  - "@aufi / maufart"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-07-21
last-updated: 2026-07-30
status: provisional
see-also:
  - "/enhancements/crane-2.0/multi-stage-kustomize-transforms"
  - "https://github.com/migtools/crane-lib/blob/main/convert/buildconfigs.go"
  - "https://shipwright.io/docs/build/build/"
replaces: []
superseded-by: []
---

# BuildConfig to Shipwright Conversion Plugin

OpenShift `BuildConfig` (`build.openshift.io/v1`) is a platform-specific CI/CD resource with no equivalent in vanilla Kubernetes. This enhancement introduces a new crane transform plugin — `BuildConfigToShipwrightPlugin` — that converts BuildConfig resources to [Shipwright](https://shipwright.io/) Build CRs (`shipwright.io/v1beta1`) as part of crane's standard export → transform → apply workflow, following crane's GitOps-friendly, offline, auditable transformation principles.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. **ImageStream resolution strategy:** When a BuildConfig references an ImageStreamTag for its base image or output, the plugin needs to resolve this to a concrete registry URL. Decision: use a layered fallthrough approach (1 → 2 → 3):
   1. **Explicit mapping** — if user provides a `--registry-mapping` flag, use it (highest priority, critical for cross-cluster migrations)
   2. **Co-exported ImageStream data** — passed to the plugin as an optional flag value (e.g., `--imagestream-mapping`) by the caller who has access to the export directory; the plugin itself only uses stdin/stdout and passed flags
   3. **Fallback** — construct internal OpenShift registry URL (`image-registry.openshift-image-registry.svc:5000/<ns>/<name>:<tag>`) with a warning (only useful when target is also OpenShift 4.x)

2. **Shipwright strategy version pinning:** Decision: default to upstream ClusterBuildStrategy names (`buildah`, `source-to-image`) but allow overrides via optional flags (e.g., `--default-build-strategy`), since cluster admins may customize strategies and downstream operators may use different naming conventions.

3. **BuildRun generation:** Should the plugin also generate a BuildRun CR (the Shipwright equivalent of triggering a build), or leave that to the user? Decision: leave it to the user — creating a BuildRun triggers an actual build, which is dangerous at scale. Build lifecycle should be managed independently by the ops engineer or external CI/CD system.

## Summary

Organizations migrating to newer versions of OpenShift to vanilla Kubernetes might need to convert their `BuildConfig` resources to a portable, cloud-native alternative. Shipwright is a CNCF Sandbox project that provides a Kubernetes-native build framework and is the natural successor for OpenShift's build system.

This enhancement adds a new crane transform plugin that performs offline, deterministic conversion of BuildConfig resources to Shipwright Build CRs. The plugin integrates into crane's multi-stage transformation pipeline, producing reviewable YAML artifacts with full whiteout and patch trail — no live cluster connectivity is required during transformation.

An earlier PoC implementation exists in [crane-lib/convert/buildconfigs.go](https://github.com/migtools/crane-lib/blob/main/convert/buildconfigs.go) implementing direct API-based conversion via `crane convert`. This enhancement replaces that approach with the plugin-based, GitOps-friendly architecture.

## Motivation

BuildConfig is one of the most complex OpenShift-specific resources. Unlike Routes (→ Ingress) or DeploymentConfigs (→ Deployment), build definitions carry significant semantic weight — source references, build strategies, registry credentials, and output image targets — that must be carefully mapped to their Shipwright equivalents.

Many organizations running OpenShift 3.x/4.x have dozens to hundreds of BuildConfigs. Manual conversion is error-prone and time-consuming. An automated, reviewable conversion within crane's existing workflow significantly accelerates migration timelines.

### Why Shipwright

- CNCF Sandbox project
- Kubernetes-native: works on any cluster with Tekton installed
- Supports the same build paradigms: Dockerfile/Buildah builds, Source-to-Image (S2I), Buildpacks
- Clean API (`shipwright.io/v1beta1`) designed for extensibility via BuildStrategy CRDs
- Active community and growing adoption

### Why a Transform Plugin (Not `crane convert`)

The existing PoC uses a direct API approach: it queries the live source cluster, resolves ImageStream references in real-time, and outputs converted resources. This violates crane's core design principles:

| Aspect | `crane convert` (PoC) | Transform Plugin (this proposal) |
|--------|----------------------|----------------------------------|
| Cluster connectivity | Required during conversion | Not required (offline) |
| Auditability | Output only, no patch trail | Full Patch + whiteout trail |
| GitOps workflow | Separate from export/transform/apply | Integrated into standard pipeline |
| Idempotency | Depends on cluster state | Deterministic from exported YAML |
| Reviewability | Binary before/after | Reviewable patches in transform |
| Multi-stage pipeline | Standalone command | Composable with other plugins |

### Goals

- Convert the main BuildConfig strategy types (Docker, S2I) to Shipwright Build CRs within crane's transform pipeline
- Map source configuration (Git, Binary, Image), output image references, and registry credentials to Shipwright equivalents
- Generate related resources where needed (ServiceAccount with pull/push secrets)
- Whiteout the original BuildConfig and produce reviewable YAML artifacts
- Clearly document unsupported fields via warnings and annotations on the output CR
- Extend the crane plugin API to support generating new resources (prerequisite)

### Non-Goals

- JenkinsPipeline strategy conversion (deprecated — users should migrate to Tekton Pipelines directly)
- Live cluster API calls during transformation
- Automatic Shipwright/Tekton installation on the target cluster
- BuildConfig trigger conversion (Shipwright triggers are a separate mechanism)
- Build history migration (Build objects are ephemeral)
- ImageStream migration (handled by the separate `skopeo` tool)

## Proposal

### User Stories

#### Story 1: Docker Strategy BuildConfig Migration

**As a** platform engineer migrating an application from OpenShift to vanilla Kubernetes, **I want to** run `crane transform 30_BuildConfigPlugin` on my exported namespace, **so that** my Dockerfile-based BuildConfig is automatically converted to a Shipwright Build using the `buildah` ClusterBuildStrategy, with Dockerfile path, build args, and base image correctly mapped, and the original BuildConfig marked for deletion.

#### Story 2: S2I Strategy BuildConfig Migration

**As a** platform engineer migrating a legacy S2I-based application, **I want to** run the BuildConfig plugin, **so that** my Source strategy BuildConfig is converted to a Shipwright Build using the `source-to-image` ClusterBuildStrategy, with the builder image reference, environment variables, and source secret correctly mapped.

#### Story 3: Batch Migration of Multiple BuildConfigs

**As a** migration engineer handling a namespace with 20+ BuildConfigs, **I want to** run the plugin once and have all BuildConfigs converted in a single transform stage, **so that** I can review all conversions together in Git, address warnings for unsupported features, and apply the complete migration in one `crane apply` operation.

#### Story 4: Review and Customize Converted Builds

**As a** platform engineer, **I want to** inspect the generated Shipwright Build YAMLs in the transform stage output directory before applying, **so that** I can verify the conversion, add custom stages for manual adjustments if needed, and commit the full transformation trail to Git for team review via PR.

### Implementation Details/Notes/Constraints

#### Plugin System Prerequisite

This plugin requires the crane plugin API extension that enables generating new resources (adding `NewResources []unstructured.Unstructured` to `PluginResponse`). This extension is backward compatible on the JSON wire format via the `omitempty` tag (existing plugins that omit `NewResources` continue to work unchanged). Note that adding the field to the Go struct will break any downstream code using unkeyed `PluginResponse{...}` literals — those must be updated to use keyed fields. Details in a [separate plan](https://github.com/aufi/move-crane/blob/main/drafts/plugin-update-new-resource-plan.md). Implementation touches:

- **crane-lib:** `transform/plugin.go` (PluginResponse), `transform/runner.go` (RunnerResponse, Runner.Run)
- **crane:** `Orchestrator` (artifact creation for new resources) and potentially `Writer` (plugin dirs update).


#### Integration with Crane Workflow

```text
crane export -n myapp
    ↓
    export/resources/
    ├── BuildConfig_build.openshift.io_v1_myapp_myapp-build.yaml
    ├── Deployment_apps_v1_myapp_myapp.yaml
    ├── ImageStream_image.openshift.io_v1_myapp_myapp.yaml  (if present)
    └── ...
    ↓
crane transform  (multi-stage pipeline)
    ↓
    Stage 10_KubernetesPlugin     → clean metadata, cluster-specific fields
    Stage 20_OpenshiftPlugin      → Route→Ingress, other OCP conversions
    Stage 30_BuildConfigPlugin    → BuildConfig→Shipwright Build  ← THIS ENHANCEMENT
    ↓
    transform/30_BuildConfigPlugin/
    ├── resources/
    │   ├── BuildConfig_...myapp-build.yaml                      (whiteout)
    │   ├── Build_shipwright.io_v1beta1_myapp_myapp-build.yaml   (NEW, skeleton only)
    │   └── ...other resources unchanged...
    ├── patches/
    │   └── myapp--build.openshift.io-v1--BuildConfig--myapp-build.patch.yaml
    └── kustomization.yaml
    ↓
crane apply  →  output/resources/  →  kubectl apply -f
```

#### Conversion Logic

The plugin processes each resource in the stage input:

1. **Filter:** Skip non-BuildConfig resources (return empty response)
2. **Whiteout** the original BuildConfig (mark for deletion via `IsWhiteOut: true`)
3. **Generate** a new Shipwright Build CR with mapped fields (unsupported strategies fail the conversion with a clear error and warning)
4. **Optionally generate** a ServiceAccount if pull/push secrets are referenced (scaffolded for the user to wire into their own BuildRun definitions)
5. Return the new resource(s) via `NewResources` in PluginResponse

#### Field Mapping

**Strategy mapping:**

| BuildConfig Strategy | Shipwright ClusterBuildStrategy | Notes |
|---------------------|-------------------------------|-------|
| `dockerStrategy` | `buildah` | Dockerfile path, build args, base image mapped as paramValues |
| `sourceStrategy` (S2I) | `source-to-image` | Builder image mapped as paramValue, env vars copied directly |
| `customStrategy` | _(no conversion)_ | Conversion fails with warning |
| `jenkinsPipelineStrategy` | _(out of scope)_ | Conversion fails with warning |

**Source mapping:**

| BuildConfig Source | Shipwright Source | Notes |
|-------------------|------------------|-------|
| `git.uri` + `git.ref` | `source.type: Git`, `source.git.url` + `revision` | Direct mapping |
| `git.httpProxy/httpsProxy` | _(not directly supported)_ | Shipwright's Git clone runs in a separate container; proxy must be configured via `GIT_CONTAINER_TEMPLATE` at the cluster level. Warning emitted. |
| `sourceSecret` | `source.git.cloneSecret` | Direct mapping |
| `contextDir` | `source.contextDir` | Direct mapping |
| `binary` | `source.type: Local` | Requires manual `shp build upload` after apply |
| `images[0]` | `source.type: OCIArtifact` | Single image source only |
| `dockerfile` (inline) | _(not supported)_ | Warning: write to file in repo |

**Output mapping:**

| BuildConfig Output | Shipwright Output | Notes |
|-------------------|------------------|-------|
| `output.to` (DockerImage) | `output.image` | Direct reference |
| `output.to` (ImageStreamTag) | `output.image` | Resolved via layered fallthrough: explicit `--registry-mapping`, co-exported ImageStream YAML, or internal registry fallback. |
| `output.pushSecret` | `output.pushSecret` | Direct mapping |

**Known unsupported fields** — the plugin emits clear warnings for each:

| Feature | Note |
|---------|------|
| Docker volumes | Shipwright doesn't support build-time volumes |
| S2I custom scripts | Not available in Shipwright S2I strategy |
| Incremental builds | Not supported |
| ConfigMaps/Secrets as source | Not available |
| ForcePull | No equivalent |
| Image squash (--squash) | No equivalent |
| Multiple image sources | Shipwright supports single source only |
| Resource limits (CPU/memory) | Shipwright supports resource overrides but mapping is not implemented in this version |

#### Conversion Example

**Input** — OpenShift BuildConfig:

```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: myapp-build
  namespace: myapp
spec:
  source:
    type: Git
    git:
      uri: https://github.com/example/myapp.git
      ref: main
    contextDir: src
    sourceSecret:
      name: git-credentials
  strategy:
    type: Docker
    dockerStrategy:
      dockerfilePath: Dockerfile.prod
      buildArgs:
        - name: GO_VERSION
          value: "1.21"
      from:
        kind: DockerImage
        name: golang:1.21-alpine
  output:
    to:
      kind: DockerImage
      name: quay.io/example/myapp:latest
    pushSecret:
      name: quay-push-secret
```

**Output** — Shipwright Build:

```yaml
apiVersion: shipwright.io/v1beta1
kind: Build
metadata:
  name: myapp-build
  namespace: myapp
  annotations:
    crane.konveyor.io/converted-from: build.openshift.io/v1/BuildConfig/myapp-build
spec:
  source:
    type: Git
    git:
      url: https://github.com/example/myapp.git
      revision: main
      cloneSecret: git-credentials
    contextDir: src
  strategy:
    name: buildah
    kind: ClusterBuildStrategy
  paramValues:
    - name: dockerfile
      value: Dockerfile.prod
    - name: build-args
      values:
        - value: "GO_VERSION=1.21"
    - name: from
      value: golang:1.21-alpine
  output:
    image: quay.io/example/myapp:latest
    pushSecret: quay-push-secret
```

#### Plugin Repository Structure (draft)

```text
crane-plugin-buildconfig-to-shipwright/
├── buildconfig/converter.go         # BuildConfig → Shipwright Build mapping
├── buildconfig/converter_test.go    # Unit tests for each strategy/source type
├── testdata/            # Sample BuildConfig YAMLs for testing
├── main.go              # Plugin entry point, GVK filter
├── go.mod
└── README.md
```

Plugin flags:
- `--search-registries` — comma-separated search registries for image resolution
- `--insecure-registries` — comma-separated insecure registries
- `--block-registries` — comma-separated blocked registries
- `--strategy-mapping` — override default ClusterBuildStrategy names (e.g., `docker=my-buildah,s2i=my-s2i`)

### Security, Risks, and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Shipwright not installed on target cluster** | Build CRs fail on `kubectl apply` | Validate command should check it, document Shipwright installation in user guide |
| **ImageStreamTag references can't be resolved offline** | Output image URL incomplete | Use exported ImageStream data; if resolution fails through all layers (explicit mapping, co-exported ImageStream, internal registry fallback), fail the conversion with a clear error |
| **BuildConfig uses unsupported features** | Incomplete conversion | Emit warnings per unsupported field; annotate output CR with `crane.konveyor.io/warnings` listing each gap |
| **Plugin API extension (prerequisite) delayed** | Blocks plugin development | Plugin can be developed against a local crane-lib branch in parallel |
| **Registry credentials in Secrets** | Credentials copied to output directory | No change from current crane behavior — Secrets are already exported as-is. Users should exclude Secrets from Git commits (e.g., via `.gitignore` or external secret management) and avoid committing sensitive manifests to the transformation trail. Secret redaction in crane's export pipeline is a desirable future improvement but out of scope for this enhancement. |

## Design Details

### Implementation Phases

| Phase | Scope | Repositories |
|-------|-------|-------------|
| **1 — Plugin API Extension** | Add `NewResources` to PluginResponse/RunnerResponse, update Runner and Orchestrator | crane-lib, crane |
| **2 — BuildConfig Plugin** | New plugin: conversion logic, tests, testdata | new crane-plugin-buildconfig |
| **3 — Crane Workflow Updates** | Export filtering by GVK, plugin execution control in transform | crane |
| **4 — Documentation** | Plugin development guide, AI-friendly examples, reference implementation docs | crane, plugin, docs |

### Test Plan

**Unit tests (crane-lib):**
- `TestRunnerWithNewResources` — plugin generates 1 new resource
- `TestRunnerWhiteoutWithNewResource` — plugin whiteouts original + generates replacement
- `TestRunnerBackwardCompatibility` — old plugin (without NewResources) works unchanged

**Unit tests (plugin):**
- Docker strategy conversion (Dockerfile path, build args, base image, env vars)
- S2I strategy conversion (builder image, env vars, source secret)
- Custom strategy passthrough with warning
- Git source mapping (URL, ref, contextDir, cloneSecret, proxy config)
- Binary source mapping (Local type)
- Image source mapping (OCIArtifact type)
- Output mapping: DockerImage direct reference
- Output mapping: ImageStreamTag resolution from exported data
- ServiceAccount generation when pull/push secrets present
- Unsupported fields produce appropriate warnings

**Integration tests (crane):**
- End-to-end: `crane export` → `crane transform 30_BuildConfigPlugin` → `crane apply` with sample BuildConfigs
- Multi-stage: BuildConfig plugin output flows correctly through subsequent stages
- Backward compatibility: existing plugins unaffected when BuildConfig plugin is added

**E2E tests:**
- Export from OpenShift cluster with BuildConfigs → transform → apply to Shipwright-enabled cluster → verify builds succeed

### Upgrade / Downgrade Strategy

**Upgrade:** The plugin is opt-in — users install it upstream via `crane plugin-manager` and add a transform stage. Existing workflows are completely unaffected.

**Downgrade:** Removing the plugin simply removes the BuildConfig conversion stage. Previously converted output remains valid (it is standard YAML). The plugin API extension in crane-lib is backward compatible — removing it would require a crane-lib downgrade, but existing plugins would continue to work without the `NewResources` field.

## Implementation History

- `2026-07-21`: Enhancement proposed as `provisional`

## Drawbacks

- **Incomplete coverage:** Not all BuildConfig features have Shipwright equivalents. Some conversions will require manual follow-up. This is inherent to the API gap between the two systems, not a flaw in the plugin approach.
- **Prerequisite dependency:** The plugin API extension must land first, which gates the plugin development timeline. However, both can be developed in parallel against a local branch.
- **Shipwright adoption:** If organizations choose a different build system (e.g., Tekton Pipelines directly, GitHub Actions), this plugin is not useful. However, Shipwright's CNCF status and Red Hat backing make it the most likely target for OpenShift build migration.

## Alternatives

### 1. Keep Using `crane convert` (Direct API Approach)

The existing PoC in crane-lib works but requires live cluster connectivity during conversion, produces no transformation trail, and doesn't integrate with crane's multi-stage pipeline. Rejected because it violates crane's GitOps-first design principles.

### 2. Kustomize-Based Conversion (No Go Plugin)

A shell script + Kustomize approach could handle simple Dockerfile-only cases but lacks the semantic understanding needed for strategy mapping, ImageStream resolution, and ServiceAccount generation. Better suited as a lightweight alternative for simple builds, not as the primary conversion path.

### 3. Manual Conversion with Templates

Provide YAML templates and let users convert manually. This doesn't scale for organizations with many BuildConfigs and defeats the purpose of automated migration tooling.

## Infrastructure Needed

- **New repository:** `migtools/crane-plugin-buildconfig-to-shipwright` — the plugin implementation
- **Existing repositories modified:** `migtools/crane-lib` (plugin API extension), `migtools/crane` (orchestrator update, workflow improvements)
- **CI:** Standard Go CI pipeline for the new plugin repository; E2E tests require an OpenShift cluster with BuildConfigs and a target cluster with Shipwright installed
