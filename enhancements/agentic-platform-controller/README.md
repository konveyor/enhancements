---
title: agentic-platform-controller
authors:
  - "@djzager"
reviewers:
  - TBD
  - "@dymurray"
approvers:
  - TBD
creation-date: 2026-06-17
last-updated: 2026-06-17
status: provisional
see-also:
  - "/enhancements/kai/agent-driven-migration/README.md"
  - "https://github.com/redhat-et/skillimage"
  - "https://github.com/kubernetes-sigs/agent-sandbox"
  - "https://agentskills.io"
replaces: []
superseded-by: []
---

# Agentic Platform Controller

A Kubernetes-native controller for managing AI agent workloads. Introduces
seven CRDs under the `konveyor.io` API group and six controllers in a new
`konveyor/agentic-controller` repository. Agents are composed from portable
skill packages, LLM provider configurations, and container images, then
executed as Agent Sandbox workloads. The controller handles skill resolution
(OCI, git, inline markdown), LLM provider verification, context budget
computation, and execution lifecycle management.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. **Model role names**: The AgentRun selects models with roles (e.g.
   `primary`, `efficient`). The base image harness maps roles to
   runtime-specific env vars. Should the set of recognized roles be
   fixed by convention (a known set like `primary`, `planner`,
   `efficient`) or free-form (any string, mapped by the harness)?

2. **SkillCard content hashing**: When a SkillCard uses inline content,
   the controller builds an OCI artifact and pushes it. Should it use
   content-addressable tags (hash of the content) to avoid rebuilding
   unchanged skills, or always rebuild on spec change?

3. **LLMProvider verification frequency**: The LLMProvider controller
   launches a container to verify connectivity. Should this happen only
   on create/update, or periodically (e.g. every N minutes) to detect
   endpoint outages?

4. **Agent Sandbox API stability**: Agent Sandbox is v1beta1. What is the
   project's timeline to v1? If the API changes significantly before our
   v1, what is the migration cost?

## Summary

The Konveyor agentic platform defines a set of Kubernetes CRDs for
composing and executing AI agent workloads. The platform separates
capability definition (what skills, providers, and models are available)
from execution (specific selections for a particular run). Skills are
sourced from OCI registries, git repositories, or inline content and
resolved to OCI artifacts by the controller. Agent workloads run in
Agent Sandbox pods with skills mounted as ImageVolumes.

This enhancement covers the controller that reconciles these CRDs. The
IDE-side agent integration is covered by the
[agent-driven-migration](/enhancements/kai/agent-driven-migration/README.md)
enhancement. Agent container image composition (how language-specific
images are built) is out of scope and will be a separate enhancement.

## Motivation

### Current State

The Konveyor POC for AI-powered migration (tackle2-addon-kai) uses Hub's
addon framework to launch agent containers. This couples agent execution
to Hub's task lifecycle, requires Hub API changes for every new resource
type, and makes the agent platform dependent on Hub's availability as
a workload launcher.

The POC also uses pallet to sync skills from git repos at container
startup, adding network dependency and startup latency. Skills are not
versioned as artifacts, making reproducibility and supply chain security
difficult.

### Goals

1. **Kubernetes-native agent management**: Define agent capabilities and
   execute agent workloads using standard Kubernetes CRDs, tooling
   (kubectl, GitOps, RBAC), and patterns (controller-runtime).

2. **Skill lifecycle management**: Resolve skills from OCI registries,
   git repositories, or inline content to versioned OCI artifacts. Use
   skillimage's packaging format and Go library for OCI operations.

3. **LLM provider verification**: Validate that configured LLM endpoints
   are reachable and models are available before accepting agent
   execution requests.

4. **Portable execution**: Agent workloads run in Agent Sandbox pods,
   decoupled from Hub. Hub remains a runtime data service (analysis
   results, app metadata) but does not launch or manage agent workloads.

