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
  - "/enhancements/kai/migration-intelligence/README.md"
  - "https://github.com/konveyor/enhancements/pull/259"
replaces: []
superseded-by: []
---

# Agent-Driven Migration Assistance

Evolve the Konveyor editor extension's AI solution pipeline from a single-shot
LLM workflow to a pluggable agent architecture where autonomous AI agents
(Goose, OpenCode, or others) can explore the codebase, execute tools, iterate on
fixes, and produce multi-file migration solutions — all under user-controlled
permission policies.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. **Credential refresh for Hub LLM proxy**: Agent subprocesses receive Hub
   bearer tokens at spawn time. These tokens have a ~5 minute TTL. Once expired,
   the agent cannot make LLM requests and there is no mechanism to deliver
   refreshed tokens to the running subprocess. This blocks Hub LLM proxy
   integration with agent mode. See
   [editor-extensions#1334](https://github.com/konveyor/editor-extensions/issues/1334).
   Options under consideration:
   - Agent-side credential refresh (requires upstream Goose changes to OpenAI provider)
   - Long-lived API keys from Hub
   - Credential file sidecar pattern (extension writes refreshed tokens to disk,
     agent re-reads on auth failure)

2. ~~**AI artifact distribution**~~: **Resolved** — addressed by the
   [Migration Intelligence](/enhancements/kai/migration-intelligence/README.md)
   enhancement. Skills follow the [Agent Skills](https://agentskills.io) format
   and are distributed via hub profile bundles alongside rulesets.

3. **Backend consolidation**: The POC supports Goose and OpenCode as agent
   backends. Should the project commit to supporting multiple backends long-term,
   or converge on one? The pluggable architecture allows deferring this decision,
   but supporting multiple backends has ongoing maintenance cost.

## Summary

The Konveyor editor extension currently generates migration solutions by sending
a single prompt to an LLM and receiving file modifications back. This works for
straightforward changes but falls short for complex migrations that require the
AI to explore the codebase, understand dependency relationships, iterate on
failed attempts, and coordinate changes across multiple files.

This enhancement introduces a pluggable agent architecture that replaces the
single-shot LLM call with autonomous AI agents capable of:

- **Codebase exploration**: Reading files, searching for patterns, understanding
  project structure before making changes
- **Tool use**: Executing build commands, running tests, querying external
  services via MCP servers
- **Iterative problem-solving**: Attempting a fix, checking the result, and
  refining the approach
- **Multi-file coordination**: Making related changes across many files in a
  single session
- **User-controlled autonomy**: Granular permission policies that let users
  decide which agent actions require approval

The architecture is backend-agnostic: a common `AgentClient` interface allows
different agent implementations (Goose via ACP protocol, OpenCode via SDK, or
direct LLM calls) to be used interchangeably. The extension handles permission
management, file change tracking, and user review regardless of which backend
produces the changes.

A fallback "workflow mode" preserves the existing single-shot LLM behavior for
environments where autonomous agents are unavailable or undesirable.

## Motivation

### Current State Problems

**Single-shot LLM limitations**: The current `KaiInteractiveWorkflow` sends one
prompt with violation details and file contents, then receives one response with
proposed changes. The LLM cannot:
- Read additional files to understand context
- Try a fix and check if it compiles
- Discover that fixing one file requires changes in another
- Use external tools (Maven, npm, build systems)

**No iteration capability**: When a migration fix introduces new issues (e.g.,
updating an import requires updating a dependency in pom.xml), the current
pipeline has no way to detect or address cascading changes.

**Minimal migration context**: The prompt contains only the profile name (e.g.,
"JavaEE to Quarkus"), violation messages, and file contents. There is no broader
migration guidance, no language-specific knowledge, and no access to external
tools that could improve solution quality.

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

3. **Tool integration via MCP**: Expose Konveyor's analysis capabilities
   (run analysis, get incidents, apply changes) as MCP tools that agents can
   invoke. Enable agents to use additional MCP servers for language-specific
   tasks (Maven search, npm audit, etc.).

4. **Granular permission control**: Users control what the agent can do
   autonomously vs. what requires approval. Three autonomy levels (auto, smart,
   ask) with per-category overrides for file editing, command execution, web
   access, and MCP tools.

5. **File change review**: All agent-produced file modifications are presented
   for user review via diff UI before being applied. Support both inline review
   (per-change as it happens) and batch review (queue changes, review all at
   once).

6. **Backward compatibility**: The existing direct LLM workflow remains available
   as a configuration option (`agentMode: false`). Users who don't want
   autonomous agents can use the familiar single-shot behavior.

### Non-Goals

1. **Building a new agent framework**: This enhancement integrates existing agent
   tools (Goose, OpenCode), it does not create a new one.

2. **Replacing the analysis pipeline**: The analyzer, rulesets, and violation
   detection are unchanged. This enhancement only affects the solution generation
   side.

3. **Automated migration without human review**: All agent-produced changes go
   through user review. The goal is to assist developers, not to run unattended.

4. **Multi-user or server-side agent hosting**: Agents run locally as
   subprocesses of the VS Code extension. Server-side agent hosting is out of
   scope.

5. **AI artifact distribution**: The mechanism for distributing migration-specific
   skills and prompts is addressed by the
   [Migration Intelligence](/enhancements/kai/migration-intelligence/README.md)
   enhancement.

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
│  │       │              │    ToolPermissionHandler                │  │
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
│  │  • Agent settings (provider, model, autonomy, permissions)    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              ↕                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ Agent Subprocess (Goose / OpenCode)                           │  │
│  │  • Receives migration prompt                                  │  │
│  │  • Explores codebase, executes tools                          │  │
│  │  • Streams text, tool calls, permission requests              │  │
│  │  • Connects to Konveyor MCP Server for analysis data          │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

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

#### Story 2: Developer configures agent autonomy level

A developer working on a sensitive codebase wants the agent to help but doesn't
trust it to execute arbitrary commands. They open Agent Settings and configure:
- Autonomy level: "smart" (default)
- File editing: "ask" (require approval for every file change)
- Command execution: "deny" (never allow shell commands)
- Web access: "auto" (allow without asking)

The agent can now read files and search the web freely, but every file
modification requires explicit approval and shell commands are blocked entirely.

#### Story 3: Developer uses workflow mode for simple fixes

A developer prefers the focused, predictable behavior of single-shot LLM fixes.
They set `agentMode: false` in settings. When they click "Get Solution," the
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

#### Agent Orchestrator

The `AgentOrchestrator` is the central coordinator for solution requests:

```
User clicks "Get Solution"
  → AgentOrchestrator.run(incidents, scope)
    → Check agentMode configuration
    → If agentMode=true AND agent backend available:
        Agent Path:
        1. Get/create AgentClient from feature registry
        2. Create new session
        3. Pre-cache incident files (for diff detection)
        4. Build migration prompt from incidents
        5. Send prompt to agent
        6. Handle streaming events:
           - Text chunks → accumulate into chat messages
           - Tool calls → display in chat with status
           - Permission requests → route through ToolPermissionHandler
           - File changes → route through FileChangeRouter
        7. On completion: scan for any missed file changes
    → If agentMode=false:
        Workflow Path:
        1. Create DirectLLMClient (wraps KaiInteractiveWorkflow)
        2. Send prompt, receive structured file modifications
        3. Present per-file accept/reject via existing UI
```

#### Tool Permission System

Permissions are evaluated through a layered policy:

```
ToolPermissionPolicy:
  autonomyLevel: "auto" | "smart" | "ask"    (global default)
  overrides:                                   (per-category)
    fileEditing: "auto" | "ask" | "deny"
    commandExecution: "auto" | "ask" | "deny"
    webAccess: "auto" | "ask" | "deny"
    mcpTools: "auto" | "ask" | "deny"
    other: "auto" | "ask" | "deny"
```

The **"smart"** default auto-approves read-only operations and web access, but
requires approval for file writes and command execution. This balances agent
productivity with user control.

A `ToolClassifier` maps tool names to categories. For example, Goose reports
tools as `"Developer: Text Editor"` which maps to `text_editor` → `fileEditing`.
Read-only operations (viewing files, undoing edits) are detected and never
permission-gated.

The policy is translated to backend-native formats:
- Goose: `GOOSE_MODE` environment variable
- OpenCode: Permission block configuration

#### MCP Bridge Architecture (Interim)

The extension exposes Konveyor's analysis capabilities to agents via MCP:

```
Agent subprocess
  ↕ (stdio)
Konveyor MCP Server (Node.js process)
  ↕ (HTTP to localhost)
MCP Bridge Server (runs in extension process)
  ↕ (direct API calls)
Konveyor analyzer / extension state
```

The bridge server runs on a random localhost port, passed to the MCP server via
environment variable. The MCP server exposes four tools:

- `run_analysis` — Trigger the analyzer on the workspace
- `get_analysis_results` — Fetch current incidents and rulesets
- `incidents_by_file` — Query incidents for a specific file path
- `apply_file_changes` — Apply file modifications (goes through permission system)

This two-hop architecture (agent → MCP server → bridge → extension) exists
because the agent communicates with MCP servers via stdio, but the analysis state
lives in the extension process. The bridge server is the boundary between them.

**Note**: This MCP bridge is an interim solution. The
[Analyzer MCP proposal](https://github.com/konveyor/enhancements/pull/259)
proposes a standalone MCP server for analyzer-lsp that would provide these
analysis tools directly. When the Analyzer MCP lands, the bridge's analysis
tools (`run_analysis`, `get_analysis_results`, `incidents_by_file`) will be
replaced by the Analyzer MCP, and the bridge will be simplified or removed.

#### File Change Tracking

The `AgentFileTracker` detects file modifications made by the agent:

1. **Pre-caching**: Before the agent runs, cache the contents of all files
   referenced in the incidents being fixed
2. **Permission-time caching**: When a permission request arrives for a file
   write tool, cache the file's current state before the write executes
3. **Post-completion scanning**: After the agent finishes, scan all cached files
   for modifications that may have been missed (e.g., if the agent modified a
   file without going through the permission system)

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
- **Autonomy level**: Global setting with per-category overrides
- **Credentials**: Stored in VS Code SecretStorage (encrypted per platform)
- **Agent extensions**: Enable/disable Goose extensions and MCP servers

### Security, Risks, and Mitigations

**Risk: Agent executes harmful commands**
- *Mitigation*: Tool permission system with configurable autonomy levels. Default
  "smart" mode requires user approval for file writes and command execution.
  "deny" option completely blocks specific categories.

**Risk: Agent modifies files outside the workspace**
- *Mitigation*: File change tracking detects all modifications. All changes go
  through user review before being applied to the workspace. Agents operate in
  the workspace directory.

**Risk: Credential exposure to agent subprocess**
- *Mitigation*: Credentials are passed via environment variables (not command
  line arguments). VS Code SecretStorage provides encrypted storage. However,
  the credential refresh problem (issue #1334) means Hub tokens currently expire
  in the subprocess.

**Risk: MCP bridge server accessible to other processes**
- *Mitigation*: The bridge server binds to a random localhost port. It is only
  accessible from the same machine. In the future, authentication tokens could
  be added for additional security.

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
| `genai.autonomyLevel` | enum | `"smart"` | Global tool permission level |
| `genai.permissionOverrides` | object | `{}` | Per-category permission overrides |
| `experimentalChat.agentBackend` | enum | `"goose"` | Agent backend selection |
| `experimentalChat.gooseBinaryPath` | string | `""` | Path to Goose binary |
| `experimentalChat.opencodeBinaryPath` | string | `""` | Path to OpenCode binary |
| `genai.batchReviewMode` | boolean | `false` | Queue file changes for batch review |

### State Management

The extension uses a Zustand store with Immer middleware for state management.
Agent state (chat messages, pending permissions, batch review queue, agent
configuration) is stored centrally and synchronized to webviews via sync bridges.

Key state slices:
- `agentState`: Current lifecycle state (stopped/starting/running/error)
- `agentChatMessages`: Conversation history with tool calls and content blocks
- `pendingPermissions`: Permission requests awaiting user response
- `pendingBatchReview`: File changes queued for batch review
- `toolPermissionPolicy`: Current permission configuration

### Test Plan

**Unit tests**:
- ToolClassifier: Verify tool name → category mapping for all known tool patterns
- ToolPermissionHandler: Verify policy evaluation for all autonomy level ×
  category combinations
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
- Agent settings: configure provider, model, autonomy level
- Batch review: queue multiple files → navigate carousel → accept/reject each
- Workflow mode fallback: agentMode=false produces same behavior as before

### Upgrade / Downgrade Strategy

**Upgrade from pre-agent extension**: The `agentMode` setting defaults to `true`,
but the extension gracefully falls back to workflow mode if no agent binary is
available. Users who upgrade without installing Goose/OpenCode will see no change
in behavior. A notification suggests installing an agent backend for enhanced
capabilities.

**Downgrade**: Setting `agentMode: false` restores the previous single-shot LLM
behavior. No data migration is needed — agent chat history is session-scoped and
not persisted between extension activations.

**Backend switching**: Users can switch between Goose and OpenCode via settings
at any time. Each backend starts fresh with a new session. Configuration
(provider, model, credentials) is shared across backends where possible.

## Implementation History

- **2026-02**: POC implementation on `feature/goose` branch
  ([editor-extensions#1243](https://github.com/konveyor/editor-extensions/issues/1243))
  - Pluggable AgentClient interface with Goose, OpenCode, and DirectLLM backends
  - Tool permission system with autonomy levels and per-category overrides
  - MCP bridge server exposing Konveyor analysis tools
  - Chat UI with streaming, tool calls, and permission review
  - Batch file review carousel
  - Agent settings panel
  - Zustand + Immer state management
  - ~14,000 lines of new code across 156 files

- **2026-03**: Credential refresh blocker identified
  ([editor-extensions#1334](https://github.com/konveyor/editor-extensions/issues/1334))

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

5. **Credential management**: The credential refresh problem
   ([#1334](https://github.com/konveyor/editor-extensions/issues/1334)) blocks
   Hub LLM proxy integration with agent mode, limiting deployment options for
   enterprise users.

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

- **CI changes**: Build and test matrix should verify that the extension works
  correctly in both agent mode and workflow mode. Agent-dependent tests may need
  mock backends.

- **Documentation**: User-facing documentation for agent setup (installing Goose,
  configuring providers, understanding permission levels) and migration from the
  previous LLM-only workflow.

- **Upstream coordination**: The credential refresh problem requires changes in
  Goose's OpenAI provider implementation
  ([block/goose#7812](https://github.com/block/goose/pull/7812)). Coordination
  with the Goose project team is needed.
