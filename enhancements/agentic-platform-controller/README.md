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
last-updated: 2026-06-18
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
`konveyor/agentic-controller` repository. Agents are composed from portable
skill packages, LLM provider configurations, and container images, then
executed as Agent Sandbox workloads. The controller handles skill resolution
(OCI, git, inline markdown), LLM provider verification, context budget
computation, and execution lifecycle management.

Agent workspaces are ephemeral — git is the persistence layer. The agent
clones a repo, works on a branch, commits results (code and session
context), pushes, and the pod dies. Everything durable lives in git.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. **Model role names**: The AgentRun selects models with roles (e.g.
   `primary`, `efficient`). The base image harness maps roles to
   runtime-specific env vars. Should the set of recognized roles be
   fixed by convention or free-form?

2. **SkillCard content hashing**: When a SkillCard uses inline content,
   should the controller use content-addressable tags (hash of content)
   to avoid rebuilding unchanged skills?

3. **LLMProvider verification frequency**: Should the LLMProvider
   controller verify connectivity only on create/update, or
   periodically?

4. **Agent Sandbox API stability**: Agent Sandbox is v1beta1. What is
   the migration cost if the API changes before our v1?

5. **Incremental push frequency**: How often should the harness push
   during long-running agent sessions? Time-based (every N minutes),
   commit-count-based, or configurable?

## Summary

The Konveyor agentic platform defines Kubernetes CRDs for composing and
executing AI agent workloads. The platform separates capability
definition (what skills, providers, and models are available) from
execution (specific selections for a particular run). Skills are sourced
from OCI registries, git repositories, or inline content and resolved to
OCI artifacts. Agent workloads run in Agent Sandbox pods with skills
mounted as ImageVolumes.

Workspaces are ephemeral. The agent clones the application's git repo
into an EmptyDir, creates a branch, does its work, commits code changes
and session context (`.konveyor/` files) to the branch, and pushes.
When the pod dies, the EmptyDir is gone — all durable state is in git.
This eliminates PVC lifecycle management between runs and enables
parallel agents on the same application via separate branches.

This enhancement covers the controller that reconciles these CRDs. Agent
container image composition is covered by the
[agent-base-image-composition](/enhancements/agentic-platform-agent-images/README.md)
enhancement. IDE-side agent integration is covered by the
[agent-driven-migration](/enhancements/kai/agent-driven-migration/README.md)
enhancement.

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

Workspaces in the addon model are EmptyDir volumes — entirely ephemeral.
The only durable output is a git branch pushed by the addon before pod
termination. Session context (what the agent did, why, what to do next)
is lost when the pod dies.

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

4. **Git-as-persistence**: Agent workspaces are ephemeral (EmptyDir).
   All durable state — code changes, session context, handoff notes —
   is committed to a git branch. No PVCs survive between runs. Cross-
   stage continuity in playbooks is through branch content.

5. **Runtime-agnostic CRDs**: The CRDs do not encode assumptions about
   specific agent runtimes (goose, opencode). Runtime selection is
   determined by the container image. The base image harness translates
   generic controller inputs to runtime-specific configuration.

6. **Portable execution**: Agent workloads run in Agent Sandbox pods,
   decoupled from Hub. Hub remains a runtime data service (analysis
   results, app metadata) but does not launch or manage agent workloads.

### Non-Goals

1. **Agent container image composition**: How language-specific agent
   images are built is covered by the
   [agent-base-image-composition](/enhancements/agentic-platform-agent-images/README.md)
   enhancement.

2. **Tekton integration**: Tekton Pipelines integration for multi-stage
   orchestration is future work.

3. **Skill authoring tools**: How users author, test, and publish skills
   is the domain of the skillimage project.

4. **Hub API changes**: No new Hub API endpoints, database tables, or Go
   models are required. The UI talks directly to the Kubernetes API for
   CRD operations via a proxy route.