5. **Runtime-agnostic CRDs**: The CRDs do not encode assumptions about
   specific agent runtimes (goose, opencode). Runtime selection is
   determined by the container image. The base image harness translates
   generic controller inputs to runtime-specific configuration.

6. **MVP scope**: Ship SkillCard, SkillCollection, LLMProvider, Agent,
   and AgentRun as the initial release. These five CRDs and four
   controllers enable single-agent execution end to end.

### Non-Goals

1. **Agent container image composition**: How language-specific agent
   images are built (devcontainers, buildpacks, fat images, etc.) is a
   separate enhancement. This enhancement assumes images exist and are
   selected by the user.

2. **Tekton integration**: Tekton Pipelines integration for multi-phase
   orchestration is future work. The MVP uses a simple sequential
   controller for AgentPlaybookRun (post-MVP).

3. **Skill authoring tools**: How users author, test, and publish skills
   is the domain of the skillimage project. This enhancement consumes
   skills; it does not provide authoring tools.

4. **Hub API changes**: No new Hub API endpoints, database tables, or Go
   models are required. The UI talks directly to the Kubernetes API for
   CRD operations via a proxy route.

## Proposal

### Architecture Overview

```
                    ┌─────────────────────────────────────────────┐
                    │          Kubernetes API Server               │
                    │                                             │
                    │  SkillCard    SkillCollection   LLMProvider │
                    │  Agent       AgentPlaybook                      │
                    │  AgentRun    AgentPlaybookRun                   │
                    └──────────────┬──────────────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────────────┐
                    │      konveyor/agentic-controller            │
                    │                                             │
                    │  ┌──────────────┐  ┌────────────────────┐  │
                    │  │ SkillCard    │  │ SkillCollection    │  │
                    │  │ Controller   │  │ Controller         │  │
                    │  │              │  │                    │  │
                    │  │ OCI ref:     │  │ Creates child      │  │
                    │  │  validate    │  │ SkillCard CRs for  │  │
                    │  │ Git source:  │  │ git-sourced entries│  │
                    │  │  clone/build │  │                    │  │
                    │  │  /push       │  │ Reports ready when │  │
                    │  │ Inline:      │  │ all children       │  │
                    │  │  build/push  │  │ resolved           │  │
                    │  └──────┬───────┘  └────────────────────┘  │
                    │         │                                   │
                    │         ▼ resolved OCI ref in status        │
                    │                                             │
                    │  ┌──────────────┐  ┌────────────────────┐  │
                    │  │ Agent        │  │ LLMProvider        │  │
                    │  │ Controller   │  │ Controller         │  │
                    │  │              │  │                    │  │
                    │  │ Watches deps │  │ Launches base      │  │
                    │  │ Computes     │  │ image container    │  │
                    │  │ context      │  │ to verify endpoint │  │
                    │  │ budget       │  │ connectivity       │  │
                    │  │ Reports      │  │                    │  │
                    │  │ readiness    │  │ Reports discovered │  │
                    │  └──────────────┘  │ models in status   │  │
                    │                    └────────────────────┘  │
                    │  ┌──────────────┐  ┌────────────────────┐  │
                    │  │ AgentRun     │  │ AgentPlaybookRun       │  │
                    │  │ Controller   │  │ Controller         │  │
                    │  │              │  │ (post-MVP)         │  │
                    │  │ Validates    │  │                    │  │
                    │  │ selections   │  │ Creates AgentRuns  │  │
                    │  │ Creates      │  │ sequentially per   │  │
                    │  │ Sandbox      │  │ phase              │  │
                    │  │ Tracks       │  │                    │  │
                    │  │ completion   │  │ Manages session +  │  │
                    │  └──────────────┘  │ workspace PVCs     │  │
                    │                    └────────────────────┘  │
                    └─────────────────────────────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────────────┐
                    │      Agent Sandbox (agents.x-k8s.io)       │
                    │                                             │
                    │  Sandbox with:                              │
                    │  - Agent container image (runtime + tools)  │
                    │  - ImageVolumes (skills as OCI artifacts)   │
                    │  - Secret env vars (LLM credentials)       │
                    │  - Controller env vars (instructions, etc.) │
                    │  - Workspace PVC                            │
                    └─────────────────────────────────────────────┘
```

