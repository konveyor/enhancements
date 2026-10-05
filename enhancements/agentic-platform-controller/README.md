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
last-updated: 2026-07-23
status: provisional
see-also:
  - "/enhancements/kai/agent-driven-migration/README.md"
  - "/enhancements/agentic-platform-agent-images/README.md"
  - "https://github.com/redhat-et/skillimage"
  - "https://github.com/kubernetes-sigs/agent-sandbox"
  - "https://agentskills.io"
replaces: []
superseded-by: []
---

# Agentic Platform Controller

A Kubernetes-native controller for managing AI agent workloads. Introduces
seven CRDs under the `konveyor.io` API group and six controllers in a new
`konveyor/agentic-controller` repository.

Agents are templates that declare available skills, gateways (provider/model
combinations), a container image, a prompt, and typed parameters. AgentRuns
are invocations that supply concrete values — a gateway selection, parameter
values, instructions, and additional env/envFrom passed through to the Sandbox.
The controller validates, resolves skills to ImageVolumes, and creates
Agent Sandbox workloads. It does not interpret parameter values or call
any external API.

Agent workspaces are ephemeral — git is the persistence layer. The
harness (in the base image) clones a repo, the agent works, the harness
commits and pushes. Everything durable lives in git.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. **Model role names**: Resolved. Model roles follow a small known
   convention (`primary`, `efficient`, optional `planner`). Free-form
   roles are allowed but unmapped by the harness — passed through as
   env vars.

2. **SkillCard content hashing**: Resolved. Content-addressable tags
   by default. The controller hashes inline/git-sourced skill content
   and uses SHA-based OCI tags to avoid rebuilding unchanged skills.

3. **Gateway verification frequency**: Resolved. Verification on
   create/update only. Periodic health checks are deferred.

4. **Agent Sandbox API stability**: Resolved. Agent Sandbox is a hard
   dependency. Pin the version and maintain a CI contract test. Bare
   Pods are not a fallback.

5. **Incremental push frequency**: Resolved. The harness pushes (git
   push) after each successful commit batch. This is an event-based
   strategy. The push frequency detail is owned by the
   [agent-base-image-composition](/enhancements/agentic-platform-agent-images/README.md)
   enhancement.

## Summary

The Konveyor agentic platform defines Kubernetes CRDs for composing and
executing AI agent workloads. The design follows the Tekton Task/TaskRun
pattern: an Agent declares what is available and what inputs it needs;
an AgentRun supplies concrete values and triggers execution.

The controller is domain-agnostic. It does not call Hub, Backstage, or
any inventory system. Parameters are opaque key-value pairs — the
controller passes them through without interpretation. For
Konveyor-managed agents, Hub injects its connectivity info (`HUB_BASE_URL`,
`HUB_APP_ID`, scoped API token) into the AgentRun at create time, and
the harness resolves application metadata from Hub at runtime —
following the established addon pattern. For non-Konveyor use cases
(CLI, GitOps), the caller supplies all values directly. The controller
exposes interfaces and validation logic as importable Go packages so
Hub can reuse them.

Skills are sourced from OCI registries, git repositories, or inline
content and resolved to OCI artifacts by the SkillCard controller.
All skills are mounted at `/opt/skills/{name}/` via ImageVolumes —
baked-in and mounted skills coexist in the same directory.

Workspaces are ephemeral (EmptyDir). The harness clones, the agent
works, the harness commits and pushes. No PVCs survive between runs.

This enhancement covers the controller. Agent container image
composition is covered by the
[agent-base-image-composition](/enhancements/agentic-platform-agent-images/README.md)
enhancement. IDE-side integration is covered by the
[agent-driven-migration](/enhancements/kai/agent-driven-migration/README.md)
enhancement.

## Motivation

### Current State

The Konveyor POC for AI-powered migration (tackle2-addon-kai) uses Hub's
addon framework to launch agent containers. This couples agent execution
to Hub's task lifecycle and pod scheduling, requires Hub API changes for
every new resource type, and makes the agent platform dependent on Hub
for orchestration. The new architecture decouples orchestration
(controller manages CRDs and Sandbox pods) from data resolution
(harness calls Hub at runtime), while preserving the proven addon
runtime pattern where the thing-in-the-pod resolves what it needs from
Hub via a scoped API token.

The POC also uses pallet to sync skills from git repos at container
startup, adding network dependency and startup latency. Skills are not
versioned as artifacts, making reproducibility and supply chain security
difficult.

Workspaces in the addon model are EmptyDir volumes. The only durable
output is a git branch pushed before pod termination. Session context
(what the agent did, why, what to do next) is lost when the pod dies.

### Goals

1. **Kubernetes-native agent management**: CRDs, standard RBAC, GitOps,
   controller-runtime patterns.

2. **Domain-agnostic CRDs**: The controller does not encode assumptions
   about Hub, Backstage, git hosting, or any specific use case.
   Parameter values and env/envFrom are opaque to the controller.

3. **Tekton-style param model**: Agent declares typed params, AgentRun
   supplies values. The controller validates and passes through.

4. **Skill lifecycle management**: Resolve skills from OCI, git, or
   inline to versioned OCI artifacts via skillimage.

5. **Gateway verification**: Validate that LLM endpoints are
   reachable by launching the actual base image container.

6. **Git-as-persistence**: Ephemeral workspaces, durable git branches.
   No PVCs between runs.

### Non-Goals

1. **Agent container image composition**: Covered by the
   [agent-base-image-composition](/enhancements/agentic-platform-agent-images/README.md)
   enhancement.

2. **Tekton integration**: Deferred. The architecture does not preclude
   it.

3. **Skill authoring tools**: Domain of the skillimage project.

4. **Hub or inventory integration in the controller**: The controller
   does not call any external API. For Konveyor-managed agents, the
   harness resolves application metadata from Hub at runtime using a
   scoped API token — following the established addon pattern.

5. **Interactive / human-in-the-loop execution**: ACP-based interactive
   sessions are a future enhancement.

6. **Session handoff schema**: The schema for `.konveyor/session.json`
   and `.konveyor/handoff.md` is owned by the
   [agent-base-image-composition](/enhancements/agentic-platform-agent-images/README.md)
   enhancement.

## Proposal

### Architecture Overview