5. **Interactive / human-in-the-loop execution**: ACP-based interactive
   agent sessions are a future enhancement. This enhancement covers
   batch execution (run to completion).

## Proposal

### Architecture Overview

```
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
                    │  ┌──────────────┐  ┌────────────────────┐  │
                    │  │ SkillCard    │  │ SkillCollection    │  │
                    │  │ Controller   │  │ Controller         │  │
                    │  │ OCI/git/     │  │ Creates child      │  │
                    │  │ inline →     │  │ SkillCard CRs,     │  │
                    │  │ resolved ref │  │ reports readiness   │  │
                    │  └──────────────┘  └────────────────────┘  │
                    │                                             │
                    │  ┌──────────────┐  ┌────────────────────┐  │
                    │  │ Agent        │  │ LLMProvider        │  │
                    │  │ Controller   │  │ Controller         │  │
                    │  │ Context      │  │ Launches base      │  │
                    │  │ budget,      │  │ image to verify    │  │
                    │  │ readiness    │  │ connectivity       │  │
                    │  └──────────────┘  └────────────────────┘  │
                    │                                             │
                    │  ┌──────────────┐  ┌────────────────────┐  │
                    │  │ AgentRun     │  │ AgentPlaybookRun   │  │
                    │  │ Controller   │  │ Controller         │  │
                    │  │ Creates      │  │ Creates AgentRuns  │  │
                    │  │ Sandbox,     │  │ sequentially per   │  │
                    │  │ tracks       │  │ stage, same branch │  │
                    │  │ completion   │  │                    │  │
                    │  └──────────────┘  └────────────────────┘  │
                    └──────────────┬──────────────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────────────┐
                    │      Agent Sandbox (agents.x-k8s.io)       │
                    │                                             │
                    │  Sandbox with:                              │
                    │  - Agent container image (runtime + tools)  │
                    │  - ImageVolumes (skills as OCI artifacts)   │
                    │  - Secret env vars (LLM credentials)       │
                    │  - KONVEYOR_* env vars (source, branch,    │
                    │    instructions, model roles)               │
                    │  - EmptyDir at /workspace                   │
                    └─────────────────────────────────────────────┘
                                   │
                            ┌──────▼──────┐
                            │  Git Remote │  ← branch with code
                            │             │    changes + .konveyor/
                            └─────────────┘    session context
```

### Workspace Model: Git-as-Persistence

Agent workspaces are **ephemeral**. The workspace is an EmptyDir that
dies with the pod. All durable state lives in git:

1. The harness clones the application's git repo into `/workspace/repo`
2. Creates (or checks out) a target branch
3. The agent works — harness commits incrementally with structured
   trailers
4. On exit, harness commits `.konveyor/` context files to the branch
5. Pushes the branch to the remote
6. Writes `/.konveyor/results.json` (pod-local, read by controller)
7. Pod terminates. EmptyDir is destroyed. Work is in git.

**Cross-stage continuity** in AgentPlaybooks: all stages share the same
target branch. Each stage checks out the branch, reads the previous
stage's `.konveyor/handoff.md`, does its work, commits, pushes. The
branch content IS the handoff mechanism.

**Parallel agents**: Multiple AgentRuns against the same application
use different branches (`konveyor/migrate-app-123-attempt-1`,
`konveyor/migrate-app-123-attempt-2`). No shared state, no contention.

**Session context on the branch** (`.konveyor/` directory):

| File | Purpose | Written by |
|---|---|---|
| `handoff.md` | Human-readable summary for the next stage | Harness on exit |
| `session.json` | Session metadata (model, tokens, timing, tool calls) | Harness on exit |
| `results.json` | Structured output for traceability | Harness on exit |