### CRD Definitions

#### SkillCard

An individual agent skill or rule. The CRD spec adopts the
skillimage.io/v1alpha1 SkillCard metadata format in a
Kubernetes-idiomatic structure.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: SkillCard
metadata:
  name: maven-migration
  namespace: konveyor
spec:
  # Exactly one of image, source, or inline must be set
  image: quay.io/konveyor/skills/maven-migration:1.0.0

  # -- OR --
  source:
    url: https://github.com/konveyor/skills/tree/main/maven-migration
    ref: main  # optional, branch/tag/commit

  # -- OR --
  inline:
    content: |
      You must never use javax.* imports.
      Always replace with jakarta.* equivalents.

  # skillimage metadata (under spec, not metadata)
  displayName: Maven Migration
  version: "1.0.0"
  description: Migrates Maven POM files from Java EE to Jakarta EE.
  type: skill  # skill (default) or rule
  tags:
    - java
    - maven
    - migration
  authors:
    - name: Konveyor Team
  license: Apache-2.0

status:
  resolvedImage: quay.io/konveyor/skills/maven-migration:1.0.0
  conditions:
    - type: Ready
      status: "True"
```

**Controller behavior**:

| Source type | Action |
|---|---|
| `image` | Validate the OCI ref exists in the registry. Set `status.resolvedImage`. |
| `source` | Clone the git repo using skillimage `pkg/source.Resolve()`. Build the OCI image using `pkg/oci.Client.Build()`. Push to the configured registry. Set `status.resolvedImage`. |
| `inline` | Create a temporary skill directory with the content as `SKILL.md` and a generated `skill.yaml`. Build and push. Set `status.resolvedImage`. |

Re-reconciliation triggers: spec changes, referenced git repo changes
(periodic or webhook-driven).

#### SkillCollection

A group of skills referenced by OCI image, git source, or SkillCard CR
name.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: SkillCollection
metadata:
  name: java-migration-skills
  namespace: konveyor
spec:
  version: "1.0.0"
  description: Skills for Java EE to Quarkus migration
  skills:
    - name: maven-migration
      skillCardRef: maven-migration  # reference a SkillCard CR

    - name: javax-imports
      image: quay.io/konveyor/skills/javax-imports:1.0.0  # OCI ref

    - source: https://github.com/konveyor/skills/tree/main/ejb-to-cdi  # git

status:
  resolvedSkills:
    - name: maven-migration
      resolvedImage: quay.io/konveyor/skills/maven-migration:1.0.0
      ready: true
    - name: javax-imports
      resolvedImage: quay.io/konveyor/skills/javax-imports:1.0.0
      ready: true
    - name: ejb-to-cdi
      resolvedImage: registry.internal:5000/konveyor/ejb-to-cdi:abc123
      ready: true
  conditions:
    - type: Ready
      status: "True"
```

**Controller behavior**: For `skillCardRef` entries, watch the referenced
SkillCard's status. For `image` entries, validate the OCI ref. For
`source` entries, create a child SkillCard CR with the git source and
let the SkillCard controller handle resolution. Report aggregate
readiness.

#### LLMProvider

An LLM service endpoint with credentials and available models.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: LLMProvider
metadata:
  name: anthropic-provider
  namespace: konveyor
spec:
  endpoint: https://api.anthropic.com
  credentialRef:
    secretName: anthropic-credentials
    key: api-key
  models:
    - name: claude-sonnet-4-20250514
      contextWindow: 200000
      tier: premium
    - name: claude-haiku
      contextWindow: 200000
      tier: efficient

status:
  connectionVerified: true
  lastVerified: "2026-06-17T10:30:00Z"
  discoveredModels:
    - claude-sonnet-4-20250514
    - claude-haiku
  conditions:
    - type: Ready
      status: "True"
