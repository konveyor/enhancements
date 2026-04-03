---
title: migration-intelligence
authors:
  - "@djzager"
reviewers:
  - "@pranavgaikwad"
  - "@sjd78"
  - "@jwmatthews"
approvers:
  - "@pranavgaikwad"
  - "@sjd78"
  - "@jwmatthews"
creation-date: 2026-04-02
last-updated: 2026-04-03
status: implementable
see-also:
  - "/enhancements/kai/agent-driven-migration/README.md"
  - "https://github.com/konveyor/editor-extensions/issues/1243"
  - "https://github.com/konveyor/enhancements/pull/259"
  - "https://agentskills.io"
replaces: []
superseded-by: []
---

# Migration Intelligence

Enable AI agents to produce high-quality, phased migration plans by
providing them with migration-specific context through reusable skill
files, interactive user engagement, and analysis results — delivered
via an MCP server (stdio and HTTP) that works in the IDE today and in
hub containers at scale.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Terminology

This enhancement uses the following terms. All sections use these
definitions consistently.

- **Skill**: A folder in `.konveyor/skills/` following the
  [Agent Skills](https://agentskills.io) format. Each skill is a
  directory containing a `SKILL.md` file (with YAML frontmatter for
  `name` and `description`, plus Markdown instructions) and optional
  `scripts/`, `references/`, and `assets/` subdirectories. Skills use
  progressive disclosure — agents load only the name and description
  at discovery time, reading full instructions only when activated.

- **Migration plan**: A type of skill that captures high-level migration
  guidance, patterns, and organizational preferences applicable to a
  class of applications. Migration plans are typically created by
  architects and distributed via the hub. A migration plan answers:
  what does this migration mean, what patterns should be followed, and
  what organizational preferences apply.

- **Prompt**: A template stored in `.konveyor/prompts/` that guides
  agent behavior during specific migration workflows (skill creation,
  plan generation, phased execution). Prompts replace hardcoded
  intelligence in the extension, allowing workflow logic to be
  distributed and versioned alongside skills.

- **Migration intelligence package**: A standalone package in the
  editor-extensions repo (with its own `package.json`) containing
  built-in prompts, skills, agent instructions (`agents.md`), and
  MCP configuration. This is the portable unit of migration
  intelligence — the extension bundles it, but any MCP-compatible
  agent can consume it directly.

- **Profile**: An existing Konveyor concept — a bundle of rulesets
  (and now skills and prompts) synced from the hub. Profiles are
  associated with archetypes and distributed to matching applications.

## Open Questions

1. ~~**Skill format standardization**~~: **Resolved** — skills follow
   the [Agent Skills](https://agentskills.io) format: a directory with
   a `SKILL.md` file containing YAML frontmatter (`name`, `description`)
   and Markdown instructions, plus optional `scripts/`, `references/`,
   and `assets/` subdirectories.

2. **Hub API for skills**: What endpoints are needed for CRUD operations
   on skills? Should skills be a first-class resource in the hub or
   embedded in profile bundles?

3. **Archetype-skill association**: How should architects link skills to
   archetypes? Via tags (dynamic, like archetype membership) or explicit
   references?

4. **Prompt scope**: What workflows need prompt templates in
   `.konveyor/prompts/`? Is this a separate directory or can prompts
   be a category of skill?

## Summary

Today, when an AI agent attempts to fix migration violations, it receives
a profile name and a list of violations. It has no understanding of what
the migration means, what patterns the codebase uses, or what the
organization's preferences are. This is insufficient for producing
migration plans — the agent needs context before it can plan.

Migration Intelligence addresses this by introducing:

- **Skills** — directories in `.konveyor/skills/` following the
  [Agent Skills](https://agentskills.io) format that capture
  institutional knowledge about a migration (patterns, conventions,
  constraints). Created interactively by agents exploring the codebase
  and asking users questions, or authored manually by architects.
  Migration plans are a specific type of skill focused on high-level
  guidance for a class of applications.

- **A Migration Intelligence MCP server** — tools for skill management
  (`get_migration_context`, `save_skill`, `ask_user`) that any
  MCP-compatible agent can use. Supports both stdio and HTTP transports.
  Separate from the Analyzer MCP (PR #259) which provides analysis
  capabilities.

- **A migration intelligence package** — a standalone package in the
  editor-extensions repo containing built-in prompts, skills, agent
  instructions, and MCP configuration. The extension bundles this
  package, but it has its own `package.json` so others can consume it
  directly. This is what makes the CLI and IDE experiences equivalent.

- **A curated IDE experience** — a guided flow that takes users from
  analysis results through skill creation, plan generation, phased
  execution, and review — without requiring freeform prompt engineering.
  The extension acts as a thin MCP client, with intelligence driven by
  the migration intelligence package and workspace skills rather than
  hardcoded logic.

- **Skill distribution via the hub** — skills travel in profile bundles
  alongside rulesets, managed by architects via the hub UI and
  distributed to developers via the centralized configuration feature.

## Motivation

### The Planning Problem

The ideal migration path (analyze -> plan -> execute phases -> review)
breaks down at step 2: planning. To produce a useful phased plan, the
agent needs to understand:

- What does this migration actually mean? (e.g., Java EE -> Quarkus
  means javax -> jakarta, EJBs -> CDI, JMS -> Reactive Messaging)
- What patterns does this codebase use? (e.g., `@SessionScoped` on
  REST endpoints, custom base classes, internal frameworks)
- What are the organization's preferences? (e.g., use RabbitMQ not
  Kafka, prefer RESTEasy Reactive, don't touch Flyway migrations)
- What should be preserved vs. changed?

None of this context exists in the current pipeline. The agent gets
violations and file contents. That's not enough to plan.

### The Distribution Problem

Even if a developer creates useful migration context for one
application, that knowledge is trapped in their local workspace. An
architect who understands the migration deeply has no way to share
that knowledge with the 50 developers migrating similar applications
across the portfolio.

### Goals

1. **Enable migration planning**: Give agents enough context to produce
   phased migration plans from analysis results.

2. **Capture institutional knowledge**: Provide tools for agents and
   users to create skills interactively — exploring the codebase,
   asking questions, synthesizing findings.

3. **Distribute knowledge at scale**: Skills created by architects
   travel through the existing hub profile bundle mechanism to all
   developers working on matching applications.

4. **Curated IDE experience**: A guided flow that doesn't require users
   to write prompts. The extension orchestrates skill creation,
   planning, and phased execution with structured interactions.

5. **Portable MCP tools**: The intelligence layer is an MCP server
   supporting stdio and HTTP transports — running in the IDE via
   stdio, from the CLI, or in hub containers via HTTP.

### Non-Goals

1. **Replacing the Analyzer MCP (PR #259)**: This enhancement builds on
   top of analysis results. It does not re-implement analysis tools.

2. **Fully autonomous migration**: All agent-produced plans and changes
   require user review.

3. **Hub UI for plan execution**: The hub manages skills and tracks
   progress. Interactive plan execution happens in the IDE or DevSpaces.

4. ~~**Defining the skill format standard**~~: Resolved — skills use
   the [Agent Skills](https://agentskills.io) format.

## Proposal

### Skills

Skills follow the [Agent Skills](https://agentskills.io) format. Each
skill is a directory in `.konveyor/skills/` containing a `SKILL.md`
file with YAML frontmatter and Markdown instructions:

```text
.konveyor/skills/
├── java-ee-to-quarkus/
│   ├── SKILL.md
│   └── references/
│       └── cdi-patterns.md
└── reactive-messaging/
    └── SKILL.md
```

A migration plan is a specific type of skill. Here is an example
`SKILL.md` for a migration plan:

```markdown
---
name: java-ee-to-quarkus
description: >
  Migration plan for Java EE to Quarkus. Use when planning or
  executing a migration from JBoss EAP / Java EE to Quarkus 3.x.
---

# Java EE to Quarkus Migration Plan

## Application Context
Monolith e-commerce application on JBoss EAP 7.4, migrating to
Quarkus 3.x. Uses EJB, JPA, JMS, and JAX-RS.

## Key Patterns
- Services use @Stateless — convert to @ApplicationScoped
- ShippingService uses @Remote — refactor to REST endpoint
- JMS with JMSContext — migrate to SmallRye Reactive Messaging
  with RabbitMQ

## Organizational Preferences
- Use RabbitMQ (not Kafka) for messaging
- Use RESTEasy Reactive (not Classic)
- Preserve Flyway migrations — do not modify

## Things to Watch
- OrderService.save() needs explicit @Transactional
- CatalogService.updateInventoryItems() needs @Transactional
```

The Agent Skills format provides progressive disclosure — agents load
only the `name` and `description` at discovery time, reading full
`SKILL.md` instructions only when activated. Skills can also bundle
`scripts/`, `references/`, and `assets/` directories alongside
`SKILL.md` for executable code, documentation, and templates.

### Migration Intelligence MCP Server

A separate MCP server (distinct from the Analyzer MCP in PR #259) that
provides migration intelligence tools:

| Tool | Purpose |
|------|---------|
| `get_migration_context` | Returns active profile, labels, workspace info, existing skills |
| `get_skills` | Lists skills (name + description for discovery, full SKILL.md on activation) |
| `save_skill` | Creates a skill directory with SKILL.md in `.konveyor/skills/` |
| `ask_user` | Presents structured questions with options, returns answers |

#### Transport

The MCP server must support both **stdio** and **HTTP** transports:

- **stdio**: Primary transport for IDE use. The extension (or any MCP
  client) starts the MCP server as a subprocess. No ports opened, no
  HTTP connections required — critical for users who cannot open ports
  on their machines.

- **HTTP**: Transport for hub containers, DevSpaces, and remote
  scenarios where the MCP server runs as a long-lived service.

The same server binary supports both transports, selected at startup.
This ensures the tools work identically whether loaded by an IDE
extension, a CLI agent, or a hub container agent.

### Curated IDE Experience

The extension provides a guided flow that does not require freeform
prompt input. The extension acts as a thin MCP client — intelligence
is driven by skills and prompts from `.konveyor/skills/` and
`.konveyor/prompts/`, not by hardcoded logic in the extension.

#### Phase 1: Skill Creation

When a user has analysis results and no existing skills:

1. Extension detects analysis results are available
2. Offers: "Would you like to create a migration plan for this project?"
3. Agent calls `get_migration_context` + analysis tools (via Analyzer MCP)
4. Agent explores the codebase (reads key files)
5. Agent summarizes findings in the chat
6. Agent calls `ask_user` with structured questions about preferences
7. User responds with preferences (messaging provider, REST framework, etc.)
8. Agent calls `save_skill` with synthesized knowledge
9. Skill file appears in `.konveyor/skills/`

#### Phase 2: Plan Generation

With skills available:

1. Agent reads skills + analysis results
2. Agent produces a phased migration plan (markdown)
3. Plan is presented for review (approve/reject/reorder phases)
4. Plan saved to `.konveyor/skills/` as a migration plan

#### Phase 3: Phased Execution

With an approved migration plan:

1. Agent executes one phase at a time, guided by prompts and skills
2. Between phases, agent prompts user to review changes and confirm
   continuation
3. Re-analysis between phases catches cascading issues
4. Process repeats until migration is complete

Phased execution leverages prompts and skills to guide the user through
the migration plan step by step. The specific MCP tools needed to
support this workflow will be defined as the implementation matures —
changes to the Migration Intelligence MCP server may be required.

### Skill Distribution via Hub

Skills travel in profile bundles — the same mechanism that distributes
rulesets today.

#### Current Profile Bundle Structure

```
.konveyor/profiles/{profileId}/
├── profile.yaml
├── .hub-metadata.json
└── rules/
    ├── rule1.yaml
    └── rule2.yaml
```

#### Extended Bundle Structure

```text
.konveyor/profiles/{profileId}/
├── profile.yaml
├── .hub-metadata.json
├── rules/
│   ├── rule1.yaml
│   └── rule2.yaml
├── skills/
│   └── java-ee-to-quarkus/
│       └── SKILL.md
└── prompts/
    └── plan-generation.md
```

Adding `skills/` and `prompts/` directories to the bundle requires:

1. **Hub side**: Profile bundle creation includes skill and prompt files
2. **Extension side**: `ProfileSyncClient` extracts skills and prompts
   from the bundle alongside rulesets

No new distribution mechanism is needed. The existing
`syncProfiles()` -> TAR download -> extract pipeline handles additional
file types automatically.

### User Stories

#### Story 1: Architect creates a migration plan

An architect understands the Java EE to Quarkus migration deeply. They
open a representative application in VS Code, run analysis, and use the
guided flow to create a migration plan. The agent explores the codebase,
asks about organizational preferences (messaging provider, REST
framework, testing approach), and produces a comprehensive migration
plan.

The architect reviews the skill, makes edits, and pushes it to the hub
as part of a profile bundle. All developers migrating applications with
matching profiles receive this migration plan automatically via
centralized configuration.

#### Story 2: Developer uses a migration plan

A developer opens their application in VS Code. The extension syncs
profiles from the hub, including a migration plan created by the
architect. When the developer starts a migration, the agent has
rich context: the migration plan, the analysis results, and the
codebase. It produces a phased execution plan that reflects
organizational preferences.

The developer reviews the plan, approves it, and the agent executes
phases one at a time. Between phases, the agent asks structured
questions to confirm before proceeding.

#### Story 3: Architect manages skills in the hub

An architect opens the hub UI, navigates to an archetype (e.g., "Java EE
Monoliths targeting Quarkus"), and uploads a migration plan. The skill
is associated with the archetype. When developers' applications match
the archetype's criteria tags, they receive the skill in their profile
bundle.

### Implementation Details

#### Architecture

```
┌──────────────────────────────────────────────────┐
│  Agent (IDE / CLI / Container)                   │
└──────┬───────────────────────────┬───────────────┘
       │ stdio or HTTP             │ stdio or HTTP
┌──────┴──────────────┐  ┌────────┴────────────────┐
│ Migration           │  │ Analyzer MCP (PR #259)  │
│ Intelligence MCP    │  │                         │
│                     │  │ Analysis tools          │
│ get_migration_ctx   │  │ (run analysis,          │
│ get_skills          │  │  get results, etc.)     │
│ save_skill          │  │                         │
│ ask_user            │  │                         │
└──────┬──────────────┘  └────────┬────────────────┘
       │ reads from                │
┌──────┴───────────────────────────┴───────────────┐
│                                                  │
│  Migration intelligence package                  │
│  (editor-extensions repo, own package.json)      │
│  ├── prompts/     (workflow prompt templates)    │
│  ├── skills/      (built-in default skills)      │
│  ├── agents.md    (agent instructions)           │
│  └── mcp config   (MCP server configuration)     │
│                                                  │
│  .konveyor/ (workspace)                          │
│  ├── skills/      (local + hub-distributed)      │
│  ├── prompts/     (local + hub-distributed)      │
│  └── profiles/    (synced from hub)              │
│                                                  │
└──────────────────────────────────────────────────┘
```

**Key principle**: A developer running `claude --mcp konveyor` (or any
MCP-compatible agent) on a project should get the same migration
intelligence as a developer using the VS Code extension. The migration
intelligence package is the portable unit — the extension bundles it,
but it has its own `package.json` so anyone can point their agent at
it directly. The extension adds UI affordances (clickable options,
guided flows, chat panel rendering) — it does not add different
intelligence.

**Intelligence layering** — context is assembled from three sources,
with later sources able to extend or override earlier ones:

1. **Package defaults**: Built-in prompts, skills, and agent
   instructions from the migration intelligence package
2. **Hub-distributed**: Architect-authored skills and prompts synced
   via profile bundles into `.konveyor/profiles/{id}/`
3. **Workspace-local**: User or agent-created skills in
   `.konveyor/skills/` and `.konveyor/prompts/`

The extension acts as a **thin MCP client**. It:

- Starts MCP servers as subprocesses (stdio transport, no ports opened)
- Renders agent responses and user interactions in the chat UI
- Translates MCP interactions to VS Code commands where needed
  (e.g., rendering `ask_user` questions in the chat panel)

The extension does **not**:

- Run an HTTP server or open ports
- Hardcode migration intelligence or prompt construction logic
- Duplicate analysis context that the Analyzer MCP provides

All intelligence lives in the migration intelligence package and in
workspace skill/prompt files. The prompt builder does not construct
complex prompts from hardcoded logic — it assembles context from
the package and `.konveyor/`, the same content any MCP-connected
agent would consume.

#### `ask_user` Tool Design

The `ask_user` tool enables structured user interaction through the
MCP protocol:

1. Agent calls `ask_user` with `{ questions: [{ question, options }] }`
2. MCP server uses the protocol's interaction capabilities to present
   questions to the MCP client (the IDE or CLI)
3. The MCP client renders the questions in its UI (e.g., chat panel
   with clickable options in VS Code)
4. User selects options
5. Responses return through the MCP protocol to the agent

In a container context: the container pauses, answers arrive via the
hub API, and the container resumes.

This works over both stdio and HTTP transports without requiring the
extension to run an HTTP server.

#### Skill and Prompt Loading

When the agent needs migration context, the MCP server assembles it
from three layers:

1. **Package defaults**: Built-in prompts and skills from the migration
   intelligence package
2. **Hub-distributed**: Skills and prompts from
   `.konveyor/profiles/{id}/skills/` and `.konveyor/profiles/{id}/prompts/`
3. **Workspace-local**: Skills and prompts from `.konveyor/skills/`
   and `.konveyor/prompts/`

Content from later layers can extend or override earlier layers. Skills
matching the active profile (via label overlap or frontmatter) are
included in the agent's context. Prompts for the current workflow phase
are included.

This replaces the current approach where the extension's prompt builder
hardcodes intelligence. Skills and prompts are files — shipped in the
package, authored by architects, or created locally — consumed
identically by any MCP-connected agent.

### Security, Risks, and Mitigations

**Risk: Skill files contain sensitive information**
- *Mitigation*: Skills are markdown files in the workspace, subject
  to the same access controls as source code. Hub-distributed skills
  go through the hub's authentication.

**Risk: `ask_user` tool used to social-engineer users**
- *Mitigation*: Konveyor MCP tools are auto-approved (no permission
  prompt), but `ask_user` questions are clearly labeled as coming
  from the agent. Users choose from predefined options, not freeform
  input to the agent.

**Risk: Stale skills produce bad plans**
- *Mitigation*: Skills are versioned alongside profiles. Hub sync
  updates skills when architects push new versions. Local skills
  can be regenerated by re-running the skill creation flow.

## Design Details

### Test Plan

**Unit tests**:
- Skill discovery from `.konveyor/skills/` (SKILL.md frontmatter parsing)
- Skill activation (full SKILL.md loading)
- Prompt loading from `.konveyor/prompts/`
- `ask_user` question rendering and response collection

**Integration tests**:
- Full skill creation flow: agent explores -> asks -> saves
- Skill distribution via profile bundle sync
- MCP server stdio and HTTP transport lifecycle
- `ask_user` interaction lifecycle

**E2E tests**:
- Guided skill creation with coolstore application
- Verify skills are included in agent context for subsequent migrations

### Upgrade / Downgrade Strategy

**Upgrade**: Skills are opt-in. Existing workspaces without
`.konveyor/skills/` work exactly as before. The MCP server
gracefully handles the absence of skills and prompts.

**Downgrade**: Removing the migration intelligence MCP server has no
effect on analysis or existing fix workflows. Skills and prompts
remain as inert markdown files in the workspace.

**Bundle compatibility**: Profile bundles with `skills/` and `prompts/`
directories are backward-compatible — older extensions that don't
understand these directories ignore them during extraction.

## Implementation History

- **2026-04**: POC on `feature/migration-intelligence` branch
  - Migration Intelligence MCP server with 4 tools
  - Skill creation demonstrated with coolstore application
  - Interactive `ask_user` with clickable options in chat UI
  - Skill injection into migration prompts
  - Auto-approval of Konveyor MCP tools

## Drawbacks

1. **Skill quality depends on the agent**: AI-generated skills may
   contain inaccuracies. Architect review is important before
   distributing skills via the hub.

2. **Additional complexity in profile bundles**: Adding skills and
   prompts to bundles requires hub-side changes and UI for management.

3. **Maintenance burden**: Skills can become stale as migrations
   evolve. Need a process for architects to update skills.

## Alternatives

### Alternative 1: Embed migration knowledge in rulesets

Instead of separate skill files, extend ruleset YAML with migration
guidance fields.

**Pros**: Single source of truth for detection + resolution guidance.
**Cons**: Mixes concerns. Rule authors and migration architects are
often different people. Rulesets are structured YAML; migration plans
are prose.

### Alternative 2: No skill distribution — local only

Skills stay in the workspace, never distributed via hub.

**Pros**: Simpler. No hub changes needed.
**Cons**: Every developer creates their own skills from scratch.
Architect knowledge doesn't scale.

### Alternative 3: Use the solution server for context

Leverage the solution server's historical hints as migration context
instead of explicit skills.

**Pros**: Automatic — no manual skill creation needed.
**Cons**: Hints are per-violation, not per-migration. They don't
capture organizational preferences or codebase-wide patterns.

## Infrastructure Needed

- **Migration intelligence package**: Standalone package in the
  editor-extensions repo with its own `package.json` containing
  built-in prompts, skills, `agents.md`, and MCP configuration
- **Hub API**: Endpoints for skill and prompt CRUD operations on profiles
- **Hub UI**: Skill management interface (upload, associate with
  archetypes, preview)
- **Profile bundle format**: Support for `skills/` and `prompts/`
  directories in bundles (hub-side bundle creation, extension-side
  extraction)
- **MCP server binary**: Must support both stdio and HTTP transports