These files travel with the branch. The next stage's agent reads them
naturally when it checks out the branch.

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
    ref: main

  # -- OR --
  inline:
    content: |
      You must never use javax.* imports.
      Always replace with jakarta.* equivalents.

  displayName: Maven Migration
  version: "1.0.0"
  description: Migrates Maven POM files from Java EE to Jakarta EE.
  type: skill  # skill (default) or rule
  tags: [java, maven, migration]
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
| `image` | Validate OCI ref exists. Set `status.resolvedImage`. |
| `source` | Clone via skillimage `pkg/source.Resolve()`, build via `pkg/oci.Client.Build()`, push to registry. Set `status.resolvedImage`. |
| `inline` | Create temp skill directory with content as `SKILL.md`, build and push. Set `status.resolvedImage`. |

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
      skillCardRef: maven-migration
    - name: javax-imports
      image: quay.io/konveyor/skills/javax-imports:1.0.0
    - source: https://github.com/konveyor/skills/tree/main/ejb-to-cdi

status:
  resolvedSkills:
    - name: maven-migration
      resolvedImage: quay.io/konveyor/skills/maven-migration:1.0.0
      ready: true
  conditions:
    - type: Ready
      status: "True"
```

**Controller behavior**: For `skillCardRef` entries, watch the referenced
SkillCard's status. For `image` entries, validate the OCI ref. For
`source` entries, create a child SkillCard CR. Report aggregate
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
using the agent base image with provider credentials and endpoint
injected. The container runs a verification entrypoint that tests the
real network path and credential chain. Updates status with
`connectionVerified` and `discoveredModels`.

#### Agent

A capability definition declaring what is available for execution.
An Agent does not select a specific model — it declares what is
available. Model selection happens at execution time in the AgentRun.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: Agent
metadata:
  name: java-migration-agent
  namespace: konveyor
spec:
  image: quay.io/konveyor/agent-java-goose:latest
  prompt: |
    You are a Java migration specialist.

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

**Controller behavior**: Watch referenced SkillCards, SkillCollections,
LLMProviders. Compute context budget (always-loaded rule content vs.
smallest context window). Report readiness.

#### AgentRun

A request to execute a single Agent with specific selections against
an application's git repository.

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

  source:
    url: https://github.com/acme/legacy-app.git
    credentialRef: git-creds
    baseBranch: main

  instructions: |
    Migrate this application from Java EE 7 to Quarkus 3.x.

  parameters:
    APP_ID: "123"
    HUB_BASE_URL: "https://hub.konveyor.svc"

status:
  sandboxName: migrate-app-123-sandbox
  branch: konveyor/migrate-app-123
  commitSHA: abc123def456
  phase: Running
  startTime: "2026-06-17T10:35:00Z"
  conditions:
    - type: Succeeded
      status: "Unknown"
      reason: Running
```

**Controller behavior**:

1. Validate selected providers/models are in the Agent's available set
2. Resolve all skills from the Agent to OCI image refs
3. Determine target branch: `konveyor/<agentrun-name>` (or use
   `spec.source.targetBranch` if set)
4. Create an Agent Sandbox with:
   - Agent's container image
   - ImageVolumes for each resolved skill
   - Secret env vars for LLM credentials
   - `KONVEYOR_SOURCE_URL`, `KONVEYOR_SOURCE_CREDENTIAL`,
     `KONVEYOR_TARGET_BRANCH`, `KONVEYOR_BASE_BRANCH`
   - `KONVEYOR_MODEL_PRIMARY`, `KONVEYOR_MODEL_EFFICIENT`
   - `KONVEYOR_AGENT_PROMPT`, `KONVEYOR_INSTRUCTIONS`
   - EmptyDir at `/workspace`
5. Watch Sandbox `Finished` condition
6. Read `/.konveyor/results.json` from the pod
7. Update AgentRun status with branch, commitSHA, summary

#### AgentPlaybook

A reusable playbook with an ordered sequence of stages. Each stage
references an Agent and carries instructions. Stages execute
sequentially, each in its own Sandbox with a fresh agent session.
Cross-stage continuity is through the shared git branch.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: AgentPlaybook
metadata:
  name: java-migration
  namespace: konveyor