```

**Controller behavior**: On create/update, launch a short-lived container
using the known agent base image with provider credentials and endpoint
injected. The container runs a verification entrypoint that attempts to
list models or send a trivial request. The controller reads the result
and updates status with `connectionVerified` and `discoveredModels`.
This tests the real network path and credential chain, not a simulation.

#### Agent

A capability definition declaring what is available for execution.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: Agent
metadata:
  name: java-migration-agent
  namespace: konveyor
spec:
  image: quay.io/konveyor/agent-java-goose:latest
  prompt: |
    You are a Java migration specialist. You analyze Java EE
    applications and migrate them to Quarkus, following best
    practices for CDI, JAX-RS, and MicroProfile.

  providers:
    - ref: anthropic-provider
    - ref: openai-provider

  skillCards:
    - ref: maven-migration
    - ref: no-javax-imports

  skillCollections:
    - ref: java-migration-skills

  memory:
    enabled: true
    serviceURL: http://mempalace.konveyor.svc:8080

status:
  contextBudget:
    rulesContentTokens: 1500
    smallestContextWindow: 200000
    utilization: "0.75%"
  unresolvedDependencies: []
  conditions:
    - type: Ready
      status: "True"
```

**Controller behavior**: Watch all referenced SkillCards,
SkillCollections, and LLMProviders. Compute context budget by summing
token counts of all `type: rule` SkillCards against the smallest context
window across all referenced models. Report readiness when all
dependencies are resolved and context budget is within limits.

#### AgentRun

A request to execute a single Agent with specific selections.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: AgentRun
metadata:
  name: migrate-app-123
  namespace: konveyor
spec:
  agentRef: java-migration-agent

  models:
    - role: primary
      provider: anthropic-provider
      model: claude-sonnet-4-20250514
    - role: efficient
      provider: anthropic-provider
      model: claude-haiku

  instructions: |
    Migrate this application from Java EE 7 to Quarkus 3.x.
    Focus on EJB to CDI conversion first, then update the
    Maven POM, then fix javax imports.

  parameters:
    APP_ID: "123"
    HUB_BASE_URL: "https://hub.konveyor.svc"

status:
  sandboxName: migrate-app-123-sandbox
  sessionID: "sess_abc123"
  phase: Running
  startTime: "2026-06-17T10:35:00Z"
  conditions:
    - type: Succeeded
      status: "Unknown"
      reason: Running
```

**Controller behavior**:

1. Validate that selected providers and models are in the referenced
   Agent's available set.
2. Resolve all SkillCards and SkillCollections from the Agent to their
   OCI image refs.
3. Create a Sandbox (agents.x-k8s.io/v1beta1) with:
   - The Agent's container image
   - ImageVolumes for each resolved skill OCI artifact
   - Secret environment variables for LLM credentials from the selected
     providers
   - Environment variables for model role mappings
     (`KONVEYOR_MODEL_PRIMARY`, `KONVEYOR_MODEL_EFFICIENT`, etc.)
   - Environment variables for instructions and parameters
   - Workspace PVC mount
4. Watch the Sandbox's `Finished` condition.
5. On completion, read the session ID and results from convention paths
   on the workspace PVC.
6. Update AgentRun status.

#### AgentPlaybook (post-MVP)

A reusable playbook with stages and phases. Defined in the CRD spec
but controller implementation is deferred.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: AgentPlaybook
metadata:
  name: full-java-migration
  namespace: konveyor
spec:
  guide: |
    This plan migrates a Java EE application to Quarkus in three
    stages: analysis, migration, and validation.

  stages:
    - name: migration
      phases:
        - name: update-pom
          agentRef: java-migration-agent
          instructions: |
            Update the Maven POM to use Quarkus BOM and dependencies.

        - name: migrate-source
          agentRef: java-migration-agent
          instructions: |
            Migrate Java source files: EJB to CDI, javax to jakarta.

    - name: validation
      phases:
        - name: build-and-test
          agentRef: java-migration-agent
          instructions: |
            Build the project and run the test suite. Fix any failures.
```

