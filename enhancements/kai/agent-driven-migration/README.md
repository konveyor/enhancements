---
title: agent-driven-migration-assistance
authors:
  - "@djzager"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-03-31
last-updated: 2026-04-09
status: provisional
see-also:
  - "https://github.com/konveyor/editor-extensions/issues/1243"
  - "https://github.com/konveyor/editor-extensions/issues/1334"
  - "/enhancements/kai/solution_server.md"
  - "/enhancements/distributed-language-extensions/README.md"
  - "https://github.com/konveyor/enhancements/pull/259"
  - "https://agentskills.io"
replaces: []
superseded-by: []
---

# Agent-Driven Migration Assistance

Evolve the Konveyor editor extension's AI solution pipeline from a single-shot
LLM workflow to a pluggable agent architecture where autonomous AI agents
(Goose, OpenCode, or others) can explore the codebase, execute tools, iterate on
fixes, and produce multi-file migration solutions — all under user-controlled
permission policies. Migration intelligence (skills, prompts, and agent
instructions) is extracted from the extension into a standalone, portable
package that any agent can consume — in the IDE, from the CLI, or in
containers.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. ~~**Credential refresh for Hub LLM proxy**~~: **Resolved** — addressed by
   the [credential file sidecar](https://github.com/konveyor/enhancements/pull/265)
   enhancement. The extension writes refreshed tokens to disk; the agent
   re-reads on auth failure.

2. **Backend consolidation**: The POC supports Goose and OpenCode as agent
   backends. Should the project commit to supporting multiple backends long-term,
   or converge on one? The pluggable architecture allows deferring this decision,
   but supporting multiple backends has ongoing maintenance cost.

3. **Session management**: When does the orchestrator start a new agent session
   vs. reuse an existing one? What context carries into a session? If session A
   made file changes, does session B know not to revert them? Is there tracking
   of context window usage? Can the user stop a running agent and redirect it
   mid-task?

## Summary

The Konveyor editor extension currently generates migration solutions through
its `KaiInteractiveWorkflow`, a LangGraph pipeline orchestrated by
`SolutionWorkflowOrchestrator`. The workflow has two modes controlled by the
`genai.agentMode` setting:

- **Without agent mode** (`genai.agentMode: false`): The workflow runs Phase 1
  only — processing incidents per-file using LLM generation without tool access.
  This is effectively single-shot-per-file: the LLM receives file contents and
  violation details, generates a fix, and returns. No tool use, no knock-on fix
  detection, no follow-up phase.

- **With agent mode** (`genai.agentMode: true`): Phase 1 additionally captures
  knock-on fix information. If identified, the user is prompted to continue.
  Phase 2 then runs follow-up agents with full tool access (file read/write/search,
  Maven dependency lookup) and a diagnostics loop for handling cascading changes.

Even with agent mode enabled, the workflow operates on a pre-defined set of
incidents, has no tool access during initial fix generation (Phase 1), and
cannot re-analyze after making changes to detect new issues. The underlying
agent is the `KaiInteractiveWorkflow` — there is no pluggable agent backend
and no integration with external agent tools like Goose or OpenCode.

This enhancement introduces a pluggable agent architecture that expands
beyond this workflow to autonomous AI agents capable of:

- **Codebase exploration**: Reading files, searching for patterns, understanding
  project structure before making changes
- **Tool use**: Executing build commands, running tests, querying external
  services via MCP servers
- **Iterative problem-solving**: Attempting a fix, checking the result, and
  refining the approach
- **Multi-file coordination**: Making related changes across many files in a
  single session
- **User-controlled autonomy**: Agent backends manage their own permission
  models; the IDE surfaces configuration for them

The architecture is backend-agnostic: a common `AgentClient` interface allows
different agent implementations (Goose via ACP protocol, OpenCode via SDK, or
direct LLM calls) to be used interchangeably. The extension handles permission
management, file change tracking, and user review regardless of which backend
produces the changes.

A fallback "workflow mode" preserves the existing single-shot LLM behavior for
environments where autonomous agents are unavailable or undesirable.

## Motivation

### Current State Problems

**Workflow limitations**: The current `KaiInteractiveWorkflow` has two modes.
Without agent mode, it is single-shot-per-file with no tool access. With agent
mode enabled, Phase 2 provides follow-up agents with file read/write/search
tools and Maven dependency lookup for knock-on fixes. Key limitations even
with agent mode:
- Phase 1 has no tool access — the LLM generates fixes without being able to
  read related files or verify its changes
- The workflow operates on a pre-defined set of incidents and cannot
  autonomously discover new issues
- The workflow cannot re-analyze after making changes to detect cascading
  issues introduced by the fixes themselves
- The two-phase structure constrains the fix→verify→iterate loop — the agent
  cannot freely interleave tool use with reasoning across phases
- There is no pluggable agent backend — the workflow is tightly coupled to the
  LangGraph-based `KaiInteractiveWorkflow`

**Minimal migration context**: The prompt contains only the profile name (e.g.,
"JavaEE to Quarkus"), violation messages, and file contents. There is no broader
migration guidance, no language-specific knowledge, and no access to external
tools that could improve solution quality. This intelligence is currently
hardcoded in the extension's prompt builder — it should live in portable skill
and prompt files that any agent can read.

**Tight coupling to one approach**: The solution pipeline is hardcoded to the
LangChain/LangGraph-based workflow. Evaluating alternative approaches or letting
users choose their preferred AI tool requires significant refactoring.

### Goals

1. **Pluggable agent backends**: Define an `AgentClient` interface that any agent
   implementation can satisfy. Ship with Goose (ACP protocol) and direct LLM
   fallback. Architecture supports adding more backends without modifying core
   orchestration.

2. **Autonomous codebase exploration**: Agents can read files, search code, and
   understand project structure before proposing changes, leading to
   higher-quality migration solutions.

3. **Tool integration via MCP**: Agents access Konveyor's analysis capabilities
   (run analysis, get incidents, apply changes) via MCP tools. This enhancement
   depends on the
   [Analyzer MCP proposal](https://github.com/konveyor/enhancements/pull/259)
   for the standalone analysis server; the extension provides an interim MCP
   bridge until that lands. Enable agents to use additional MCP servers for
   language-specific tasks (Maven search, npm audit, etc.).

4. **Permission control**: Each agent backend has its own native permission
   model (e.g., Goose uses `GOOSE_MODE`, OpenCode uses permission blocks,
   Claude Code uses `--allowedTools`). The extension delegates permission
   management to the backend rather than implementing a unified permission
   layer. The IDE provides affordances (settings UI, links to backend
   configuration) so users can configure their agent's permissions without
   leaving the extension.

5. **File change review**: All agent-produced file modifications are presented
   for user review via diff UI before being applied. Support both inline review
   (per-change as it happens) and batch review (queue changes, review all at
   once). Agents like Goose use built-in tools (e.g., Developer: Text Editor)
   that write directly to disk, bypassing MCP. The `AgentFileTracker` detects
   these changes by caching file contents before the agent runs and comparing
   against disk after tool calls complete — enabling diff rendering regardless
   of which tool the agent used.

6. **Portable migration intelligence**: Extract migration-specific knowledge
   (skills, prompts, agent instructions) from the extension into a standalone
   package that any agent can consume. A developer running Goose from the CLI
   with this package gets the same migration intelligence as a developer using
   the VS Code extension. Skills follow the
   [Agent Skills](https://agentskills.io) format.

7. **Backward compatibility**: The existing direct LLM workflow remains available
   as a configuration option (`genai.agentMode: false`). Users who don't want
   autonomous agents can use the familiar single-shot behavior.

### Non-Goals

1. **Building a new agent framework**: This enhancement integrates existing agent
   tools (Goose, OpenCode), it does not create a new one.

2. **Replacing the analysis pipeline**: The analyzer, rulesets, and violation
   detection are unchanged. This enhancement only affects the solution generation
   side.

3. **Automated migration without human review**: All agent-produced changes go
   through user review. Changes proposed via MCP tools are staged for review
   before being written to disk. Changes written directly by the agent (via its
   built-in file tools) are detected by the `AgentFileTracker` and can be
   reverted; the user reviews these changes before they are finalized. The goal
   is to assist developers, not to run unattended.

4. **Multi-user or server-side agent hosting**: Agents run locally as
   subprocesses of the VS Code extension. Server-side agent hosting is out of
   scope.

5. **Skill distribution via hub**: The mechanism for distributing
   architect-authored skills and prompts through hub profile bundles is out of
   scope. This enhancement defines the package format; hub integration is
   separate work.

## Proposal

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│ VS Code Extension Host                                              │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ Agent Feature Module                                          │  │
│  │                                                               │  │
│  │  AgentOrchestrator ──┬── AgentClient (Goose/OpenCode/Direct)  │  │
│  │       │              │         │                               │  │
│  │       │              └── AgentFileTracker                     │  │
│  │       │                                                       │  │
│  │       ├── MCP Bridge Server (localhost HTTP)                   │  │
│  │       │       │                                                │  │
│  │       │       └── Konveyor MCP Server (stdio)                 │  │
│  │       │             • run_analysis                             │  │
│  │       │             • get_analysis_results                     │  │
│  │       │             • incidents_by_file                        │  │
│  │       │             • apply_file_changes                       │  │
│  │       │                                                       │  │
│  │       └── FileChangeRouter ── BatchReviewQueue / ChatMessages │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              ↕                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ Webview UI (React + PatternFly)                               │  │
│  │  • Chat interface with streaming responses                    │  │
│  │  • Permission review (allow/reject with diff preview)         │  │
│  │  • Batch file review carousel                                 │  │
│  │  • Agent settings (provider, model, backend configuration)    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              ↕                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ Agent Subprocess (Goose / OpenCode)                           │  │
│  │  • Reads skills/prompts from intelligence package             │  │
│  │  • Explores codebase, executes tools                          │  │
│  │  • Streams text, tool calls, permission requests              │  │
│  │  • Connects to Konveyor MCP Server for analysis data          │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              ↕                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ Migration Intelligence Package (standalone, own package.json) │  │
│  │  • AGENTS.md / SKILL.md — agent instructions                  │  │
│  │  • skills/ — migration workflow skills (Agent Skills format)  │  │
│  │  • prompts/ — workflow prompt templates                       │  │
│  │  • Consumed by extension, CLI agents, or containers alike     │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Migration Intelligence Package

The extension currently hardcodes migration intelligence in its prompt
builder — constructing prompts with inline logic that understands what
context to include, how to frame violations, and what guidance to give
the agent. This is not portable: a developer using Goose from the CLI,
or running an agent in a container, gets none of this intelligence.

This enhancement extracts that intelligence into a standalone package
in the editor-extensions repo with its own `package.json`:

```text
intelligence/
├── package.json
├── AGENTS.md                    # Agent instructions (Claude Code, Cursor)
├── SKILL.md                     # Agent Skills standard (Goose, OpenCode)
├── skills/
│   ├── create-migration-guide/
│   │   └── SKILL.md             # Interview user, explore code, create guide
│   ├── generate-migration-plan/
│   │   └── SKILL.md             # Guide + violations → phased plan
│   └── execute-migration-phase/
│       └── SKILL.md             # Execute one phase, re-analyze, prompt user
└── prompts/
    └── ...                      # Workflow prompt templates
```

Skills follow the [Agent Skills](https://agentskills.io) format. Each
skill is a directory containing a `SKILL.md` file with YAML frontmatter
(`name`, `description`) and Markdown instructions, plus optional
`scripts/`, `references/`, and `assets/` subdirectories. The format
provides progressive disclosure — agents load only the `name` and
`description` at discovery time, reading full instructions only when
activated.

The skills in this package define migration workflows. They are not MCP
tools — they are instructions that any agent reads and follows. The
agent uses its built-in file tools to read/write files and the
[Analyzer MCP](https://github.com/konveyor/enhancements/pull/259) for
analysis capabilities (`run_analysis`, `get_analysis_results`).

**Key principle**: A developer running any MCP-compatible agent on a
project with this package gets the same migration intelligence as a
developer using the VS Code extension. The extension adds UI affordances
(clickable options, guided flows, chat panel rendering) — it does not
add different intelligence.

The extension bundles this package and reads from it at runtime, but the
package has no dependency on the extension. The same files work when
consumed by Goose, Claude Code, OpenCode, or any agent that can read
skill files from disk.

### User Stories

#### Story 1: Developer uses agent to fix migration violations

A developer has analyzed their Java EE application for migration to Quarkus.
The analysis found 15 violations across 8 files. The developer selects all
violations and clicks "Get Solution."

The agent receives the violations and begins working autonomously:
1. Reads the affected files plus related files (pom.xml, application.properties)
2. Understands the dependency relationships between violations
3. Makes changes to Java source files (EJB → CDI, javax → jakarta)
4. Updates pom.xml to add Quarkus dependencies and remove Java EE ones
5. Modifies application.properties for Quarkus configuration

Each file modification appears in the chat as a permission request with a diff
preview. The developer reviews each change, approving most and rejecting one
that touches a file they want to handle manually. The agent continues working
with the remaining changes.

#### Story 2: Developer configures agent permissions

A developer working on a sensitive codebase wants the agent to help but doesn't
trust it to execute arbitrary commands. They open Agent Settings in the
extension, which surfaces the backend's native permission configuration. For
example, with Goose the developer sets `GOOSE_MODE=approve` so every tool call
requires approval; with Claude Code they configure `--allowedTools` to restrict
which tools the agent can use. The extension provides quick access to these
backend-specific settings without requiring the developer to leave the IDE.

#### Story 3: Developer uses workflow mode for simple fixes

A developer prefers the focused, predictable behavior of single-shot LLM fixes.
They set `genai.agentMode: false` in settings. When they click "Get Solution," the
extension sends one prompt to the configured LLM and presents the response as a
set of file diffs for per-file accept/reject — the same behavior as before the
agent architecture was introduced.

#### Story 4: Agent uses Konveyor analysis tools

During a migration fix, the agent realizes it needs to check whether its changes
introduced new violations. It calls the `run_analysis` MCP tool, waits for
results, then calls `get_analysis_results` to check. Finding that its pom.xml
change triggered a new dependency-related violation, it makes an additional fix
and re-checks. The developer sees this iteration in the chat log and approves
the final set of changes.

### Implementation Details

#### AgentClient Interface

The core abstraction is the `AgentClient` interface that all backends implement:

```typescript
interface AgentClient extends EventEmitter {
  // Lifecycle
  start(): Promise<void>;
  stop(): Promise<void>;
  dispose(): void;

  // Communication
  sendMessage(message: string, sessionId?: string): Promise<void>;
  createSession(): Promise<string>;
  cancelGeneration(): Promise<void>;

  // Events emitted:
  // - stateChange: AgentState transitions (stopped → starting → running → error)
  // - streamingChunk: Incremental text/thinking content
  // - streamingComplete: Full message with all content blocks
  // - toolCall: Tool invocation with name, arguments, status
  // - toolCallUpdate: Progress updates for long-running tools
  // - permissionRequest: Tool needs user approval before executing
  // - error: Error conditions
}
```

Each backend translates its native protocol to these events:
- **GooseClient**: JSON-RPC 2.0 over stdio (ACP protocol), parses
  `session/update` notifications into events
- **OpencodeAgentClient**: OpenCode SDK, maps SDK callbacks to events
- **DirectLLMClient**: Wraps existing KaiInteractiveWorkflow, emits synthetic
  permissionRequest events for file writes

The `AgentClient` interface is an internal abstraction — new backends are added
by implementing the interface in the extension codebase. Third-party registration
of agent backends (e.g., via a plugin API) is out of scope, but the interface
makes it straightforward to add new implementations as the agent ecosystem
evolves.

#### Agent Orchestrator

The `AgentOrchestrator` is the central coordinator for solution requests:

```
User clicks "Get Solution"
  → AgentOrchestrator.run(incidents, scope)
    → Check genai.agentMode configuration
    → If genai.agentMode=true AND agent backend available:
        Agent Path:
        1. Get/create AgentClient from feature registry
        2. Create new session
        3. Pre-cache incident files (for diff detection)
        4. Build migration prompt from incidents + intelligence package
        5. Send prompt to agent
        6. Handle streaming events:
           - Text chunks → accumulate into chat messages
           - Tool calls → display in chat with status
           - Permission requests → handled by agent backend natively
           - File changes → route through FileChangeRouter
        7. On completion: scan for any missed file changes
    → If genai.agentMode=false:
        Workflow Path:
        1. Create DirectLLMClient (wraps KaiInteractiveWorkflow)
        2. Send prompt, receive structured file modifications
        3. Present per-file accept/reject via existing UI
```

#### Permission Management

Permission management is delegated to each agent backend's native model.
Each backend has its own mechanism for controlling what the agent can do
autonomously:

- **Goose**: `GOOSE_MODE` environment variable (`auto`, `approve`, etc.)
- **OpenCode**: Permission block configuration
- **Claude Code**: `--allowedTools` flag and permission settings

The extension does not implement a unified permission layer or attempt to
translate between these models. Instead, the IDE provides affordances for
configuring each backend's permissions:

- Settings UI surfaces backend-specific permission options
- Links to backend documentation for advanced configuration
- Sensible defaults when launching agent subprocesses

#### MCP Bridge Architecture (Interim)

This bridge is a temporary stand-in for the
[Analyzer MCP](https://github.com/konveyor/enhancements/pull/259), which is a
dependency of this enhancement. Once the Analyzer MCP is available, the bridge
is replaced entirely. The extension exposes Konveyor's analysis capabilities to
agents via MCP in the interim:

```
Agent subprocess
  ↕ (stdio)
Konveyor MCP Server (Node.js process)
  ↕ (HTTP to localhost)
MCP Bridge Server (runs in extension process)
  ↕ (direct API calls)
Konveyor analyzer / extension state
```

The extension starts the MCP bridge server in `initializeAgent` with workspace
context from the extension store (workspace root, profiles, rulesets, analyzer
client). The bridge port is passed to the MCP server via `KONVEYOR_BRIDGE_PORT`
environment variable when the agent configures its MCP servers. In CLI/container
contexts, the equivalent startup would be handled by kantra or a similar CLI
tool, passing workspace path and configuration at startup.

**Note**: Some users report being unable to open ports in their environments.
The current bridge uses HTTP on localhost, but the MCP specification also
supports stdio transport. If port restrictions are a blocker, the bridge (or
the Analyzer MCP that replaces it) should support MCP over stdio as an
alternative transport.

The MCP server exposes four tools:

- `run_analysis` — Trigger the analyzer on the workspace
- `get_analysis_results` — Fetch current incidents and rulesets
- `incidents_by_file` — Query incidents for a specific file path
- `apply_file_changes` — Propose file modifications through the extension's
  review pipeline. Agents can also write files directly using their built-in
  tools (e.g., Goose's `Developer: Text Editor`); the `AgentFileTracker`
  detects those changes after the fact. This tool provides an explicit,
  permission-gated channel so that changes are routed through batch review
  or inline approval rather than written to disk first

This two-hop architecture (agent → MCP server → bridge → extension) exists
because the agent communicates with MCP servers via stdio, but the analysis state
lives in the extension process. The bridge server is the boundary between them.

When the Analyzer MCP is available, the bridge is removed. The Analyzer MCP
replaces the bridge's analysis tools (`run_analysis`, `get_analysis_results`,
`incidents_by_file`) with a standalone server that communicates via stdio and
manages analysis lifecycle internally (e.g., triggering re-analysis on file
changes rather than exposing `run_analysis` as a separate tool). This also
eliminates the localhost HTTP hop and the port-availability concern.

#### File Change Tracking

The `AgentFileTracker` detects file modifications made by the agent:

1. **Pre-caching**: Before the agent runs, cache the contents of all files
   referenced in the incidents being fixed
2. **Permission-time caching**: When a permission request arrives for a file
   write tool, cache the file's current state before the write executes
3. **Post-completion scanning**: Agents can modify files through paths the
   extension doesn't directly observe — bash commands (`sed -i`, `echo >`),
   auto-approved tool calls, or tools the classifier didn't recognize as
   file-editing. After the agent finishes, scan all cached files against current
   disk state to detect any changes that weren't caught during the session. This
   ensures the batch review UI shows ALL modifications for user approval

Detected changes are routed through the `FileChangeRouter`:
- **Batch review mode**: Changes queue to `pendingBatchReview` for carousel-style
  review (accept/reject each file)
- **Inline mode**: Changes appear as chat messages with diff previews

#### Chat UI

The agent chat interface provides:

- **Streaming responses**: Text appears incrementally as the agent generates it
- **Tool call indicators**: Shows which tools the agent is using and their status
- **Permission review**: Inline diff previews with Allow/Reject buttons and
  cardinality options (once/always)
- **Thinking blocks**: Agent reasoning is captured (optionally displayed)
- **Resource links**: URIs referenced by the agent are rendered as clickable links
- **Incident navigation**: Violations mentioned in chat link to source locations

#### Agent Settings

A settings panel allows configuration of:

- **LLM provider and model**: Supports OpenAI, Anthropic, Azure, Ollama, Bedrock,
  and custom endpoints
- **Agent backend**: Goose or OpenCode (future: additional backends)
- **Backend permissions**: Surface backend-native permission configuration
- **Credentials**: Stored in VS Code SecretStorage (encrypted per platform)
- **Agent extensions**: Enable/disable Goose extensions and MCP servers

### Security, Risks, and Mitigations

**Risk: Agent executes harmful commands**
- *Mitigation*: Each agent backend provides its own permission model for
  controlling tool access. The extension surfaces these settings in the IDE
  and launches backends with sensible defaults. Users can restrict agent
  autonomy through backend-native configuration.

**Risk: Agent modifies files outside the workspace**
- *Mitigation*: File change tracking detects all modifications. Changes proposed
  via MCP tools are staged for review before disk write. Changes written directly
  by agent file tools are detected by `AgentFileTracker`, which pre-caches
  original content so they can be reverted. In both paths, the user reviews
  changes before they are finalized. Agents operate in the workspace directory.

**Risk: Credential exposure to agent subprocess**
- *Mitigation*: Credentials are passed via environment variables (not command
  line arguments). VS Code SecretStorage provides encrypted storage. Hub token
  refresh is handled by the
  [credential file sidecar](https://github.com/konveyor/enhancements/pull/265)
  pattern — the extension writes refreshed tokens to disk, and the agent
  re-reads on auth failure.

**Risk: MCP bridge server accessible to other processes**
- *Mitigation*: The bridge is an interim stand-in that will be replaced by the
  Analyzer MCP (which communicates via stdio, eliminating the network surface
  entirely). While the bridge is in use, it binds to a random localhost port
  and is only accessible from the same machine.

**Risk: Agent produces incorrect migration code**
- *Mitigation*: All changes require user review. The diff UI shows exactly what
  changed. Batch review mode lets users review all changes before any are applied.
  The agent can also re-run analysis via MCP tools to validate its changes.

## Design Details

### Configuration

New VS Code settings:

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `genai.agentMode` | boolean | `true` | Enable autonomous agent mode |
| `experimentalChat.agentBackend` | enum | `"goose"` | Agent backend selection |
| `experimentalChat.gooseBinaryPath` | string | `""` | Path to Goose binary |
| `experimentalChat.opencodeBinaryPath` | string | `""` | Path to OpenCode binary |
| `genai.batchReviewMode` | boolean | `false` | Queue file changes for batch review |

### State Management

The extension uses a Zustand store with Immer middleware for state management.
Agent state (chat messages, batch review queue, agent configuration) is stored
centrally and synchronized to webviews via sync bridges.

Key state slices:
- `agentState`: Current lifecycle state (stopped/starting/running/error)
- `agentChatMessages`: Conversation history with tool calls and content blocks
- `pendingBatchReview`: File changes queued for batch review

### Test Plan

**Unit tests**:
- FileChangeRouter: Verify routing to batch queue vs. chat messages

**Integration tests**:
- Agent lifecycle: start → send message → receive response → stop
- Permission flow: tool call → permission request → user response → tool
  execution
- File change detection: agent modifies file → tracker detects → diff generated
- MCP bridge: agent calls MCP tool → bridge routes to extension → result returned

**E2E tests**:
- Full solution workflow: select violations → get solution → review changes →
  accept/reject
- Agent settings: configure provider, model, backend permissions
- Batch review: queue multiple files → navigate carousel → accept/reject each
- Workflow mode fallback: `genai.agentMode=false` produces same behavior as before

### Upgrade / Downgrade Strategy

**Upgrade from pre-agent extension**: `genai.agentMode` defaults to `true`,
but the extension gracefully falls back to workflow mode if no agent binary is
available. Users who upgrade without installing Goose/OpenCode will see no change
in behavior. A notification suggests installing an agent backend for enhanced
capabilities.

**Downgrade**: Setting `genai.agentMode: false` restores the previous single-shot LLM
behavior. No data migration is needed — agent chat history is session-scoped and
not persisted between extension activations.

**Backend switching**: Users can switch between Goose and OpenCode via settings
at any time. Each backend starts fresh with a new session. Configuration
(provider, model, credentials) is shared across backends where possible.

## Implementation History

- **2026-02**: POC implementation on `feature/goose` branch
  ([editor-extensions#1243](https://github.com/konveyor/editor-extensions/issues/1243))
  - Pluggable AgentClient interface with Goose, OpenCode, and DirectLLM backends
  - Permission management delegated to agent backends
  - MCP bridge server exposing Konveyor analysis tools
  - Chat UI with streaming, tool calls, and permission review
  - Batch file review carousel
  - Agent settings panel
  - Zustand + Immer state management
  - ~14,000 lines of new code across 156 files

- **2026-03**: Credential refresh blocker identified
  ([editor-extensions#1334](https://github.com/konveyor/editor-extensions/issues/1334))
  — resolved via [credential file sidecar](https://github.com/konveyor/enhancements/pull/265)

## Drawbacks

1. **Complexity**: The agent architecture adds significant complexity compared to
   the single-shot LLM approach — subprocess management, permission systems, MCP
   bridging, file tracking, and multiple UI components.

2. **External dependencies**: Agents (Goose, OpenCode) are external projects with
   their own release cycles, bugs, and breaking changes. The extension must track
   upstream changes and adapt.

3. **Binary distribution**: Agent binaries (Goose, OpenCode) must be installed
   separately. This adds friction to the user experience compared to the
   self-contained LLM workflow.

4. **Resource consumption**: Agent subprocesses consume additional memory and CPU.
   Goose in particular spawns its own subprocess tree. This may be problematic on
   resource-constrained machines.

5. ~~**Credential management**~~: The credential refresh problem
   ([#1334](https://github.com/konveyor/editor-extensions/issues/1334)) is
   resolved by the
   [credential file sidecar](https://github.com/konveyor/enhancements/pull/265)
   enhancement.

## Alternatives

### Alternative 1: Enhance the existing LangGraph workflow

Instead of introducing external agents, improve `KaiInteractiveWorkflow` with
more nodes, tools, and iteration loops.

**Pros**:
- No external dependencies
- Full control over the agent's behavior
- No subprocess management or credential forwarding
- In-process execution, simpler architecture

**Cons**:
- Requires building and maintaining a full agent framework
- Cannot leverage existing agent ecosystems (Goose extensions, MCP servers)
- Duplicates work that agent frameworks already do well
- Higher ongoing development cost for equivalent capability

### Alternative 2: Single agent backend (Goose only)

Commit to Goose as the only agent backend, removing the pluggable interface.

**Pros**:
- Simpler architecture (no interface abstraction, no backend switching)
- Deeper integration with one tool
- Less code to maintain

**Cons**:
- Vendor lock-in to a single project
- Cannot evaluate alternatives without major refactoring
- Users who prefer other tools cannot use them
- If Goose development stalls, the extension is stuck

### Alternative 3: IDE-native agent (VS Code Copilot Chat integration)

Use VS Code's built-in Copilot Chat API to provide agent capabilities.

**Pros**:
- Native VS Code experience
- No external binary installation
- Built-in credential management via VS Code accounts
- Consistent with VS Code ecosystem direction

**Cons**:
- Copilot Chat API is still evolving and has limited extensibility
- Ties the extension to GitHub Copilot licensing
- Cannot control the agent's tool use or iteration strategy
- May not support the autonomous workflows needed for complex migrations
- Not available in VS Code forks (VSCodium, Theia, etc.)

## Infrastructure Needed

- **Migration intelligence package**: Standalone package in the editor-extensions
  repo with its own `package.json` containing built-in skills, prompts, and
  agent instructions (`AGENTS.md` / `SKILL.md`). The extension bundles this
  package; CLI and container agents consume it directly from disk.

- **CI changes**: Build and test matrix should verify that the extension works
  correctly in both agent mode and workflow mode. Agent-dependent tests may need
  mock backends.

- **Documentation**: User-facing documentation for agent setup (installing Goose,
  configuring providers, understanding permission levels) and migration from the
  previous LLM-only workflow.

- **Upstream coordination**: Coordination with the Goose project team may be
  needed for agent-side integration of the credential file sidecar pattern
  ([enhancements#265](https://github.com/konveyor/enhancements/pull/265)).