spec:
  guide: |
    Migrate a Java EE application to Quarkus.

  stages:
    - name: discover-and-plan
      agentRef: discovery-agent
      instructions: |
        Analyze the application. Write MIGRATION_PLAN.md.
        Write .konveyor/handoff.md summarizing findings.

    - name: implement
      agentRef: migration-agent
      instructions: |
        Read MIGRATION_PLAN.md and .konveyor/handoff.md.
        Execute the migration. Commit changes incrementally.
        Update .konveyor/handoff.md with what you did.

    - name: review
      agentRef: review-agent
      instructions: |
        Review all changes on this branch.
        Run the build and tests. Write REVIEW.md.
```

**Controller behavior**: Validate all referenced Agents exist and are
ready. Report readiness.

#### AgentPlaybookRun

A request to execute an AgentPlaybook against an application. The
controller creates AgentRuns sequentially, one per stage, all pushing
to the same target branch.

```yaml
apiVersion: konveyor.io/v1alpha1
kind: AgentPlaybookRun
metadata:
  name: migrate-app-123-full
  namespace: konveyor
spec:
  playbookRef: java-migration

  models:
    - role: primary
      provider: anthropic-provider
      model: claude-sonnet-4-20250514

  source:
    url: https://github.com/acme/legacy-app.git
    credentialRef: git-creds
    baseBranch: main

  parameters:
    APP_ID: "123"
    HUB_BASE_URL: "https://hub.konveyor.svc"

status:
  branch: konveyor/migrate-app-123-full
  currentStage: implement
  stages:
    - name: discover-and-plan
      agentRunRef: migrate-app-123-full-discover
      phase: Succeeded
      commitSHA: abc123
    - name: implement
      agentRunRef: migrate-app-123-full-implement
      phase: Running
    - name: review
      phase: Pending
  conditions:
    - type: Succeeded
      status: "Unknown"
      reason: StageRunning
```

**Controller behavior**:

1. Determine target branch: `konveyor/<playbookrun-name>`
2. Write the playbook's `guide` to the branch (via stage 1's harness
   as `.konveyor/guide.md`)
3. For each stage sequentially:
   a. Create an AgentRun with the stage's Agent and instructions
   b. Set `targetBranch` to the shared branch
   c. Wait for the AgentRun to succeed
   d. Update per-stage status
4. On final stage completion, update AgentPlaybookRun status

Each stage's agent sees the previous stage's work on the branch:
committed code changes, `.konveyor/handoff.md`, `.konveyor/session.json`.
Fresh agent session per stage — no session PVC, no session resumption.

### User Stories

#### Story 1: Platform admin configures skills and providers

```bash
kubectl apply -f skills/maven-migration.yaml      # OCI ref
kubectl apply -f skills/community-rules.yaml       # git source
kubectl apply -f skills/no-javax-rule.yaml         # inline content
kubectl apply -f collections/java-migration.yaml
kubectl apply -f providers/anthropic.yaml

kubectl get skillcards
# NAME              TYPE    SOURCE   READY
# maven-migration   skill   oci      True
# community-rules   skill   git      True
# no-javax-rule     rule    inline   True