```text
                    ┌─────────────────────────────────────────────┐
                    │          Kubernetes API Server               │
                    │                                             │
                    │  SkillCard    SkillCollection   Gateway     │
                    │  Agent       AgentWorkflow                  │
                    │  AgentRun    AgentWorkflowRun               │
                    └──────────────┬──────────────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────────────┐
                    │      konveyor/agentic-controller            │
                    │                                             │
                    │  SkillCard controller    (OCI/git/inline)   │
                    │  SkillCollection controller (child CRs)     │
                    │  Gateway controller     (verification)      │
                    │  Agent controller        (budget, readiness)│
                    │  AgentRun controller     (validate, Sandbox)│
                    │  AgentWorkflowRun ctrl  (sequential runs)  │
                    └──────────────┬──────────────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────────────┐
                    │      Agent Sandbox (agents.x-k8s.io)       │
                    │                                             │
                    │  Sandbox with:                              │
                    │  - Agent container image                    │
                    │  - ImageVolumes at /opt/skills/{name}/      │
                    │  - KONVEYOR_PARAM_* env vars                │
                    │  - User-specified env / envFrom             │
                    │  - LLM credential Secrets                   │
                    │  - EmptyDir at /workspace                   │
                    └──────────────┬──────────────────────────────┘
                                   │
                            ┌──────▼──────┐
                            │  Git Remote │ ← branch with code
                            │             │   + .konveyor/ context
                            └─────────────┘
```

### Agent as Template, AgentRun as Invocation

The Agent declares what the execution needs. The AgentRun supplies
concrete values. The controller validates and passes them through.
This follows the Tekton Task/TaskRun pattern.

**Agent declares:**
- `params` — typed parameters (name, type, description, default)
- `gateways` — available gateways (provider/model combinations)
- `skillCards`, `skillCollections` — available skills
- `image` — container image with runtime and toolchains
- `prompt` — standing instructions

**AgentRun supplies:**
- `params` — values for each declared parameter
- `gateway` — gateway selection for this run
- `instructions` — task-specific instructions (composed with prompt)
- `env`, `envFrom` — additional env vars and secret refs passed
  through to the Sandbox as raw Kubernetes primitives

The controller injects params as `KONVEYOR_PARAM_{NAME}` env vars.
It does not interpret their values. For Konveyor-managed agents,
Hub adds `HUB_BASE_URL`, `HUB_APP_ID`, and a scoped API token to the
AgentRun's env/envFrom at create time, and the harness uses those
to call Hub at runtime for application metadata and credential
resolution. For non-Konveyor use cases, the caller supplies git
URLs, MCP server addresses, and credentials directly as params or
envFrom.

### Workspace Model: Git-as-Persistence

Workspaces are **ephemeral** (EmptyDir). All durable state lives in
**git**. No PVCs survive between runs.

The harness (entrypoint in the base image) holds git credentials —
the agent does not. The harness reads git coordinates from params
(`KONVEYOR_PARAM_SOURCE_URL`, `KONVEYOR_PARAM_TARGET_BRANCH`) and
handles the full git lifecycle:

1. Harness reads git credentials from mounted Secret
2. Clone the source repo into `/workspace/repo` (harness uses credentials)
3. Configure the workspace remote without credentials (agent cannot push)
4. Create or checkout the target branch
5. Agent works, harness commits incrementally
6. On exit: harness commits `.konveyor/handoff.md` and `.konveyor/session.json`
7. Harness pushes to the target repo (using credentials)
8. Harness writes `/.konveyor/results.json` (pod-local, read by controller)

The agent does not receive git credentials in its environment
variables or git configuration. The workspace remote is configured
without authentication — `git push` from the agent will fail. The
harness reads credentials from a mounted Secret and performs all
push operations on the agent's behalf. In the POC (Agent Sandbox,
single container), the credential exists on the container filesystem.
This reduces accidental exposure but does not provide full isolation.
Full credential isolation requires OpenShell filesystem policy
enforcement (future).

**Cross-stage continuity** in AgentWorkflows: all stages share the
same target branch. Each stage reads the previous stage's committed
`.konveyor/handoff.md`.

**Parallel agents**: different branches, same source. No contention.

**Session context on the branch** (`.konveyor/` directory):

| File | Purpose |
|---|---|
| `handoff.md` | Summary for next stage |
| `session.json` | Model, tokens, timing, tool calls |

### CRD Definitions

#### SkillCard

An individual agent skill or rule. Supports three source types:
OCI image ref, git source, inline markdown. All resolve to an OCI
artifact mounted at `/opt/skills/{name}/`.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: SkillCard
metadata:
  name: maven-migration
spec:
  # Exactly one of image, source, or inline
  image: quay.io/konveyor/skills/maven-migration:1.0.0

  displayName: Maven Migration
  version: "1.0.0"
  description: Migrates Maven POM files from Java EE to Jakarta EE.
  type: skill  # skill (default) or rule
  tags: [java, maven, migration]

status:
  resolvedImage: quay.io/konveyor/skills/maven-migration:1.0.0
  conditions:
    - type: Ready
      status: "True"
```

**Controller behavior**:

| Source | Action |
|---|---|
| `image` | Validate OCI ref exists. Set `status.resolvedImage`. |
| `source` | Clone via skillimage, build, push. Set `status.resolvedImage`. |
| `inline` | Build from content, push. Set `status.resolvedImage`. |

#### SkillCollection

A group of skills referenced by OCI image, git source, or SkillCard
CR name.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: SkillCollection
metadata:
  name: java-migration-skills
spec:
  version: "1.0.0"
  skills:
    - name: maven-migration
      skillCardRef: maven-migration
    - name: javax-imports
      image: quay.io/konveyor/skills/javax-imports:1.0.0
    - name: ejb-to-cdi
      source: https://github.com/konveyor/skills/tree/main/ejb-to-cdi

status:
  conditions:
    - type: Ready
      status: "True"
```

**Controller behavior**: Watch referenced SkillCards. Create child
SkillCard CRs for git sources. Report aggregate readiness.

#### Gateway

A specific provider/model combination with endpoint and credentials.
Each Gateway declares a provider type, an endpoint, credentials, and
one model. The provider type (e.g. `anthropic`, `openai`,
`gcp-vertex-ai`) is injected as `KONVEYOR_LLM_PROVIDER` so the
harness can map credentials to provider-specific env vars. This is
a pre-OpenShell shim — when `inference.local` eliminates
provider-specific credential mapping, the field is removed. To offer
multiple models, create multiple Gateway CRs.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: Gateway
metadata:
  name: anthropic-sonnet
spec:
  provider: anthropic
  endpoint: https://api.anthropic.com
  credentialRef:
    secretName: anthropic-credentials
    key: api-key
  model:
    name: claude-sonnet-4-20250514
    contextWindow: 200000
    tier: premium

status:
  connectionVerified: true
  conditions:
    - type: Ready
      status: "True"
```

**Controller behavior**: Launch a container using the agent base
image to verify connectivity. Update status.

#### Agent

A template declaring what is available and what inputs are needed.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: Agent
metadata:
  name: java-migration-agent
spec:
  image: quay.io/konveyor/agent-java-goose:latest
  prompt: |
    You are a Java migration specialist.

  gateways:
    - ref: anthropic-sonnet
    - ref: openai-gpt4o

  skillCards:
    - ref: maven-migration

  skillCollections:
    - ref: java-migration-skills

  params:
    - name: source_url
      type: string
      description: Git URL of the application source
    - name: source_branch
      type: string
      default: main
    - name: target_branch
      type: string
      description: Branch to push results to

status:
  conditions:
    - type: Ready
      status: "True"
```

