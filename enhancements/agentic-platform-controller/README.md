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
last-updated: 2026-06-22
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

Agents are templates that declare available skills, LLM providers, a
container image, a prompt, and typed parameters. AgentRuns are invocations
that supply concrete values — model selections, parameter values,
instructions, and additional env/envFrom passed through to the Sandbox.
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

1. **Model role names**: The AgentRun selects models with roles (e.g.
   `primary`, `efficient`). Should recognized roles be fixed by
   convention or free-form?

2. **SkillCard content hashing**: When a SkillCard uses inline content,
   should the controller use content-addressable tags to avoid
   rebuilding unchanged skills?

3. **LLMProvider verification frequency**: Should verification happen
   only on create/update, or periodically?

4. **Agent Sandbox API stability**: Agent Sandbox is v1beta1. What is
   the migration cost if the API changes before our v1?

5. **Incremental push frequency**: How often should the harness push
   during long-running agent sessions?

## Summary

The Konveyor agentic platform defines Kubernetes CRDs for composing and
executing AI agent workloads. The design follows the Tekton Task/TaskRun
pattern: an Agent declares what is available and what inputs it needs;
an AgentRun supplies concrete values and triggers execution.

The controller is domain-agnostic. It does not call Hub, Backstage, or
any inventory system. The creator of the AgentRun (UI, CLI, Backstage
plugin, CI pipeline) resolves application metadata and supplies it as
parameter values and env/envFrom on the AgentRun. The controller passes
these through without interpretation.

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
to Hub's task lifecycle, requires Hub API changes for every new resource
type, and makes the agent platform dependent on Hub.

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

5. **LLM provider verification**: Validate that LLM endpoints are
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
   does not call any external API. The UI resolves application metadata
   before creating the AgentRun.

5. **Interactive / human-in-the-loop execution**: ACP-based interactive
   sessions are a future enhancement.

## Proposal

### Architecture Overview

```text
                    ┌─────────────────────────────────────────────┐
                    │          Kubernetes API Server               │
                    │                                             │
                    │  SkillCard    SkillCollection   LLMProvider │
                    │  Agent       AgentPlaybook                  │
                    │  AgentRun    AgentPlaybookRun               │
                    └──────────────┬──────────────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────────────┐
                    │      konveyor/agentic-controller            │
                    │                                             │
                    │  SkillCard controller    (OCI/git/inline)   │
                    │  SkillCollection controller (child CRs)     │
                    │  LLMProvider controller  (verification)     │
                    │  Agent controller        (budget, readiness)│
                    │  AgentRun controller     (validate, Sandbox)│
                    │  AgentPlaybookRun ctrl   (sequential runs)  │
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
- `providers` — available LLM providers and models
- `skillCards`, `skillCollections` — available skills
- `image` — container image with runtime and toolchains
- `prompt` — standing instructions

**AgentRun supplies:**
- `params` — values for each declared parameter
- `models` — specific provider/model selections
- `instructions` — task-specific instructions (composed with prompt)
- `env`, `envFrom` — additional env vars and secret refs passed
  through to the Sandbox as raw Kubernetes primitives

The controller injects params as `KONVEYOR_PARAM_{NAME}` env vars.
It does not interpret their values. Git URLs, Hub tokens, MCP
server addresses — all just parameter values to the controller.

### Workspace Model: Git-as-Persistence

Workspaces are **ephemeral** (EmptyDir). All durable state lives in
**git**. No PVCs survive between runs.

The harness (entrypoint in the base image) reads git coordinates
from params (`KONVEYOR_PARAM_SOURCE_URL`, `KONVEYOR_PARAM_TARGET_BRANCH`)
and handles the full git lifecycle:

1. Clone the source repo into `/workspace/repo`
2. Create or checkout the target branch
3. Agent works, harness commits incrementally
4. On exit: commit `.konveyor/handoff.md` and `.konveyor/session.json`
5. Push to the target repo
6. Write `/.konveyor/results.json` (pod-local, read by controller)

**Cross-stage continuity** in AgentPlaybooks: all stages share the
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

#### LLMProvider

An LLM service endpoint with credentials and available models.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: LLMProvider
metadata:
  name: anthropic-provider
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
  discoveredModels: [claude-sonnet-4-20250514, claude-haiku]
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

  providers:
    - ref: anthropic-provider
    - ref: openai-provider

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
  contextBudget:
    rulesContentTokens: 1500
    smallestContextWindow: 200000
    utilization: "0.75%"
  conditions:
    - type: Ready
      status: "True"
```

**Controller behavior**: Watch dependencies. Compute context budget.
Report readiness.

#### AgentRun

An invocation that supplies concrete values and triggers execution.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: AgentRun
metadata:
  name: migrate-app-123