kubectl get llmproviders
# NAME                 VERIFIED   MODELS
# anthropic-provider   True       claude-sonnet-4-20250514, claude-haiku
```

#### Story 2: Developer runs a single-agent migration

```bash
kubectl apply -f runs/migrate-app-123.yaml
kubectl get agentruns
# NAME              AGENT                   BRANCH                         PHASE
# migrate-app-123   java-migration-agent    konveyor/migrate-app-123       Running
```

After completion, the developer reviews the branch:
```bash
git fetch origin
git diff main..konveyor/migrate-app-123
# Code changes + .konveyor/session.json + .konveyor/handoff.md
```

#### Story 3: Architect runs a multi-stage playbook

```bash
kubectl apply -f runs/migrate-app-123-full.yaml
kubectl get agentplaybookruns
# NAME                    PLAYBOOK          STAGE              BRANCH
# migrate-app-123-full    java-migration    implement          konveyor/migrate-app-123-full
```

Three stages execute sequentially on the same branch. Each stage reads
the previous stage's handoff, does its work, commits, pushes.

### Implementation Details

#### Repository structure

```
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
  config/
    crd/               # generated CRD manifests
    rbac/
    manager/
  charts/agentic-controller/
    Chart.yaml
    charts/zot/        # optional subchart for dev
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
| `github.com/redhat-et/skillimage/pkg/oci` | OCI image build/push/pull |
| `github.com/redhat-et/skillimage/pkg/source` | Git source resolution |
| `github.com/redhat-et/skillimage/pkg/skillcard` | SkillCard parsing |
| `github.com/redhat-et/skillimage/pkg/collection` | SkillCollection parsing |

#### In-cluster OCI registry

The controller pushes built skill artifacts (from git sources and inline
content) to an OCI registry. The registry URL is configured via flag,
ConfigMap, or environment variable.

**OpenShift**: Auto-detect the built-in image registry at
`image-registry.openshift-image-registry.svc:5000`.

**Vanilla Kubernetes**: Helm chart includes Zot (project-zot/zot-minimal)
as an optional subchart. The konveyor operator deploys Zot for
production.

The controller does not own registry lifecycle. If git or inline source
SkillCards are created without a configured registry, the controller
sets a clear error condition on the CR.

#### Harness and git workflow

The harness entrypoint (`/usr/local/bin/konveyor-harness`) in the base
image handles the git workspace lifecycle. The controller sets env vars;
the harness does the git work. See the
[agent-base-image-composition](/enhancements/agentic-platform-agent-images/README.md)
enhancement for harness details.

The harness git workflow:

1. **Clone**: `git clone $KONVEYOR_SOURCE_URL /workspace/repo`
2. **Branch**: If `KONVEYOR_TARGET_BRANCH` exists on remote, check it
   out. Otherwise create it from `KONVEYOR_BASE_BRANCH`.
3. **Credentials**: Write git credentials from
   `KONVEYOR_SOURCE_CREDENTIAL` to `$HOME/.git-credentials`.
4. **Agent execution**: Discover skills, configure runtime, compose
   prompt, invoke agent with `/workspace/repo` as working directory.
5. **Incremental commits**: Commit during execution with structured
   trailers (`Konveyor-AgentRun`, `Konveyor-Agent`, `Konveyor-Model`).
6. **Context commit**: On exit, commit `.konveyor/handoff.md` and
   `.konveyor/session.json` to the branch.
7. **Push**: `git push origin $KONVEYOR_TARGET_BRANCH`
8. **Results**: Write `/.konveyor/results.json` (pod-local) with branch,
   SHA, files modified, duration.

The harness pushes incrementally during long-running sessions to protect
against pod crashes.

#### Env var contract

The controller sets these env vars on every Sandbox:

| Env var | Purpose |
|---|---|
| `KONVEYOR_SOURCE_URL` | Git repo URL to clone |
| `KONVEYOR_SOURCE_CREDENTIAL` | Path to credential file for git auth |
| `KONVEYOR_TARGET_BRANCH` | Branch name to create/checkout |
| `KONVEYOR_BASE_BRANCH` | Branch to base work on (default: repo default) |
| `KONVEYOR_AGENT_PROMPT` | Agent's standing prompt |
| `KONVEYOR_INSTRUCTIONS` | Run-specific instructions |
| `KONVEYOR_MODEL_PRIMARY` | Primary model (provider/model) |
| `KONVEYOR_MODEL_EFFICIENT` | Efficient model (optional) |
| `KONVEYOR_MEMORY_URL` | Memory service endpoint (optional) |
| `KONVEYOR_VERIFY_PROVIDER` | When `true`, run LLM verification mode |