#### AgentPlaybookRun (post-MVP)

A request to execute an AgentPlaybook. Controller implementation is
deferred.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: AgentPlaybookRun
metadata:
  name: migrate-app-123-full
  namespace: konveyor
spec:
  agentPlaybookRef: full-java-migration

  models:
    - role: primary
      provider: anthropic-provider
      model: claude-sonnet-4-20250514

  parameters:
    APP_ID: "123"
    HUB_BASE_URL: "https://hub.konveyor.svc"
```

### User Stories

#### Story 1: Platform admin configures skills and providers

A platform admin creates SkillCard CRs for the organization's migration
rules. Some are OCI artifacts from the team's registry, some are pulled
from a public git repo, and a few quick rules are defined inline:

```bash
kubectl apply -f skills/maven-migration.yaml      # OCI ref
kubectl apply -f skills/community-rules.yaml       # git source
kubectl apply -f skills/no-javax-rule.yaml         # inline content
kubectl apply -f collections/java-migration.yaml   # groups them
kubectl apply -f providers/anthropic.yaml           # LLM endpoint
```

The admin checks that skills resolved and the provider connected:

```bash
kubectl get skillcards
# NAME              TYPE    SOURCE   READY
# maven-migration   skill   oci      True
# community-rules   skill   git      True
# no-javax-rule     rule    inline   True

kubectl get llmproviders
# NAME                 VERIFIED   MODELS
# anthropic-provider   True       claude-sonnet-4-20250514, claude-haiku
```

#### Story 2: Architect creates an Agent

An architect composes an Agent from the available skills and providers:

```bash
kubectl apply -f agents/java-migration-agent.yaml
kubectl get agents
# NAME                    READY   CONTEXT-BUDGET   PROVIDERS
# java-migration-agent    True    0.75%            anthropic-provider, openai-provider
```

The Agent controller reports readiness once all dependencies resolve
and the context budget is within limits.

#### Story 3: Developer runs a migration

A developer (or the Konveyor UI on their behalf) creates an AgentRun:

```bash
kubectl apply -f runs/migrate-app-123.yaml
kubectl get agentruns
# NAME              AGENT                   PHASE     AGE
# migrate-app-123   java-migration-agent    Running   2m
```

The controller creates a Sandbox with the Agent's image, mounts skills
as ImageVolumes, injects LLM credentials, and starts the agent. The
developer monitors progress via the UI or kubectl.

### Implementation Details

#### Repository structure

```
konveyor/agentic-controller/
  api/
    v1alpha1/
      skillcard_types.go
      skillcollection_types.go
      llmprovider_types.go
      agent_types.go
      agentplaybook_types.go
      agentrun_types.go
      agentplaybookrun_types.go
      groupversion_info.go
      zz_generated.deepcopy.go
  internal/
    controller/
      skillcard_controller.go
      skillcollection_controller.go
      llmprovider_controller.go
      agent_controller.go
      agentrun_controller.go
      # agentplaybookrun_controller.go (post-MVP)
    registry/
      client.go          # OCI registry client (wraps skillimage pkg/oci)
      detect.go          # auto-detect OpenShift built-in registry
  config/
    crd/                 # generated CRD manifests
    rbac/                # RBAC for the controller
    manager/             # controller-manager deployment
  charts/
    agentic-controller/
      Chart.yaml
      values.yaml
      templates/
      charts/
        zot/             # optional subchart for dev environments
  cmd/
    manager/
      main.go
  Dockerfile
  go.mod
  go.sum
  Makefile