**Controller behavior**: Watch dependencies. Validate referenced
resources exist. Report readiness. Skill-to-image compatibility
validation (e.g., a Maven skill on a Python image) is deferred.
Self-describing image labels with machine-readable capability
declarations (APB pattern) are the planned mechanism — see the
agent-base-image-composition enhancement.

#### AgentRun

An invocation that supplies concrete values and triggers execution.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: AgentRun
metadata:
  name: migrate-app-123
spec:
  agentRef: java-migration-agent

  gateway: anthropic-sonnet

  params:
    - name: source_url
      value: https://github.com/acme/legacy-app.git
    - name: source_branch
      value: main
    - name: target_branch
      value: konveyor/migrate-app-123

  instructions: |
    Migrate this application from Java EE 7 to Quarkus 3.x.

  # For Konveyor-managed agents, Hub injects these at create time:
  env:
    - name: HUB_BASE_URL
      value: https://hub.konveyor.svc
    - name: HUB_APP_ID
      value: "123"

  envFrom:
    - secretRef:
        name: agentrun-migrate-app-123-token  # Hub-minted scoped token

status:
  phase: Running
  startTime: "2026-06-22T10:35:00Z"
  conditions:
    - type: Succeeded
      status: "Unknown"
      reason: Running
```

**Controller behavior**:

1. Validate params match Agent's declarations
2. Resolve skills → OCI image refs (from SkillCard status)
3. Resolve gateway → credential Secret, set `KONVEYOR_LLM_PROVIDER`, `KONVEYOR_LLM_ENDPOINT`, `KONVEYOR_LLM_MODEL`, `KONVEYOR_LLM_API_KEY`
4. Inject params as `KONVEYOR_PARAM_*` env vars
5. Pass through `env` and `envFrom` unchanged
6. Create Sandbox (image + ImageVolumes for skills + env/envFrom + EmptyDir workspace). ImageVolumes are a Kubernetes PodSpec feature (K8s 1.33+) specified via `podTemplate.spec.volumes` on the Sandbox CR.
7. Watch Sandbox `Finished` condition
8. Read `/.konveyor/results.json`, update AgentRun status

The controller makes **no external API calls**. It reads CRs from
the Kubernetes API and creates Sandboxes. That's it.

#### AgentWorkflow

An ordered sequence of stages. Each stage references an Agent and
carries instructions.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: AgentWorkflow
metadata:
  name: java-migration
spec:
  guide: |
    Migrate a Java EE application to Quarkus.

  stages:
    - name: discover-and-plan
      agentRef: discovery-agent
      instructions: Analyze the app. Write MIGRATION_PLAN.md.
    - name: implement
      agentRef: migration-agent
      instructions: Read MIGRATION_PLAN.md. Execute migration.
    - name: review
      agentRef: review-agent
      instructions: Review all changes. Run build and tests.
```

#### AgentWorkflowRun

Execute a workflow. The controller creates AgentRuns sequentially,
all sharing the same target branch.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: AgentWorkflowRun
metadata:
  name: migrate-app-123-full
spec:
  workflowRef: java-migration

  gateway: anthropic-sonnet

  params:
    - name: target_branch
      value: konveyor/migrate-app-123-full

  # Hub injects HUB_BASE_URL, HUB_APP_ID, and scoped token at create time;
  # harness resolves git coordinates and credentials from Hub at runtime

status:
  currentStage: implement
  stages:
    - name: discover-and-plan
      phase: Succeeded
    - name: implement
      phase: Running
    - name: review
      phase: Pending
```

**Controller behavior**: Create AgentRuns per stage on the shared
branch. Each stage reads `.konveyor/handoff.md` from the previous.

### Skill Mounting

All skills live at `/opt/skills/{skillName}/SKILL.md`. Built-in
skills are baked into the base image. ImageVolume-mounted skills
are mounted into subdirectories alongside them:

```text
/opt/skills/
  konveyor-migration/SKILL.md   ← baked into image
  javax-rules/SKILL.md          ← ImageVolume from SkillCard
  custom-rules/SKILL.md         ← ImageVolume from SkillCard
```

`skillctl build` puts `SKILL.md` at the root of the OCI image.
Mount at `/opt/skills/{name}` → lands at the right path. No
symlinks, no fan-out. The agent runtime points at `/opt/skills/`
and discovers all skills regardless of source.

### Observability and Agent Interaction

For the POC, Goose is launched with `goose serve` which exposes
the full ACP (Agent Client Protocol) over HTTP (SSE) and WebSocket
on a single `/acp` endpoint. The Sandbox is created with
`service: true` for a stable DNS name.

The controller does not connect to agent pods via ACP. It stays a
standard stateless reconciler watching CRs:

| Concern | Owner | Mechanism |
|---|---|---|
| Create Sandbox, set initial status | Controller | Watch AgentRun, create Sandbox |
| Report pod phase (Running/Finished/Failed) | Controller | Watch Sandbox conditions |
| Set completionTime, duration | Controller | On Sandbox Finished condition |
| Git push result (branch, commit SHA) | Harness | Single AgentRun CR status update via SA token |

Real-time agent interaction (streaming output, tool calls,
permission requests, cancellation) is handled by the UI connecting
directly to the agent pod's ACP endpoint via Hub proxy. ACP
`session/load` replays full conversation history on connect, so
the UI can join a running session at any time without missing
messages. For unattended runs, `GOOSE_MODE=auto` disables
permission requests.

When adding agent runtimes that do not implement ACP-over-HTTP
(OpenCode, Claude, etc.), the harness adds a stdio-to-HTTP bridge.
The external interface is identical regardless of runtime.

See [ADR 0002: ACP Transport and Agent Observability](https://github.com/konveyor/agentic-controller/blob/main/docs/adr/0002-acp-transport-and-observability.md)
for the full design.

### User Stories

#### Story 1: Platform admin configures skills and gateways

```bash
kubectl apply -f skills/maven-migration.yaml
kubectl apply -f gateways/anthropic-sonnet.yaml

kubectl get skillcards
# NAME              TYPE    READY
# maven-migration   skill   True