Hub-specific parameters (`APP_ID`, `HUB_BASE_URL`, etc.) are passed as
generic key-value pairs from `spec.parameters`.

### Security, Risks, and Mitigations

**Risk: Agent containers execute arbitrary code.**
- *Mitigation*: Agent Sandbox provides network isolation. Non-root
  execution. RBAC controls who can create AgentRuns. OpenShell
  (NVIDIA) can be layered on top for L7 egress policy, Landlock
  filesystem isolation, and credential boundary enforcement.

**Risk: LLM credentials exposed in environment variables.**
- *Mitigation*: Stored in Kubernetes Secrets, injected by controller.
  RBAC restricts Secret access to controller's service account.

**Risk: Git credentials in the Sandbox.**
- *Mitigation*: Credentials are scoped (read/write to the specific
  repo). The harness writes them to `$HOME/.git-credentials` inside
  the pod's EmptyDir, destroyed with the pod. Agent Sandbox network
  isolation limits where the credentials can be used.

**Risk: Agent pushes to wrong branch or repo.**
- *Mitigation*: The harness only pushes to `KONVEYOR_TARGET_BRANCH` on
  `KONVEYOR_SOURCE_URL`. The controller generates the branch name. Git
  credentials can be scoped to specific repos via deploy keys or app
  tokens.

**Risk: Skill OCI artifacts contain malicious content.**
- *Mitigation*: Mounted as read-only ImageVolumes. Contain markdown,
  not executables. Support cosign signing for supply chain verification.

**Risk: Agent Sandbox API changes.**
- *Mitigation*: SIG project with community governance. Sandbox primitive
  is simple; fallback to bare Pods is feasible. Pin to specific API
  version.

## Design Details

### Phased Rollout

#### POC: Single AgentRun

Five CRDs, five controllers. Single-agent execution end to end.

| CRD | Controller | Scope |
|---|---|---|
| SkillCard | SkillCard controller | POC |
| SkillCollection | SkillCollection controller | POC |
| LLMProvider | LLMProvider controller | POC |
| Agent | Agent controller | POC |
| AgentRun | AgentRun controller | POC |

The agent clones the app's repo, works on a branch, commits results
and session context, pushes. Git strategy: direct push to the source
repo (option A).

#### Phase 2: AgentPlaybook (flat stages)

Two more CRDs, one more controller.

| CRD | Controller | Scope |
|---|---|---|
| AgentPlaybook | AgentPlaybook controller | Phase 2 |
| AgentPlaybookRun | AgentPlaybookRun controller | Phase 2 |

An AgentPlaybook is a flat sequence of stages — no phases within
stages. Each stage is an AgentRun with a fresh session. All stages
share the same target branch. Cross-stage continuity is through
committed files (`.konveyor/handoff.md`, code changes).

#### Phase 3: Configurable git strategy

Add configurability for how changes are committed back. The git
strategy is configured per-application or per-run:

| Strategy | Behavior | Use case |
|---|---|---|
| `direct` | Push branch to app's own repo | POC, simple setups |
| `fork` | Controller creates a fork, pushes there, creates PR | Production, keeps user repo clean |
| `mirror` | Controller uses in-cluster bare repo | Air-gapped environments |

The harness behavior changes per strategy. The controller handles fork
creation (forge API) or mirror maintenance based on configuration.

### Test Plan

**Unit tests**:
- SkillCard controller: all three source paths (OCI, git, inline)
- SkillCollection controller: child SkillCard creation, readiness
- Agent controller: context budget computation, dependency resolution
- AgentRun controller: Sandbox creation, env var injection, status
- AgentPlaybookRun controller: sequential stage execution, branch
  sharing

**Integration tests** (envtest):
- Full lifecycle: LLMProvider + SkillCards + Agent + AgentRun
- Skill resolution: git source → OCI artifact
- AgentPlaybookRun: three stages sequentially, same branch
- Error handling: non-existent Agent, unresolved skills