spec:
  agentRef: java-migration-agent

  models:
    - role: primary
      provider: anthropic-provider
      model: claude-sonnet-4-20250514

  params:
    - name: source_url
      value: https://github.com/acme/legacy-app.git
    - name: source_branch
      value: main
    - name: target_branch
      value: konveyor/migrate-app-123

  instructions: |
    Migrate this application from Java EE 7 to Quarkus 3.x.

  env:
    - name: HUB_BASE_URL
      value: https://hub.konveyor.svc
    - name: APP_ID
      value: "123"

  envFrom:
    - secretRef:
        name: hub-agent-token
    - secretRef:
        name: git-write-creds

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
3. Resolve LLM providers → credential Secrets
4. Inject params as `KONVEYOR_PARAM_*` env vars
5. Pass through `env` and `envFrom` unchanged
6. Create Agent Sandbox (image + ImageVolumes + env + EmptyDir)
7. Watch Sandbox `Finished` condition
8. Read `/.konveyor/results.json`, update AgentRun status

The controller makes **no external API calls**. It reads CRs from
the Kubernetes API and creates Sandboxes. That's it.

#### AgentPlaybook

An ordered sequence of stages. Each stage references an Agent and
carries instructions.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: AgentPlaybook
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

#### AgentPlaybookRun

Execute a playbook. The controller creates AgentRuns sequentially,
all sharing the same target branch.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: AgentPlaybookRun
metadata:
  name: migrate-app-123-full
spec:
  playbookRef: java-migration

  models:
    - role: primary
      provider: anthropic-provider
      model: claude-sonnet-4-20250514

  params:
    - name: source_url
      value: https://github.com/acme/legacy-app.git
    - name: target_branch
      value: konveyor/migrate-app-123-full

  envFrom:
    - secretRef:
        name: hub-agent-token
    - secretRef:
        name: git-write-creds

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

### User Stories

#### Story 1: Platform admin configures skills and providers

```bash
kubectl apply -f skills/maven-migration.yaml
kubectl apply -f providers/anthropic.yaml

kubectl get skillcards
# NAME              TYPE    READY
# maven-migration   skill   True

kubectl get llmproviders
# NAME                 VERIFIED   MODELS
# anthropic-provider   True       claude-sonnet-4-20250514, claude-haiku
```

#### Story 2: Developer runs a migration via the UI

1. User selects application from Hub inventory
2. UI resolves git URLs + credential Secret names from Hub
3. UI creates AgentRun CR via Hub's passthrough proxy to the k8s API
4. UI watches AgentRun status via the same passthrough
5. On completion: shows branch link

#### Story 3: CLI user runs an agent against any repo

```bash
kubectl apply -f - <<EOF
apiVersion: konveyor.io/v1alpha1
kind: AgentRun
metadata:
  name: review-infra-repo
spec:
  agentRef: code-review-agent
  models:
    - role: primary
      provider: anthropic-provider
      model: claude-sonnet-4-20250514
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

No Hub, no application inventory. Just a repo and an agent.

### Implementation Details

#### Repository structure

```text
konveyor/agentic-controller/
  api/v1alpha1/
    skillcard_types.go
    skillcollection_types.go
    llmprovider_types.go
    agent_types.go
    agentplaybook_types.go
    agentrun_types.go
    agentplaybookrun_types.go
    groupversion_info.go
  internal/controller/
    skillcard_controller.go
    skillcollection_controller.go
    llmprovider_controller.go
    agent_controller.go
    agentrun_controller.go
    agentplaybookrun_controller.go
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

#### UI access via Hub passthrough

The UI accesses CRDs through Hub's existing service passthrough
pattern (`ServiceHandler.Forward`). Hub proxies requests to the
Kubernetes API server using its service account token and provides
RBAC scope checking. The UI does not need a separate `/k8s` proxy
route — all traffic flows through Hub.

This is the POC approach. The long-term API surface (whether Hub
should expose curated REST endpoints for agent resources instead
of raw k8s API passthrough) is left for future investigation.
Trade-offs to evaluate include: k8s API conventions vs Hub
conventions, privilege escalation via Hub's SA, and cross-plane
data joining (agent resources in etcd, application data in Hub's
DB).

### Security, Risks, and Mitigations

**Risk: Agent containers execute arbitrary code.**
- *Mitigation*: Agent Sandbox provides network isolation. Non-root
  execution. RBAC controls AgentRun creation. OpenShell can layer
  on top for L7 egress policy (future).

**Risk: Credentials in the Sandbox.**
- *Mitigation*: LLM and git credentials are Kubernetes Secrets
  mounted via `envFrom`. Destroyed with the pod. RBAC restricts
  Secret access. The controller's service account does not need
  access to the credential Secrets — the Sandbox spec references
  them and kubelet mounts them.

**Risk: Skill OCI artifacts contain malicious content.**
- *Mitigation*: Mounted as read-only ImageVolumes. Contain markdown,
  not executables. Support cosign signing.

