---
title: agent-base-image-composition
authors:
  - "@hitpatel"
reviewers:
  - TBD
  - "@djzager"
  - "@dymurray"
approvers:
  - TBD
creation-date: 2026-06-18
last-updated: 2026-06-18
status: provisional
see-also:
  - "/enhancements/agentic-platform-controller/README.md"
  - "https://github.com/redhat-et/skillimage"
  - "https://github.com/kubernetes-sigs/agent-sandbox"
  - "https://agentskills.io"
replaces: []
superseded-by: []
---

# Agent Base Image Composition

Defines the container image layers for the Konveyor Agentic Platform agent
workloads. Covers the base image contents, runtime layers, language
toolchain layers, filesystem conventions, non-root execution, multi-arch
support, user extension path, and build pipeline.

This enhancement fills the gap identified in the
[agentic-platform-controller](/enhancements/agentic-platform-controller/README.md)
enhancement:

> "Agent container image composition (how language-specific images are
> built) is out of scope and will be a separate enhancement."

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions 

1. **graphify installation method**: graphify is currently a pip package
   with C extensions (tree-sitter). Should it remain pip-installed
   (requires Python dev headers in the image) or be pre-compiled as a
   standalone binary to reduce image size?

2. **Core skills in /opt/skills/**: Which skills, if any, should be baked
   into agent-base? The agentic-platform-controller enhancement
   references "core skills" but does not enumerate them. Should the
   base image be domain-agnostic (no migration skills) or include
   Konveyor migration skills by default?

3. **Python in agent-base**: Python 3.12 is required by graphify.
   Should Python also be in language-specific layers for Python
   application migrations, or is the base Python sufficient?


## Summary

Agent container images follow a layered hierarchy. Each layer adds
capabilities on top of the previous one. The base image contains the
harness entrypoint, Konveyor tooling (skillctl, graphify), and system
utilities. Runtime layers add a specific agent runtime (goose or
opencode). Language layers add build tools for specific application
types.

Skills are NOT baked into the image. They are mounted at runtime via
ImageVolumes from OCI artifacts resolved by the SkillCard controller.
The base image contains `skillctl` (the skillimage CLI) for skill
discovery and inspection at runtime, not the skills themselves.

## Motivation

### Current State

Migration harnesses exist today as research and developer tooling.
Projects like migration-harness (goose-based) and migIQ (Claude Code
skills) have proven the workflow — code graph analysis, planning,
execution, verification — across multiple LLM runtimes and models.
These run on developer machines as CLI tools and have been tested with
real application migrations (Java EE to Quarkus).

This enhancement takes those proven skills and runs them on-cluster
in Sandboxes, where:
- End users get a platform to run full migrations — from legacy code
  to deployed application on OpenShift — without local tooling setup
- Migrations run at scale — N applications in parallel, each in its
  own isolated Sandbox
- Skills, models, and images are managed as cluster resources, not
  local config files

### Goals

Define the agent base image — its contents, layer hierarchy, filesystem
conventions, and build pipeline — so that Sandboxes have a consistent,
runtime-agnostic foundation to run agent workloads on.

Specifically:

1. **Define what goes in agent-base**: harness entrypoint, skillctl,
   graphify, system utilities, filesystem layout
2. **Define runtime layers**: how goose and opencode are added on top
3. **Define language layers**: how Java, Node.js, .NET, Go toolchains
   are added
4. **UBI compliance**: Red Hat Universal Base Image for OpenShift
   certification
5. **Non-root execution**: compatible with OpenShift's restricted SCC
6. **Multi-architecture**: amd64 and arm64
7. **User extensibility**: enterprise users extend with a single FROM
8. **Skills mounted, not baked**: image provides skillctl for skill
   discovery, not the skills themselves

### Non-Goals

CRDs, controller logic, and harness internals are covered by the
agentic-platform-controller enhancement. Skill authoring is the domain
of the skillimage project. This enhancement covers what's in the image
and where things live on the filesystem.

## Proposal

### Image Layer Hierarchy

```
agent-base                          UBI 9 + harness + skillctl + graphify + tools
  └── agent-base-{runtime}          extends base + runtime binary (goose, opencode, ...)
        └── agent-{lang}-{runtime}  extends runtime + language toolchain
```

The naming pattern is `agent-{lang}-{runtime}`. Examples:

| Image | Base | Adds |
|---|---|---|
| `agent-base` | UBI 9 | harness, skillctl, graphify, git, jq, curl, Python 3.12 |
| `agent-base-goose` | agent-base | goose binary |
| `agent-base-opencode` | agent-base | opencode binary |
| `agent-java-{runtime}` | agent-base-{runtime} | JDK 21 + Maven |
| `agent-node-{runtime}` | agent-base-{runtime} | Node.js 20 + npm |
| `agent-dotnet-{runtime}` | agent-base-{runtime} | .NET 8 SDK |
| `agent-go-{runtime}` | agent-base-{runtime} | Go 1.23 |
| `agent-full-{runtime}` | agent-base-{runtime} | All language toolchains |

Where `{runtime}` is `goose`, `opencode`, or any future runtime.
Where `{lang}` is `java`, `node`, `dotnet`, `go`, or `full`.

### Layer 0: agent-base

Base image for all Konveyor agent workloads. Contains no agent runtime
and no skills. Provides the harness entrypoint, Konveyor tooling, and
system utilities.

**Base**: `registry.access.redhat.com/ubi9/ubi-minimal:latest`

**Contents**:

| Path | Component | Purpose |
|---|---|---|
| `/usr/local/bin/konveyor-harness` | Harness entrypoint (Go binary) | Translates KONVEYOR_* env vars to runtime-specific config, discovers skills, invokes runtime |
| `/usr/local/bin/skillctl` | skillimage CLI | Skill discovery, inspection, build, push, pull of OCI skill artifacts |
| `/usr/local/bin/graphify` | graphify CLI (Python package) | Code graph analysis via tree-sitter AST extraction |
| `/usr/local/bin/fetch-analysis` | Konveyor tool | Fetches analysis results from Hub API |
| `/usr/local/bin/run-analysis` | Konveyor tool | Runs Kantra analysis against application source |
| `/usr/bin/git` | git | Version control |
| `/usr/bin/jq` | jq | JSON processing |
| `/usr/bin/curl` | curl | HTTP client |
| `/usr/bin/ssh` | ssh | Git SSH access |
| `/usr/bin/python3` | Python 3.12 | Required by graphify |

**Directories**:

| Path | Purpose | Permissions |
|---|---|---|
| `/opt/skills/` | Core skills (baked in, if any) | Read-only, readable by any UID |
| `/.konveyor/` | Harness output (session ID, results) | Writable by any UID |
| `/workspace/` | Workspace PVC mount point | Writable by any UID |
| `$HOME/.config/` | Runtime config (written by harness at startup) | Writable by any UID |

**Entrypoint**: `/usr/local/bin/konveyor-harness`

**Estimated size**: ~400MB (UBI minimal + Python 3.12 + graphify + Go binaries + CLI tools)

### Layer 1: Runtime Layers (agent-base-{runtime})

Each runtime layer extends agent-base with a specific agent runtime
binary installed to `/usr/local/bin/`. The harness auto-detects which
runtime is present by checking PATH.

```dockerfile
FROM quay.io/konveyor/agent-base:latest
# Install the runtime binary (goose, opencode, etc.)
# Must be at /usr/local/bin/{runtime} for non-root access
```

| Runtime | Binary source | Estimated size |
|---|---|---|
| goose | GitHub releases (`goose-{arch}-unknown-linux-gnu.tar.bz2`) | ~500MB |
| opencode | opencode.ai install script | ~450MB |

New runtimes are added by creating a new `agent-base-{runtime}`
Dockerfile. No changes to agent-base or language layers required.

### Layer 2: Language Toolchain Layers (agent-{lang}-{runtime})

Each language layer extends a runtime layer with build tools needed
by the verify and fix phases to compile and test migrated code.

```dockerfile
FROM quay.io/konveyor/agent-base-{runtime}:latest
# Install language toolchain
```

| Language | What's installed | Estimated total size |
|---|---|---|
| `agent-java-{runtime}` | OpenJDK 21, Maven 3.9+ | ~1.1GB |
| `agent-node-{runtime}` | Node.js 20 LTS, npm | ~700MB |
| `agent-dotnet-{runtime}` | .NET 8 SDK | ~900MB |
| `agent-go-{runtime}` | Go 1.23 | ~900MB |
| `agent-full-{runtime}` | All of the above | ~2.0GB |

Every language layer is identical regardless of runtime — only the
FROM line changes. A single Dockerfile with a build arg handles all
combinations:

```dockerfile
ARG RUNTIME=goose
FROM quay.io/konveyor/agent-base-${RUNTIME}:latest

ARG LANG=java
# ... install toolchain based on LANG
```

Build example:

```bash
# Java + goose
docker build --build-arg RUNTIME=goose --build-arg LANG=java -t agent-java-goose .

# Java + opencode
docker build --build-arg RUNTIME=opencode --build-arg LANG=java -t agent-java-opencode .
```

### Filesystem Conventions

#### Skill Locations

The harness discovers skills from two locations:

| Path | Source | Priority |
|---|---|---|
| ImageVolume mounts | Mounted by controller from SkillCard OCI artifacts | Primary |
| `/opt/skills/` | Baked into the image | Fallback |

If a skill with the same name exists in both locations, the ImageVolume
mount takes precedence. The harness uses `skillctl` to read skill
metadata (`skill.yaml`) from each location to determine type (skill vs
rule) and other properties.

#### Output Convention

The harness writes results to `/.konveyor/` on the workspace PVC:

```
/.konveyor/
  session-id              # agent runtime session ID for resumption
  results.json            # structured results (status, duration, errors)
```

The AgentRun controller reads this path after Pod completion to update
AgentRun CR status. This path is a transient communication channel —
the CR is the permanent record.

#### Workspace PVC

The workspace PVC is mounted at `/workspace/`. All agent work happens
here — cloned repositories, migration artifacts, phase outputs. The
PVC persists across Sandboxes within an AgentPlaybookRun.

### Non-Root Execution

OpenShift runs containers with a random non-root UID by default
(restricted SCC). All images must support this.

**Requirements**:

1. All binaries installed to system paths (`/usr/local/bin/`,
   `/usr/bin/`), not user-specific paths like `/root/.local/bin/`.

2. Writable directories use group permissions:
   ```dockerfile
   RUN mkdir -p /workspace /.konveyor \
       && chmod 775 /workspace /.konveyor \
       && chgrp -R 0 /workspace /.konveyor
   ```

3. The harness uses `$HOME` (set by OpenShift to a writable tmpdir)
   for runtime config, not hardcoded `/root/`.

4. `/opt/skills/` is read-only. Skills are reference content, not
   written to at runtime.

5. The harness creates `$HOME/.config/<runtime>/` at startup for
   runtime-specific config files (goose config, opencode config).

**Tested with**: `runAsUser: 1000650000` (typical OpenShift random UID).

### Multi-Architecture

All images are built for amd64 and arm64 using multi-arch manifests.

Architecture-specific components:

| Component | amd64 | arm64 |
|---|---|---|
| UBI 9 | Multi-arch base | Multi-arch base |
| Harness | Go cross-compile | Go cross-compile |
| skillctl | Go cross-compile | Go cross-compile |
| graphify | pip install (builds tree-sitter C extensions per arch) | pip install |
| goose | `goose-x86_64-unknown-linux-gnu.tar.bz2` | `goose-aarch64-unknown-linux-gnu.tar.bz2` |
| JDK | `java-21-openjdk` (arch-specific RPM) | `java-21-openjdk` (arch-specific RPM) |

Build pipeline uses `podman manifest create` or `docker buildx` to
produce multi-arch manifest lists pushed to `quay.io/konveyor/`.

### User Extension

Enterprise users extend images with a single FROM line:

```dockerfile
FROM quay.io/konveyor/agent-java-goose:latest

# Add internal CA certificates
COPY internal-ca.crt /etc/pki/ca-trust/source/anchors/
RUN update-ca-trust

# Add internal tools
RUN microdnf install -y internal-linter && microdnf clean all

# Add internal skills (baked in for environments without ImageVolumes)
COPY my-company-rules/ /opt/skills/my-company-rules/
```

**Stable contract for user images**:

| What users can depend on | Guaranteed |
|---|---|
| `/usr/local/bin/konveyor-harness` exists | Yes |
| `/usr/local/bin/skillctl` exists | Yes |
| `/usr/local/bin/graphify` exists | Yes |
| `/workspace/` mount point exists | Yes |
| `/.konveyor/` output directory exists | Yes |
| `/opt/skills/` skill directory exists | Yes |
| Entrypoint is `konveyor-harness` | Yes |
| Non-root compatible | Yes |
| KONVEYOR_* env var interface | Yes |

### Build Pipeline

**Registry**: `quay.io/konveyor/`

**Image names**:
```
quay.io/konveyor/agent-base:latest
quay.io/konveyor/agent-base-goose:latest
quay.io/konveyor/agent-base-opencode:latest
quay.io/konveyor/agent-java-goose:latest
quay.io/konveyor/agent-java-opencode:latest
quay.io/konveyor/agent-node-goose:latest
quay.io/konveyor/agent-dotnet-goose:latest
quay.io/konveyor/agent-go-goose:latest
quay.io/konveyor/agent-full-goose:latest
```

**Tagging strategy**:
- `:latest` — latest build from main branch
- `:v1.0.0` — release tags
- `:sha-abc1234` — commit-based tags for traceability

**Rebuild triggers**:
- UBI base image update (security patches)
- goose/opencode new release
- graphify new release
- skillctl new release
- Harness source code change

**CI**: GitHub Actions workflow builds all images on push to main and
on release tags. Multi-arch builds run on native runners (amd64) and
QEMU emulation (arm64).

## Design Details

### Image Size Targets

| Image | Target Size | Rationale |
|---|---|---|
| agent-base | < 500MB | UBI minimal + Python + Go binaries |
| agent-base-goose | < 600MB | Base + goose (~100MB) |
| agent-java-goose | < 1.2GB | Goose + JDK 21 (~500MB) + Maven (~100MB) |
| agent-node-goose | < 800MB | Goose + Node.js (~200MB) |
| agent-dotnet-goose | < 1.0GB | Goose + .NET SDK (~400MB) |
| agent-full-goose | < 2.2GB | Goose + all languages |

Current migration-harness monolithic image: ~2.7GB. The full image
is smaller because UBI minimal is leaner than ubuntu:24.04, and the
layered approach avoids duplicate base packages.

### Test Plan

**Image build tests** (CI):
- Each Dockerfile builds successfully for amd64 and arm64
- `konveyor-harness --version` returns expected version
- `skillctl version` returns expected version
- `graphify --version` returns expected version
- `goose --version` returns expected version (runtime layers)
- Language tools available: `java -version`, `mvn --version`,
  `node --version`, `dotnet --version`, `go version`

**Non-root tests** (CI):
- All images start successfully with `runAsUser: 1000650000`
- Harness can write to `/workspace/` and `/.konveyor/`
- Harness can create `$HOME/.config/goose/` directory
- `/opt/skills/` is readable

**Integration tests** (real cluster):
- Deploy agent-java-goose image as a Sandbox with ImageVolume skills
- Harness discovers skills from both `/opt/skills/` and ImageVolume mounts
- Harness translates KONVEYOR_* env vars and invokes goose
- Goose completes a trivial skill execution
- Results written to `/.konveyor/results.json`

### Security

- All images based on UBI 9 (Red Hat security updates, CVE scanning)
- Non-root by default (compatible with OpenShift restricted SCC)
- No credentials baked into images (injected at runtime via Secrets)
- `/opt/skills/` is read-only (skills are reference content)
- OCI artifacts (skills) support cosign signing for supply chain verification

## Drawbacks

1. **Many images to maintain**: 12+ images across runtimes and
   languages. Each goose or opencode release requires rebuilding
   multiple images. Mitigation: shared base layers minimize rebuild
   scope; CI automates builds.

2. **Python in base image**: graphify requires Python 3.12, adding
   ~150MB to every image. If graphify were a standalone binary, Python
   could move to a language layer. Mitigation: graphify is actively
   developed; pre-compiled binary may be feasible in future.

3. **UBI minimal limitations**: UBI minimal uses microdnf with a
   limited package set. Some language toolchains may require switching
   to UBI standard. Mitigation: test each language layer with UBI
   minimal; fall back to UBI standard only if needed.

## Alternatives

### Single fat image (current migration-harness approach)

One image with all languages, runtime, and skills baked in.

**Rejected because**: 2.7GB image with mostly unused components per
migration. Slow pulls, wasted disk, no runtime or language flexibility.

### Devcontainer-based images

Use devcontainer features to compose images dynamically.

**Deferred, not rejected**: Devcontainer features provide a composable
image model. However, the feature ecosystem for agent runtimes (goose,
opencode) does not exist yet. This can be revisited when devcontainer
adoption grows in the Konveyor community.

### Buildpacks

Use Cloud Native Buildpacks to detect and install language runtimes.

**Rejected because**: Buildpacks detect application language and build
the app. Agent images need language tools for verification, not for
building the agent itself. The abstraction doesn't match the use case.

## Infrastructure Needed

- **Image builds in CI**: GitHub Actions workflows for all images,
  multi-arch builds, push to quay.io/konveyor/.

- **quay.io push access**: Automated push access for CI to
  `quay.io/konveyor/agent-*` repositories.

- **Test cluster**: OpenShift cluster with Agent Sandbox and
  ImageVolumes enabled for integration testing.