kubectl get gateways
# NAME               MODEL                      VERIFIED
# anthropic-sonnet   claude-sonnet-4-20250514   True
```

#### Story 2: Developer runs a migration via the UI

1. User selects application from Hub inventory
2. User selects an Agent and gateway, provides instructions
3. UI sends create request to Hub (`POST /hub/agent/runs`) with
   agent ref, application ref, gateway, and instructions
4. Hub mints a scoped API token, adds `HUB_BASE_URL`, `HUB_APP_ID`,
   and the token to the AgentRun env/envFrom, creates the CR
5. Controller creates Sandbox pod
6. Harness calls Hub at runtime to resolve git URL, branch, and
   decrypted credentials from the application record
7. Harness clones repo, launches agent
8. UI polls AgentRun status; connects via ACP WebSocket for streaming
9. On completion: shows branch link

#### Story 3: CLI user runs an agent against any repo (future)

The CRDs support this — the controller passes params through
without interpretation. However, the current Konveyor harness
requires Hub for git credential resolution. This story requires
either a standalone harness or extending the current harness with
a non-Hub path.

```bash
kubectl apply -f - <<EOF
apiVersion: konveyor.io/v1alpha1
kind: AgentRun
metadata:
  name: review-infra-repo
spec:
  agentRef: code-review-agent
  gateway: anthropic-sonnet

  params:
    - name: source_url
      value: https://github.com/myorg/infra.git
    - name: target_branch
      value: konveyor/review-123
  instructions: Review Terraform modules for security issues.
  envFrom:
    - secretRef:
        name: git-creds
EOF
```

No Hub, no application inventory. Just a repo, an agent, and a
harness that supports this path.

### Implementation Details

#### Repository structure

```text
konveyor/agentic-controller/
  api/v1alpha1/
    skillcard_types.go
    skillcollection_types.go
    gateway_types.go
    agent_types.go
    agentworkflow_types.go
    agentrun_types.go
    agentworkflowrun_types.go
    groupversion_info.go
  internal/controller/
    skillcard_controller.go
    skillcollection_controller.go
    gateway_controller.go
    agent_controller.go
    agentrun_controller.go
    agentworkflowrun_controller.go
  internal/registry/
    client.go          # wraps skillimage pkg/oci
    detect.go          # auto-detect OpenShift registry
  config/crd/          # generated CRD manifests
  config/rbac/
  config/manager/
  charts/agentic-controller/
    Chart.yaml
    charts/zot/        # optional subchart
  cmd/manager/main.go
  Dockerfile
  go.mod
  Makefile
```

#### Key dependencies

| Dependency | Purpose |
|---|---|
| `sigs.k8s.io/controller-runtime` | Controller framework |
| `agents.x-k8s.io/v1beta1` | Agent Sandbox CRD types |
| `github.com/redhat-et/skillimage/pkg/oci` | OCI build/push/pull |
| `github.com/redhat-et/skillimage/pkg/source` | Git source resolution |
| `github.com/redhat-et/skillimage/pkg/skillcard` | SkillCard parsing |

#### In-cluster OCI registry

Skills built from git/inline need a registry to push to.

**OpenShift**: Auto-detect built-in image registry.
**Vanilla k8s**: Zot (optional Helm subchart).

Controller does not own registry lifecycle. Clear error condition
if git/inline SkillCards are created without a configured registry.

#### Harness and git workflow

The harness (`/usr/local/bin/konveyor-harness`) in the base image
handles git and runtime configuration. The controller sets env vars;
the harness reads them. See the
[agent-base-image-composition](/enhancements/agentic-platform-agent-images/README.md)
enhancement for details.

The harness reads `KONVEYOR_PARAM_*` env vars for git coordinates
and handles clone, branch, commit, push. It also reads LLM
credentials from env vars set via `envFrom` and configures the
agent runtime accordingly.

#### Hub integration

Hub's integration with the agent platform follows the established
addon pattern. The controller sits in the middle as a
domain-agnostic orchestrator. To the left, Hub and the UI handle
Konveyor-specific concerns (applications, credentials, migration
workflows). To the right, the harness in the base image handles
runtime resolution and git lifecycle. The controller does not know
about either side — it passes env vars and envFrom through without
interpretation.

```text
Hub / UI                   Controller               Harness / Runtime
(domain-specific)          (domain-agnostic)         (domain-specific)

Konveyor UI ──┐                                   ┌── Konveyor harness
              ├──► AgentRun CR ──► Sandbox ──►─────┤   (Hub client, git)
Hub (CRUD +   │                                    │
 token)───────┘                                    └── Custom harness
                                                       (other platforms)
CLI / GitOps ─────► AgentRun CR ──► Sandbox ──►──── Standalone harness
                                                    (no Hub needed)
```

Nothing stops another platform from providing its own UI, its own
harness, and using the same controller and CRDs for a completely
different use case.

##### Hub's responsibilities

**CRUD for agent CRDs.** Hub exposes REST endpoints for agent
resources under the `/hub/agent/` namespace using its
controller-runtime client — the same pattern as `AddonHandler`
and `ConfigMapHandler`. CRDs in etcd are the sole source of
truth; Hub does not store copies in its database. All reads are
request-driven (`client.List()`, `client.Get()`) with no informer
cache — the number of agent resources is small (tens, not
thousands).

| Hub Endpoint | CRD | Operations |
|---|---|---|
| `/hub/agent/agents` | Agent | List, Get, Create, Update, Delete |
| `/hub/agent/skills` | SkillCard | List, Get, Create, Update, Delete |
| `/hub/agent/skillcollections` | SkillCollection | List, Get, Create, Update, Delete |
| `/hub/agent/gateways` | Gateway | List, Get, Create, Update, Delete |
| `/hub/agent/runs` | AgentRun | List, Get, Create, Cancel |
| `/hub/agent/workflows` | AgentWorkflow | List, Get, Create, Update, Delete |
| `/hub/agent/workflowruns` | AgentWorkflowRun | List, Get, Create, Cancel |

When listing Agents and AgentWorkflows for the UI, Hub filters by
`konveyor.io/managed=true`. All other resource types (SkillCards,
SkillCollections, Gateways, AgentRuns, AgentWorkflowRuns) are
listed unfiltered. Resources without the managed label remain
usable via kubectl and other consumers.

**AgentRun creation.** When Hub receives a create request for an
AgentRun (or AgentWorkflowRun), it:

1. Mints a scoped API token with `AddonScopes` (including
   `applications:get`, `identities:decrypt`) — the same scopes
   addons receive. The token has a long TTL; the harness
   self-revokes on exit (see below).
2. Stores the token in a Kubernetes Secret
3. Adds `HUB_BASE_URL`, `HUB_APP_ID`, and the token Secret to the
   AgentRun's `spec.env` and `spec.envFrom`
4. Creates the AgentRun CR via `client.Create()`

Hub does **not** resolve application metadata, git URLs, branches,
or credentials at create time. The harness resolves what it needs
from Hub at runtime. Hub is fire-and-forget — it creates the CR
and moves on.

For AgentWorkflowRuns, the controller propagates env vars from the
AgentWorkflowRun to each child AgentRun. Every stage's harness
resolves the same application.

**Cancellation.** The UI cancels runs (never deletes them). Hub's
cancel handler:

1. Revokes the Hub API token (prevents further Hub API calls)
2. Sets `spec.cancel: true` on the AgentRun CR
3. The controller sees `spec.cancel`, deletes the Sandbox, and
   sets the phase to `Cancelled`

**Request-driven reads.** Hub does not watch or cache AgentRun
status. When the UI requests an AgentRun, Hub calls
`client.Get()` and returns the current state. All reads are
on-demand — consistent with the fire-and-forget create model.

**Runtime data service.** The harness calls Hub's existing REST
API at runtime using the scoped token to fetch application
metadata and decrypt credentials — the same endpoints addons use
today (`GET /applications/{id}`,
`GET /identities/{id}?decrypted=1`). No new Hub API capability
is required.

**ACP WebSocket proxy.** Hub proxies WebSocket connections from
the UI to agent pods for real-time ACP streaming:

1. UI polls `GET /hub/agent/runs/:name` until `phase: Running`
2. UI requests WebSocket upgrade via
   `GET /hub/agent/runs/:name/acp`
3. Hub resolves the Sandbox Service DNS from AgentRun status
4. Hub reads the ACP secret key from the Secret referenced by
   `status.secretKeyRef` and injects the `X-Secret-Key` header
5. Hub upgrades to WebSocket and proxies frames bidirectionally
6. UI receives ACP events (streaming text, tool calls, permission
   requests) and sends responses (approvals, cancellation)

The ACP secret key is generated by the controller (separate from
the Hub API token) and stored in a Kubernetes Secret. Hub reads
it to proxy; the UI never sees it directly.

##### Managed vs. standalone agents

The same agent image and harness support two modes. Platform
admins create separate Agent CRs for each mode.

**Konveyor-managed Agent** (`konveyor.io/managed=true`):

```yaml
apiVersion: konveyor.io/v1alpha1
kind: Agent
metadata:
  name: java-migration-agent
  labels:
    konveyor.io/managed: "true"