**E2E tests** (real cluster):
- Deploy controller with Agent Sandbox, run an AgentRun, verify
  branch exists on remote with expected commits
- LLMProvider verification: valid/invalid credentials
- AgentPlaybookRun: three-stage playbook, verify branch has commits
  from all three stages with `.konveyor/` context files

### Upgrade / Downgrade Strategy

**Initial deployment**: Deployed alongside existing Konveyor operator.
CRDs registered at install time. No migration from Hub-based agent
resources — CRD system runs in parallel.

**CRD versioning**: All CRDs start at `v1alpha1`. Breaking changes
expected during alpha. Conversion webhooks for `v1beta1` transition.

**Controller upgrades**: Standard controller-runtime rolling update.
Stateless — all state in CRD status and Kubernetes API.

## Implementation History

- **2026-06**: POC CRD definitions and domain glossary on
  dymurray/tackle2-ui agent branch.
- **2026-06**: Investigation of Tekton Custom Task integration with
  Agent Sandbox (documented, deferred).
- **2026-06-17**: Enhancement proposal drafted.
- **2026-06-18**: Revised with git-as-persistence workspace model,
  simplified AgentPlaybook (flat stages), phased rollout, connection
  to agent-base-image-composition enhancement.

## Drawbacks

1. **New repository and component**: Adds a Go controller to a project
   that currently runs Python, Java, and TypeScript.

2. **Agent Sandbox dependency**: v1beta1 SIG project. Mitigation:
   simple primitive, fallback to bare Pods feasible.

3. **skillimage dependency**: Young project (v0.7.2). Mitigation:
   Red Hat OCTO backing, Konveyor as significant consumer.

4. **In-cluster registry**: Required for git/inline SkillCards.
   Mitigation: Helm subchart for Zot, OpenShift auto-detection.

5. **Git as sole persistence**: If the git remote is unavailable, work
   in progress is lost on pod crash. Mitigation: incremental push
   during execution, not just on exit.

## Alternatives

### Hub as the controller

**Rejected**: ADR 0001 decouples agent execution from Hub.

### Tekton as the orchestration engine

**Deferred**: MVP requires only sequential execution. Architecture
does not preclude Tekton integration later.

### Bare Pods instead of Agent Sandbox

**Rejected**: Agent Sandbox provides warm pools, hibernation, network
isolation. Betting on the community project is stronger than building
custom pod management.

### PVCs for workspace persistence

**Rejected**: PVCs add lifecycle management complexity (create, retain,
GC per application). Git-as-persistence eliminates this — the workspace
is ephemeral, the branch is the durable state. No PVCs survive between
runs.

### SkillCards as pure OCI refs without CRDs

**Rejected**: Git sources and inline content need resolution to OCI
artifacts. CRDs provide visibility, RBAC, status reporting.

### Single base image with multiple runtimes

**Rejected**: Bloated image, different release cadences. One base per
runtime is cleaner.

### Separate branch for session context

**Deferred**: Committing `.konveyor/` files on a separate branch
(e.g., `konveyor/context/<run-name>`) keeps the working branch clean
for PRs. For now, session context goes on the same branch as code
changes — simpler, one branch to reason about. If the PR noise becomes
a problem, session files can be stripped at PR-creation time or moved
to a separate branch later.

## Infrastructure Needed

- **New repository**: `konveyor/agentic-controller`

- **Container image builds**: CI for controller image at
  `quay.io/konveyor/agentic-controller`

- **Base image builds**: CI for `agent-base`, `agent-base-goose`,
  `agent-base-opencode`, and language images. See
  [agent-base-image-composition](/enhancements/agentic-platform-agent-images/README.md).

- **OCI registry access**: Push to `quay.io/konveyor/`

- **Test cluster**: Agent Sandbox + ImageVolumes enabled