```

#### Key dependencies

| Dependency | Purpose |
|---|---|
| `sigs.k8s.io/controller-runtime` | Controller framework |
| `agents.x-k8s.io/v1beta1` | Agent Sandbox CRD types |
| `github.com/redhat-et/skillimage/pkg/oci` | OCI image build/push/pull |
| `github.com/redhat-et/skillimage/pkg/source` | Git source resolution |
| `github.com/redhat-et/skillimage/pkg/skillcard` | SkillCard parsing/validation |
| `github.com/redhat-et/skillimage/pkg/collection` | SkillCollection parsing |

#### In-cluster OCI registry

The controller pushes built skill artifacts (from git sources and inline
content) to an OCI registry. The registry URL is provided via
configuration (flag, ConfigMap, or environment variable).

**OpenShift**: The controller auto-detects the built-in image registry
at `image-registry.openshift-image-registry.svc:5000`. No additional
deployment needed.

**Vanilla Kubernetes**: The Helm chart includes Zot
(project-zot/zot-minimal) as an optional subchart, disabled by default.
Enable for development or environments without an existing registry. For
production, the konveyor operator (ansible-based) deploys Zot as part
of the platform installation.

The controller does not own registry lifecycle. If git or inline source
SkillCards are created without a configured registry, the controller
sets a clear error condition on the CR.

#### Base image and harness

Agent container images follow a layered hierarchy:

```
agent-base                  (UBI + harness + core skills + konveyor tools)
  ├── agent-base-goose      (extends base + goose binary)
  ├── agent-base-opencode   (extends base + opencode binary)
  ├── agent-java-goose      (extends goose + JDK + Maven + Gradle)
  ├── agent-java-opencode   (extends opencode + JDK + Maven + Gradle)
  └── ...