spec:
  image: quay.io/konveyor/agent-java-goose:latest
  prompt: "You are a Java migration specialist..."
  gateways:
    - ref: anthropic-sonnet
  skillCollections:
    - ref: java-migration-skills
  params:
    - name: target_framework
      type: string
      description: Target framework to migrate to
      default: quarkus-3
```

No `source_url`, `branch`, or `target_branch` params. Hub provides
the application ID via env var; the harness resolves git
coordinates and credentials from Hub at runtime. The UI only
renders form fields for params the user actually decides (here,
just `target_framework`).

**Standalone Agent** (future — not implemented in the current harness):

The controller and CRDs are domain-agnostic, so nothing prevents
a non-Hub agent. However, the current Konveyor harness
(`migration-harness`) is migration-specific and requires
`HUB_BASE_URL` + `APP_ID` — it resolves git coordinates and
credentials from Hub at runtime. A standalone path would require
either extending the Konveyor harness to support a non-Hub mode
or providing a different harness in the base image.

The CRD shape that would support a standalone agent:

```yaml
apiVersion: konveyor.io/v1alpha1
kind: Agent
metadata:
  name: java-migration-agent-standalone
spec:
  image: quay.io/example/agent-with-standalone-harness:latest
  prompt: "You are a Java migration specialist..."
  gateways:
    - ref: anthropic-sonnet
  skillCollections:
    - ref: java-migration-skills
  params:
    - name: source_url
      type: string
      required: true
    - name: branch
      type: string
      default: main
    - name: target_branch
      type: string
      required: true
    - name: target_framework
      type: string
      default: quarkus-3