**Risk: Git remote unavailable during push.**
- *Mitigation*: Harness pushes incrementally during execution, not
  just on exit. Partial work is preserved on the remote.

**Risk: Agent Sandbox API changes.**
- *Mitigation*: SIG project. Sandbox primitive is simple; fallback
  to bare Pods is feasible.

## Design Details

### Phased Rollout

#### POC: Single AgentRun

Five CRDs, five controllers. SkillCards OCI refs only.

| CRD | Controller | Scope |
|---|---|---|
| SkillCard | SkillCard controller | POC |
| SkillCollection | SkillCollection controller | POC |
| LLMProvider | LLMProvider controller | POC |
| Agent | Agent controller | POC |
| AgentRun | AgentRun controller | POC |

The UI resolves application metadata from Hub and creates the
AgentRun with all values filled in. The controller passes them
through.

#### Phase 2: AgentPlaybook (flat stages)

| CRD | Controller | Scope |
|---|---|---|
| AgentPlaybook | AgentPlaybook controller | Phase 2 |
| AgentPlaybookRun | AgentPlaybookRun controller | Phase 2 |

All stages share a target branch. Cross-stage handoff via
committed `.konveyor/handoff.md`.

#### Phase 3: Configurable git strategy + skill sources

SkillCard git source and inline content resolution. In-cluster
registry integration. Configurable git push strategy
(direct/fork/mirror) driven by the UI or CLI creating the
AgentRun with appropriate params.

#### Phase 4: Agent memory

Agent optionally references a persistent memory service (MCP).
Knowledge accumulates across runs. POC exists in
dymurray/tackle2-addon-kai.

#### Future: ACP interactive execution

Agent exposes ACP over WebSocket. Human-in-the-loop from IDE
or terminal. Nobody else in open k8s is building this.

### Test Plan

**Unit tests**:
- SkillCard controller: all three source paths
- SkillCollection controller: child creation, readiness
- Agent controller: param validation, context budget
- AgentRun controller: param validation, Sandbox creation, env
  injection, status tracking
- AgentPlaybookRun controller: sequential stages, shared branch

**Integration tests** (envtest):
- Full lifecycle: LLMProvider + SkillCards + Agent + AgentRun
- Param validation: missing required param, wrong type
- Error handling: non-existent Agent, unresolved skills

**E2E tests** (real cluster):
- Deploy controller with Agent Sandbox, run an AgentRun
- LLMProvider verification: valid/invalid credentials
- AgentPlaybookRun: three stages, verify branch has all commits

### Upgrade / Downgrade Strategy

**CRD versioning**: All CRDs start at `v1alpha1`. Breaking changes
expected. Conversion webhooks for `v1beta1`.

**Controller**: Stateless rolling update. All state in CRD status.

## Implementation History

- **2026-06**: POC CRD definitions on dymurray/tackle2-ui agent branch
- **2026-06-17**: Enhancement proposal drafted
- **2026-06-18**: Revised: git-as-persistence, simplified playbook
- **2026-06-22**: Revised: Tekton-style param model, controller
  decoupled from Hub, env/envFrom passthrough, skill mounting
  simplified to single /opt/skills/ directory

## Drawbacks

1. **New Go component**: Adds controller-runtime to the project's
   technology surface.

2. **Agent Sandbox dependency**: v1beta1 SIG project. Fallback to
   bare Pods is feasible.

3. **skillimage dependency**: Pre-1.0 Go library. Pin version, test
   in CI. Scope is limited to four packages; vendoring is feasible.

4. **In-cluster registry** (Phase 3): Required for git/inline
   SkillCards. Zot subchart or OpenShift auto-detect.

5. **Git remote as sole persistence**: Work in progress lost on pod
   crash if harness hasn't pushed. Mitigated by incremental push.

6. **UI does more work**: The UI resolves application metadata from
   Hub before creating the AgentRun. The controller doesn't help
   with this. Trade-off: controller stays domain-agnostic.

## Alternatives

### Hub adapter in the controller

The controller resolves application metadata from Hub via an
internal adapter. The AgentRun carries an application ID.

**Rejected**: Couples the controller to Hub's API. The Tekton model
is better — the AgentRun carries resolved values, the controller
passes them through. The UI resolves from Hub, Backstage, or
wherever before creating the AgentRun.

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

**Rejected**: Agent Sandbox provides warm pools, network isolation.
Betting on the SIG project is stronger than building pod management.

### SkillCards without CRDs

**Rejected**: Git/inline sources need resolution to OCI artifacts.
CRDs provide visibility, RBAC, status.

## Infrastructure Needed

- **New repository**: `konveyor/agentic-controller`
- **Container image builds**: CI for controller + base images
- **OCI registry access**: Push to `quay.io/konveyor/`
- **Test cluster**: Agent Sandbox + ImageVolumes enabled