```

The `agent-base` image contains:
- A **harness entrypoint** (Go binary) that translates controller-set
  environment variables into runtime-specific configuration
- Core skills baked into `/opt/skills/`
- Konveyor tools (`fetch-analysis`, `run-analysis`)
- git, ssh, and basic utilities

The harness entrypoint:
1. Discovers skills from ImageVolume mounts and `/opt/skills/`
2. Reads `KONVEYOR_MODEL_*` env vars and maps to runtime-specific vars
   (e.g. `GOOSE_MODEL`, `OPENCODE_MODEL`)
3. Reads LLM credentials from env vars and configures the runtime
4. Configures memory service if `KONVEYOR_MEMORY_URL` is set
5. Handles session resumption if `KONVEYOR_SESSION_ID` is set
6. Composes the prompt from `KONVEYOR_AGENT_PROMPT` +
   `KONVEYOR_INSTRUCTIONS`
7. Invokes the detected agent runtime
8. On exit, writes session ID and results to `/.konveyor/`

The harness auto-detects which runtime is installed by checking PATH.
This means the controller never needs to know which runtime a container
carries -- the image is the runtime commitment.

The harness also serves as the entrypoint for LLMProvider verification.
When invoked with `KONVEYOR_VERIFY_PROVIDER=true`, it runs a lightweight
check (model list or trivial completion) and writes the result to
`/.konveyor/verification.json`.

Image composition (how language-specific images are built, which
language runtimes to include, devcontainer vs. buildpack vs. fat image
strategies) is out of scope for this enhancement.

### Security, Risks, and Mitigations

**Risk: Agent containers execute arbitrary code.**
- *Mitigation*: Agent Sandbox provides network isolation and security
  defaults. The agent runtime's permission model controls tool access.
  Agent containers run as non-root. RBAC controls who can create
  AgentRuns.

**Risk: LLM credentials exposed in environment variables.**
- *Mitigation*: Credentials are stored in Kubernetes Secrets and
  injected as env vars by the controller. The Sandbox's security
  context prevents credential exfiltration via process listing. RBAC
  restricts Secret access to the controller's service account.

**Risk: Skill OCI artifacts contain malicious content.**
- *Mitigation*: Skills are mounted as read-only ImageVolumes. They
  contain markdown instructions and metadata, not executable code.
  OCI artifacts support cosign signing and SLSA provenance for supply
  chain verification. The controller can be configured to require
  signed artifacts.

**Risk: In-cluster registry stores unvetted content.**
- *Mitigation*: The registry is internal (ClusterIP service, not
  exposed externally). Only the controller's service account has push
  access. Git sources and inline content are transformed by the
  controller, not by untrusted users directly.

**Risk: Agent Sandbox API changes.**
- *Mitigation*: Agent Sandbox is a Kubernetes SIG project with
  community governance. The single-container Sandbox primitive is
  simple enough that fallback to bare Pods is straightforward if
  needed. Pin to a specific API version and test against it in CI.

## Design Details

### MVP scope

The MVP includes five CRDs and four controllers:

| CRD | Controller | MVP |
|---|---|---|
| SkillCard | SkillCard controller | Yes |
| SkillCollection | SkillCollection controller | Yes |
| LLMProvider | LLMProvider controller | Yes |
| Agent | Agent controller | Yes |
| AgentRun | AgentRun controller | Yes |
| AgentPlaybook | Validating webhook only | CRD defined, no controller |
| AgentPlaybookRun | Validating webhook only | CRD defined, no controller |

AgentPlaybook and AgentPlaybookRun CRDs are included in the API definitions so
the full model is visible and reviewable, but their controllers are
deferred. The AgentPlaybookRun controller is the most complex component
(session PVC management, stage boundaries, phase sequencing, handoff
files) and is not required for single-agent execution.

### Post-MVP: AgentPlaybookRun controller

The AgentPlaybookRun controller creates AgentRun CRs sequentially per
phase. For each phase:

1. Determine the Agent and instructions from the AgentPlaybook.
2. Create an AgentRun with a shared workspace PVC.
3. Within a stage, pass the session ID from the previous phase's
   AgentRun to enable session resumption.
4. At stage boundaries, start a new session ID for fresh context.
5. Track per-phase status in AgentPlaybookRun status.

Session continuity relies on the workspace PVC containing the agent
runtime's session state (e.g. goose's SQLite database at
`GOOSE_PATH_ROOT`). The harness handles session discovery and
resumption.

Tekton Pipeline integration is a future option for users who want
pipeline-based orchestration with conditionals, parallelism, retries,
and supply chain features. The current design does not preclude this.

### Test Plan

**Unit tests**:
- SkillCard controller: test all three source paths (OCI, git, inline)
- SkillCollection controller: test child SkillCard creation and
  aggregate readiness
- Agent controller: test context budget computation, dependency
  resolution
- AgentRun controller: test Sandbox creation, env var injection,
  status tracking

**Integration tests** (envtest):
- Full lifecycle: create LLMProvider + SkillCards + Agent + AgentRun,
  verify Sandbox is created with correct configuration
- Skill resolution: create SkillCard with git source, verify OCI
  artifact is built and pushed
- Error handling: create AgentRun referencing non-existent Agent,
  verify error condition

**E2E tests** (real cluster):
- Deploy the controller with Agent Sandbox, create an AgentRun, verify
  the agent container starts and completes
- LLMProvider verification: create a provider with valid/invalid
  credentials, verify status reflects reality
- SkillCard inline: create a skill with inline content, verify it
  appears as an ImageVolume in the Sandbox

### Upgrade / Downgrade Strategy

**Initial deployment**: The controller is deployed alongside the
existing Konveyor operator. CRDs are registered at install time.
No migration from Hub-based agent resources is required for the MVP
-- the CRD-based system runs in parallel.

**CRD versioning**: All CRDs start at `v1alpha1`. Breaking changes
are expected during alpha. The conversion webhook pattern will be
used when moving to `v1beta1`.

**Controller upgrades**: Standard controller-runtime rolling update.
The controller is stateless -- all state lives in CRD status fields
and the Kubernetes API.

## Implementation History

- **2026-06**: POC CRD definitions and domain glossary on
  dymurray/tackle2-ui agent branch.
- **2026-06**: Investigation of Tekton Custom Task integration with
  Agent Sandbox (documented, deferred).
- **2026-06-17**: Enhancement proposal drafted.

## Drawbacks

1. **New repository and component**: Adds a Go controller to a project
   that currently runs Python (kai), Java (analyzer), and TypeScript
   (UI). Increases the technology surface and maintainer skill
   requirements.

2. **Agent Sandbox dependency**: The project depends on a SIG Apps
   project that is v1beta1. If the API changes significantly, the
   controller must adapt. Mitigation: the Sandbox primitive is simple
   and fallback to bare Pods is feasible.

3. **skillimage dependency**: skillimage is a young project (v0.7.2,
   single primary contributor). The controller imports its Go packages
   for OCI operations. Mitigation: Red Hat OCTO backing, and the
   project actively encourages community adoption. Konveyor would be a
   significant consumer driving stability.

4. **In-cluster registry requirement**: Git and inline source SkillCards
   require a registry to push built artifacts. This is an infrastructure
   dependency that vanilla Kubernetes clusters may not have. Mitigation:
   Helm subchart for Zot, OpenShift auto-detection, clear error
   conditions when no registry is configured.

5. **AgentPlaybook/AgentPlaybookRun complexity deferred**: The PM may want
   multi-phase execution sooner than post-MVP. The risk is that the
   MVP feels incomplete without orchestration. Mitigation: single-agent
   execution via AgentRun is immediately useful and covers the primary
   developer use case.

## Alternatives

### Hub as the controller

Extend Hub (Go, gin framework) with controllers for these resources.
Hub already manages tasks and addons.

**Rejected because**: ADR 0001 explicitly decouples agent execution
from Hub. Hub is a data service, not a workload launcher. Adding
controller-runtime to Hub's gin-based architecture mixes concerns
and couples the agent platform to Hub's release cycle.

### Tekton as the orchestration engine

Use Tekton Pipelines for multi-phase execution, with a Custom Task
controller bridging to Agent Sandbox.

**Deferred, not rejected**: Tekton provides ordering, parallelism,
retries, conditionals, timeouts, and supply chain signing for free.
However, the MVP requires only sequential execution, and adding a
Tekton dependency increases the install surface. The architecture
does not preclude Tekton integration later -- AgentPlaybookRun could
generate Tekton PipelineRuns as an implementation detail.

### Bare Pods instead of Agent Sandbox

Create Pods directly instead of depending on Agent Sandbox.

**Rejected because**: Agent Sandbox provides warm pools, hibernation,
stable identity, and network isolation. These are important for
running agent workloads at scale. Betting on the community project
and contributing upstream is a stronger position than building custom
pod management.

### SkillCards as pure OCI refs without CRDs

Reference skills only by OCI image ref in the Agent spec, no
cluster-resident SkillCard CRDs.

**Rejected because**: SkillCollections support git sources and inline
content that need to be resolved to OCI artifacts. CRDs make skills
visible in the cluster (`kubectl get skillcards`), enable RBAC, and
provide status reporting. The controller uses skillimage's Go library
to handle the OCI lifecycle.

### Single base image with multiple runtimes

Ship one base image containing both goose and opencode, select at
runtime.

**Rejected because**: Increases image size with unused binaries,
runtimes update on different cadences requiring monolithic rebuilds,
and session resumption within AgentPlaybook stages requires runtime
homogeneity anyway. One base image per runtime is cleaner.

## Infrastructure Needed

- **New repository**: `konveyor/agentic-controller` for the Go
  controller, CRD types, Helm chart, and CI configuration.

- **Container image builds**: CI pipeline to build and publish the
  controller image to `quay.io/konveyor/agentic-controller`.

- **Base image builds**: CI pipeline to build `agent-base`,
  `agent-base-goose`, `agent-base-opencode`, and language-specific
  images. (Detailed image composition is a separate enhancement but
  the base image with harness is needed for LLMProvider verification.)

- **OCI registry access**: Push access to `quay.io/konveyor/` for
  publishing controller and base images.

- **Test cluster**: CI environment with Agent Sandbox installed for
  E2E testing.