```

Note: this uses a different image carrying a harness that reads
git coordinates from `KONVEYOR_PARAM_*` env vars and credentials
from mounted Secrets (via `envFrom`). The Konveyor harness does
not support this path today. The controller passes params through
without interpretation regardless of which harness the image
carries.

##### Harness behavior

The Konveyor harness (`migration-harness`) in the base image is
migration-specific — it acts as a Hub client, the same role that
the addon adapter (`shared/addon/adapter`) plays for addons today.
`HUB_BASE_URL` and `APP_ID` are required env vars; the harness
will not start without them.

**Current behavior** (Hub-only):
1. Reads `APP_ID` and `HUB_BASE_URL` from env
2. Calls `GET /applications/<id>` — gets git URL, branch
3. Calls `GET /identities/<id>?decrypted=1` — gets credentials
4. Clones the repo, configures workspace so the agent cannot push
   (credentials stay in the harness, not in env or git config)
5. Launches the agent runtime
6. On exit (success or failure): revokes the Hub API token, then
   exits

The harness commits work incrementally and pushes to the target
branch on exit.

**Future: standalone harness.** The controller is domain-agnostic
and passes params through without interpretation, so nothing
prevents a different harness that reads git coordinates from
`KONVEYOR_PARAM_*` env vars and credentials from mounted Secrets
instead of calling Hub. This would be either a separate binary in
a different base image, or an extension of the current harness
with a conditional path when `HUB_BASE_URL` is absent. This is
not in scope for the current implementation.

##### Token lifecycle

The Hub API token has three revocation paths:

- **Normal completion**: the harness revokes its own token as its
  last act before exiting (success or failure)
- **Cancellation**: Hub revokes the token in the cancel REST
  handler before setting `spec.cancel` on the CR
- **Crash / OOM-kill**: the harness cannot revoke. The token
  expires naturally via its TTL. Start with a long TTL (e.g. 24h);
  tighten later if needed.

##### Credential security

Git credentials never appear on the AgentRun CR. In managed mode,
the harness fetches them from Hub at runtime via the scoped token
and holds them internally — they are not exposed to the agent
process via env vars or git config. This is strictly better than
the earlier approach where credential Secret names were visible in
the CR's `envFrom`.

The Hub API token and the ACP secret key are separate credentials
with separate trust boundaries:

| | Hub API Token | ACP Secret Key |
|---|---|---|
| Created by | Hub | Controller |
| Purpose | Harness → Hub API calls | UI → Hub → pod ACP proxy |
| Stored in | Secret on CR's envFrom | Secret via status.secretKeyRef |
| Revoked | By harness on exit, by Hub on cancel | N/A — destroyed with pod |

When OpenShell is layered on, its network policy can restrict Hub
access to the harness binary only — the agent process would be
denied.

##### Networking

Sandbox pods in managed mode need egress to Hub's in-cluster
Service DNS. Two layers provide this:

- **Agent Sandbox**: The `SandboxTemplate` `networkPolicy` field
  allowlists egress to Hub's namespace and port. The default
  managed policy blocks RFC1918 egress, so this is required.
- **OpenShell** (if deployed): A declarative policy entry
  allowlists `hub.konveyor.svc.cluster.local` for the harness
  binary. Exact hostnames resolve to private IPs without needing
  `allowed_ips` overrides.

Standalone mode does not require Hub connectivity.

##### UI interaction

The UI renders the AgentRun create form dynamically from the
Agent's param declarations. For managed agents, the form shows
only user-decided params (e.g. `target_framework`) plus an
application picker and model selector. The UI does not resolve
git URLs or credentials — Hub and the harness handle that.

##### Run pruning

The controller prunes completed AgentRuns and AgentWorkflowRuns
based on configurable TTLs per terminal condition, following
the same pattern as Kubernetes Jobs:

- `ttlSecondsAfterSucceeded` — TTL for runs that succeeded
- `ttlSecondsAfterFailed` — TTL for runs that failed
- `ttlSecondsAfterCancelled` — TTL for runs that were cancelled

Per-condition TTLs allow different retention policies — for
example, keeping failed runs longer for debugging while
aggressively pruning successful runs. When the controller deletes
an AgentRun, owner references cascade-delete the Sandbox and
associated Secrets (ACP key, etc.) automatically.

##### Scaling protection

To prevent cluster resource exhaustion from unbounded concurrent
runs:

- **Kubernetes `ResourceQuota`**: Standard per-namespace resource
  limits. This is the primary hard protection — prevents any
  namespace from consuming unbounded compute.
- **Controller `maxConcurrentRuns`**: A configurable limit on
  concurrent AgentRuns. Runs beyond the limit remain in `Pending`
  phase until a slot opens. This provides a better UX than
  ResourceQuota rejection — runs queue rather than fail.

##### Controller impact

Minimal. The controller's existing `spec.env` and `spec.envFrom`
passthrough already handles Hub connectivity injection. Whatever
Hub puts on the CR flows through to the Sandbox container
unchanged. The controller does not know about Hub or applications.

New controller concerns:
- `spec.cancel` field: delete the Sandbox, set phase to `Cancelled`
- Per-condition TTL pruning: prune completed runs after TTL
- `maxConcurrentRuns`: hold excess runs in `Pending`

##### Reusable interfaces

The controller exposes its orchestration logic as importable Go
packages — not just CRD types, but the interfaces and functions
for param validation, model selection, skill resolution, and
Sandbox construction. Hub imports these alongside the CRD types.

This enables a future where the Hub tasking system could run
agents and agent workflows directly — using the controller's Go
packages for orchestration without creating CRs. In that model,
Hub would use the same interfaces to validate, resolve, and build
Sandboxes, tightly controlling the dependency on the agentic
controller as a library rather than a running controller.

### Security, Risks, and Mitigations

**Risk: Agent containers execute arbitrary code.**
- *Mitigation*: Agent Sandbox provides network isolation. Non-root
  execution. RBAC controls AgentRun creation. OpenShell
  (https://github.com/NVIDIA/OpenShell) is NVIDIA's secure-by-design
  runtime for autonomous agents, providing kernel-level isolation,
  declarative YAML-based security policies, and inference routing.
  OpenShell can layer on top for L7 egress policy (future).

**Risk: Credentials in the Sandbox.**
- *Mitigation*: Git push credentials are not exposed to the agent
  via env vars or git config (see credential isolation above).
  LLM credentials are mounted via `envFrom` for POC and destroyed
  with the pod. RBAC restricts Secret access. The controller's
  service account does not need access to the credential Secrets —
  the Sandbox spec references them and kubelet mounts them.

**Risk: Skill OCI artifacts contain malicious content.**
- *Mitigation*: Mounted as read-only ImageVolumes. Contain markdown,
  not executables. Support cosign signing.

**Risk: Git remote unavailable during push.**
- *Mitigation*: Harness pushes incrementally during execution, not
  just on exit. Partial work is preserved on the remote.

**Risk: Agent Sandbox API changes.**
- *Mitigation*: Hard dependency — pinned version with CI contract
  test. SIG project with active development.

**Risk: Credentials visible to agent process.**
- *Mitigation*: Git push credentials are not provided to the agent
  via environment variables or git configuration — the harness
  reads them from a mounted Secret and pushes on the agent's
  behalf. In the POC (single-container Sandbox), the credential
  exists on the container filesystem and a determined agent could
  locate it. This reduces accidental exposure (logging, prompt
  injection exfiltration) but does not provide full isolation.
  Full credential isolation requires OpenShell
  (https://github.com/NVIDIA/OpenShell) filesystem policy
  enforcement, which can deny reads to the Secret mount path at
  the kernel level. LLM API keys are injected via env vars for POC
  (standard for all agent deployments today). OpenShell inference
  routing is the target architecture for LLM credential isolation
  — the agent talks to a local proxy, OpenShell injects the real
  key. OpenShell integration may be accelerated if credential
  exposure is a blocker for public release.

**Risk: Sensitive data in pod logs.**
- *Mitigation*: RBAC restricts who can read pod logs. Credentials
  are destroyed with the pod. OpenShell L7 egress policy (future)
  prevents credential exfiltration to unauthorized endpoints.

## Design Details

### Phased Rollout

#### POC: Single AgentRun

Five CRDs, five controllers. SkillCards OCI refs only.

| CRD | Controller | Scope |
|---|---|---|
| SkillCard | SkillCard controller | POC |
| SkillCollection | SkillCollection controller | POC |
| Gateway | Gateway controller | POC |
| Agent | Agent controller | POC |
| AgentRun | AgentRun controller | POC |

**UI access:** Via Hub's CRUD endpoints under `/hub/agent/`. Hub
mints a scoped API token at AgentRun create time, injects Hub
connectivity info (`HUB_BASE_URL`, `HUB_APP_ID`, token), and creates
the CR. The harness resolves application metadata from Hub at
runtime — following the established addon pattern.

#### Phase 2: AgentWorkflow (flat stages)

| CRD | Controller | Scope |
|---|---|---|
| AgentWorkflow | AgentWorkflow controller | Phase 2 |
| AgentWorkflowRun | AgentWorkflowRun controller | Phase 2 |

All stages share a target branch. Cross-stage handoff via
committed `.konveyor/handoff.md`.

#### Phase 3: SkillCard sources

**SkillCard resolution:** Git source and inline content resolution.
In-cluster registry integration (Zot or OpenShift built-in).

#### Phase 4: Agent memory

Agent optionally references a persistent memory service (MCP).
Knowledge accumulates across runs. POC exists in
dymurray/tackle2-addon-kai.

#### ACP interactive execution (architecturally enabled in POC)

The POC uses `goose serve` to expose ACP over HTTP/WebSocket
from each agent pod. This architecturally enables human-in-the-loop
interaction from the IDE or web UI — the UI connects to the pod's
ACP endpoint via Hub proxy. When adding non-Goose runtimes, the
harness bridges stdio ACP to the same HTTP endpoint. See
[ADR 0002](https://github.com/konveyor/agentic-controller/blob/main/docs/adr/0002-acp-transport-and-observability.md).

### Test Plan

**Unit tests**:
- SkillCard controller: all three source paths
- SkillCollection controller: child creation, readiness
- Agent controller: param validation, context budget
- AgentRun controller: param validation, Sandbox creation, env
  injection, status tracking
- Gateway controller: valid endpoint verification, invalid credentials handling
- AgentWorkflowRun controller: sequential stages, shared branch

**Integration tests** (envtest):
- Full lifecycle: Gateway + SkillCards + Agent + AgentRun
- Param validation: missing required param, wrong type
- Error handling: non-existent Agent, unresolved skills

**E2E tests** (real cluster):
- Deploy controller with Agent Sandbox, run an AgentRun
- Gateway verification: valid/invalid credentials
- AgentWorkflowRun: three stages, verify branch has all commits

### Upgrade / Downgrade Strategy

**CRD versioning**: All CRDs start at `v1alpha1`. Breaking changes
expected. Conversion webhooks for `v1beta1`.

**Controller**: Stateless rolling update. All state in CRD status.

## Implementation History

- **2026-06**: POC CRD definitions on dymurray/tackle2-ui agent branch
- **2026-06-17**: Enhancement proposal drafted
- **2026-06-18**: Revised: git-as-persistence, simplified workflow
- **2026-06-22**: Revised: Tekton-style param model, controller
  decoupled from Hub, env/envFrom passthrough, skill mounting
  simplified to single /opt/skills/ directory
- **2026-07-23**: Revised: Hub integration follows established addon
  pattern — Hub injects HUB_BASE_URL, HUB_APP_ID, and scoped token
  directly on the CR (no Task envelope). Harness resolves from Hub
  at runtime. Added cancel semantics, token lifecycle, run pruning,
  scaling protection, reusable controller interfaces.
- **2026-08-04**: Renamed LLMProvider CRD to Gateway. Each Gateway
  represents one provider/model combination (single `model:` field,
  not a `models:` list). Agent `providers` field renamed to
  `gateways`. AgentRun/AgentWorkflowRun `models` list replaced with
  single `gateway` string field. Added `spec.provider` field to
  Gateway — a pre-OpenShell shim carrying the runtime provider type
  (e.g. `anthropic`, `openai`) injected as `KONVEYOR_LLM_PROVIDER`
  so the harness can map credentials to provider-specific env vars.
  Deletes cleanly when OpenShell `inference.local` eliminates
  provider-specific credential mapping.

## Drawbacks

1. **New Go component**: Adds controller-runtime to the project's
   technology surface.

2. **Agent Sandbox dependency**: v1beta1 SIG project. Hard dependency — pinned version with CI contract test.

3. **skillimage dependency**: Pre-1.0 Go library. Pin version, test
   in CI. Scope is limited to four packages; vendoring is feasible.

4. **In-cluster registry** (Phase 3): Required for git/inline
   SkillCards. Zot subchart or OpenShift auto-detect.

5. **Git remote as sole persistence**: Work in progress lost on pod
   crash if harness hasn't pushed. Mitigated by incremental push.

6. **Harness requires Hub connectivity**: For Konveyor-managed
   agents, Sandbox pods need egress to Hub's in-cluster Service
   DNS. This requires network policy configuration on the
   SandboxTemplate. Non-Konveyor use cases (where the caller
   supplies all values directly) do not need Hub connectivity.

## Landscape

### Lightspeed Agentic Operator (openshift/lightspeed-agentic-operator)

The closest project architecturally. A Go controller-runtime operator
that drives cluster remediation workflows through LLM agents in
Agent Sandbox pods. Key observations:

**Overlap:** Uses Agent Sandbox, OCI image volumes for skills, LLM
provider management with multi-tier agents, approval gates, MCP
server integration. Their `Agent` and `LLMProvider` CRDs solve
similar problems to ours.

**Differences:** Domain-locked to cluster ops via a fixed Proposal
workflow (analyze → execute → verify → escalate). No generic
execution model. No skill resolution from multiple sources. No
git workspace lifecycle. No param model.

**Opportunity:** Their `LLMProvider` CRD satisfies our requirements
and could be adopted directly. Their `Agent` would need to be
enriched (container image, system prompt, skills) to serve as a
shared primitive. See the
[Shared primitives with lightspeed-agentic-operator](#shared-primitives-with-lightspeed-agentic-operator)
alternative for a detailed gap analysis and path forward.

### Kagenti (kagenti/kagenti)

Platform middleware for deploying, securing, and governing agents.
IBM + Red Hat joint project. Provides AuthBridge (zero-trust
sidecars, SPIFFE identity), MCP Gateway (Envoy-based), Istio
ambient mesh integration, NetworkPolicy automation.

**Overlap:** Security and networking infrastructure for agent
workloads. Their OpenShell integration provides agent sandboxing.

**Differences:** Not an execution orchestrator. No skill packaging,
no LLM management, no git lifecycle. Requires A2A protocol on
every agent. Heavy infrastructure footprint (Istio, SPIRE,
Keycloak).

**Opportunity:** AuthBridge and MCP Gateway could be adopted
independently for security hardening (our Phase 4+) without
coupling to the full platform. Ladislav Smola (Red Hat) is a
maintainer.

### Fullsend (fullsend-ai/fullsend)

Autonomous SDLC platform — agents triage issues, implement code,
review PRs, and merge. Red Hat (Konflux CI team). Runs on GitHub
Actions with OpenShell containers.

**Overlap:** Validates the git-as-persistence model in production
(ephemeral sandboxes, all output as git branches). Their harness
model (three-tier config inheritance) is well-designed.

**Differences:** Runs on GitHub Actions, not Kubernetes. Fully
autonomous (no human-in-the-loop during execution). Explicitly
rejected Agent Sandbox and ACP for their use case.

**Opportunity:** If we build k8s-native agent execution well,
fullsend could be a consumer when they add Kubernetes support
(listed on their roadmap as "Later").

### Positioning

All three projects solve adjacent problems:

- **Kagenti**: security/networking layer (infrastructure)
- **Lightspeed Agentic**: cluster ops remediation (domain-specific)
- **Fullsend**: autonomous SDLC (domain-specific, not k8s-native)
- **Konveyor**: generic agent execution primitives (domain-agnostic)

We build the generic layer. Agent Sandbox and OpenShell are shared
infrastructure at the bottom. Our CRDs (Agent, AgentRun with params,
SkillCard with OCI resolution) are the reusable primitives above
the sandbox layer that none of the other projects provide in a
domain-agnostic way.

The differentiator nobody else is building: ACP-native interactive
agents in Kubernetes — fleet-managed agents that feel like local
agents from the IDE or terminal (Future phase).

## Alternatives

### Shared primitives with lightspeed-agentic-operator

The [lightspeed-agentic-operator](https://github.com/openshift/lightspeed-agentic-operator)
is the closest project to what we are building. Both projects use
Agent Sandbox for execution, OCI image volumes for skills, and
Kubernetes CRDs for LLM provider management. We evaluated whether
we could adopt their `agentic.openshift.io/v1alpha1` primitives
(`Agent`, `LLMProvider`, `Proposal`) instead of defining our own,
and identified one area of strong alignment and four requirements
that their current model does not satisfy.

**What aligns: LLMProvider / Gateway.** Their `LLMProvider` CRD is a
discriminated union across five backends (Anthropic, GoogleCloudVertex,
OpenAI, AzureOpenAI, AWSBedrock) with per-backend config structs,
CEL validation ensuring exactly one is set, and credential management
via Secret references. This is richer than what we have designed and
satisfies our requirements. We have renamed our equivalent to Gateway
(one provider/model combination per CR) to align with the OpenShell
gateway model, but the underlying concern is the same.

**Gap 1: Agent as a reusable capability template.** We need the Agent
to define the full execution environment — a container image (runtime
and toolchains), a `systemPrompt` (standing behavioral instructions),
and skills (OCI artifacts with domain knowledge). This makes Agent a
composable template: the same container image can back multiple Agents
with different personalities and skill sets, and many runs can
reference the same Agent without re-specifying the environment. Their
`Agent` CRD (`agent_types.go`) is an LLM tier configuration —
`spec.llmProvider` (single ref), `spec.model`, `spec.timeouts`,
`spec.maxTurns` — with no container image, no prompt, and no skills.
The execution environment is instead assembled by the Proposal
controller from a fixed base `SandboxTemplate` and per-Proposal
`ToolsSpec` fields.

**Gap 2: Multiple skills images per agent.** We mount multiple OCI
skill artifacts as separate ImageVolumes under a shared directory
(`/opt/skills/{name}/`). Being able to define an Agent with the
specific skills it should use — each sourced from its own OCI
image — is what makes the Agent a meaningful unit of composition.
Their `ToolsSpec.skills` field (`shared_types.go`) is a list of
`SkillsSource` entries (OCI image ref + paths), but the Proposal
controller's `patchSkillsImage()` function (`sandbox_templates.go`)
patches a single image volume on the base `SandboxTemplate`, taking
`skills[0]` — the first entry. Supporting multiple independently-
sourced skill images would require changes to how they build derived
`SandboxTemplate` resources.

**Gap 3: Skills belong on the agent definition, not the workflow
resource.** Skills define what an agent *can do* — they are part of
its identity and are reused across runs. In their model, skills are
specified on `Proposal.spec.tools` (or per-step overrides), meaning
every Proposal must re-specify the full `ToolsSpec`. Moving skills
to the Agent would enable reuse: a cluster admin defines an Agent
once with its skills, and users reference it by name in Proposals
without knowing the underlying skill composition.

**Gap 4: A generic execution primitive without the Proposal
workflow.** We need a way to say "run this Agent with these
instructions, to completion" — a single-shot execution primitive
analogous to a Tekton TaskRun. Their only execution path is the
`Proposal` CRD, which enforces a fixed state machine
(Pending → Analyzing → Proposed → Executing → Verifying → Completed,
with approval gates, escalation, and retry logic). There is no way
to execute an agent without entering this workflow. A lower-level
execution primitive (similar to our AgentRun) that Proposal could
be built on top of — creating one per step — would give both
projects a shared foundation while preserving their domain-specific
workflow as a higher-order resource.

**Path forward.** If the lightspeed-agentic-operator project is
interested in generalizing these primitives — enriching Agent to
carry image, prompt, and skills; supporting multiple skill images;
and introducing a generic execution resource below Proposal — we
would collaborate on shared CRD definitions and controllers. Their
`LLMProvider` is a strong starting point for the Gateway concern
(we have renamed ours to Gateway to align with the OpenShell
gateway model). Until such collaboration materializes, we define
our own primitives under `konveyor.io` to avoid blocking on
external alignment.

### Hub adapter in the controller

The controller resolves application metadata from Hub via an
internal adapter. The AgentRun carries an application ID.

**Rejected**: Couples the controller to Hub's API. The controller
stays domain-agnostic — it passes params through without
interpretation. Application metadata resolution belongs in the
harness (at runtime, following the addon pattern), not in the
controller.

### Smart Hub endpoint for AgentRun creation

Hub eagerly resolves application metadata (git URLs, credentials)
at AgentRun create time, building the full CR before creating it.

**Rejected**: Breaks the established Hub addon pattern where Hub
is a dumb store and the thing-in-the-pod resolves what it needs
at runtime via a scoped API token. The harness calling Hub at
runtime (like addons do today) is consistent with existing Hub
conventions and requires no new Hub resolution logic.

### Task envelope for application binding

Hub creates a Task row in its database to carry the application
reference, following the addon task model. The harness discovers
the application by fetching the Task record from Hub.

**Rejected**: The Task concept represents an addon execution
request — it has an addon image, a pod, and a lifecycle that Hub
manages. Agent runs have no addon and the controller (not Hub)
creates the pod. Expanding the Task definition for this purpose
is impractical. Hub injects the application ID directly on the
CR via env vars instead.

### Custom `source` field on AgentRun

The AgentRun has a first-class `source` field with git URL,
credential ref, and branch — not a declared param.

**Rejected**: This makes the controller aware of git, which is a
domain concept. Params are more generic — the same mechanism works
for git URLs, Hub tokens, MCP server addresses, or any future
input the agent needs.

### Custom `services` abstraction

The AgentRun has a `services` field listing external services with
URLs and credentials.

**Rejected**: Kubernetes already has env vars and Secrets. The
`env`/`envFrom` fields are the standard way to inject service
coordinates and credentials into pods. A custom abstraction adds
no value.

### PVC workspace persistence

PVCs survive between runs instead of git-as-persistence.

**Rejected**: PVC lifecycle management per application adds
operational complexity. Git is the durable state. Validated by
fullsend (Konflux) and cloud agent products.

### Tekton as orchestration engine

**Deferred**: MVP needs only sequential stages. Architecture does
not preclude Tekton integration later.

### Bare Pods instead of Agent Sandbox

**Rejected**: Agent Sandbox is a hard dependency. It provides warm pools, network isolation, and lifecycle management. Pinned version with CI contract test.

### SkillCards without CRDs

**Rejected**: Git/inline sources need resolution to OCI artifacts.
CRDs provide visibility, RBAC, status.

## Infrastructure Needed

- **Platform requirements**: Kubernetes 1.33+ (ImageVolume GA),
  OpenShift 4.20+, Agent Sandbox v0.5.x (pinned)
- **New repository**: `konveyor/agentic-controller`
- **Container image builds**: CI for controller + base images
- **OCI registry access**: Push to `quay.io/konveyor/`
- **Test cluster**: Agent Sandbox + ImageVolumes enabled
