---
title: analyzer-mcp-server
authors:
  - "@JonahSussman"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-01-23
last-updated: 2026-04-16
status: implementable
see-also:
  - "https://modelcontextprotocol.io/"
  - "https://github.com/modelcontextprotocol/specification"
  - "https://github.com/konveyor/kai/tree/main/kai_analyzer_rpc"
replaces:
  - N/A
superseded-by:
  - N/A
---

# Analyzer MCP Server

Add a standalone Model Context Protocol (MCP) server that wraps the kai-analyzer-rpc engine to expose Konveyor's analysis capabilities to AI agents, IDEs, and automation tools.

**Implementation PRs:**
- [konveyor/kai#926](https://github.com/konveyor/kai/pull/926) - Export Analyzer accessors (merged)
- [konveyor-ecosystem/analyzer-mcp#1](https://github.com/konveyor-ecosystem/analyzer-mcp/pull/1) - MCP server implementation
- [konveyor/koncur#63](https://github.com/konveyor/koncur/pull/63) - MCP and RPC target integration for e2e testing
- [konveyor/ci#207](https://github.com/konveyor/ci/pull/207) - CI action and image build matrix

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

1. What is the long-term strategy for versioning the MCP tools as the analyzer API evolves?

## Summary

This enhancement adds a standalone MCP server (`analyzer-mcp`) in a dedicated repository ([konveyor-ecosystem/analyzer-mcp](https://github.com/konveyor-ecosystem/analyzer-mcp)) that wraps the [kai-analyzer-rpc](https://github.com/konveyor/kai/tree/main/kai_analyzer_rpc) engine. By living in a separate repo, the server can import both `analyzer-lsp` and `kai-analyzer` without circular dependency issues, and it gets kai-analyzer's caching, incremental analysis, and provider lifecycle management for free.

The server exposes 9 tools through MCP, supports stdio and HTTP transports with bearer token authentication, and integrates with [koncur](https://github.com/konveyor/koncur) for end-to-end testing.

Key capabilities exposed through MCP:
- `analyze`: Execute full analysis with configurable rules, label selectors, and scoping
- `get_analysis_results`: Retrieve cached results from the last analysis run
- `analyze_incidents`: Query and filter incidents by file, rule, or category
- `list_rules`: Enumerate available rules with metadata and labels
- `validate_rules`: Validate rule syntax and structure
- `list_providers`: List available analysis providers and their capabilities
- `get_dependencies`: Extract dependency trees from analyzed projects
- `get_migration_context`: Infer migration sources and targets from rules
- `notify_file_changes`: Notify providers of file changes for incremental analysis

## Motivation

Current analyzer-lsp usage requires direct CLI invocation (via kantra) or LSP server integration, limiting automation and AI-assisted workflows. Users who want to integrate analysis into AI agents, CI/CD pipelines, or custom tooling must parse command-line output or interact with the LSP protocol, which is designed for IDEs rather than programmatic access.

The Model Context Protocol provides a standardized way for AI assistants and automation tools to interact with external services. By implementing an MCP server, we enable:

1. **AI-Assisted Migration**: AI assistants can run analyses, interpret results, and suggest code fixes in a conversational workflow
2. **Custom Workflows**: Developers can build automation tools that leverage analysis capabilities
3. **CI/CD Integration**: Remote HTTP access enables analysis in containerized and distributed environments
4. **Interactive Exploration**: Users can query rules, validate syntax, and explore dependencies conversationally

### Goals

- Expose core analysis functionality through the MCP protocol
- Leverage kai-analyzer-rpc's caching, incremental analysis, and provider management
- Support both local (stdio) and remote (HTTP) access patterns
- Maintain security through bearer token authentication for HTTP
- Integrate with koncur for e2e test coverage
- Provide a single binary with no external runtime dependencies

### Non-Goals

- Replace the existing kantra CLI interface (MCP is complementary)
- Replace the LSP server for IDE integration
- Modify core analyzer-lsp engine behavior
- Provide a web UI

## Proposal

Implement an MCP server as a standalone binary in a new repository ([konveyor-ecosystem/analyzer-mcp](https://github.com/konveyor-ecosystem/analyzer-mcp)). The server imports `kai-analyzer-rpc` in-process (passing `nil` for `*rpc.Client`) and delegates all analysis operations to `service.Analyzer`. This gives us caching, incremental analysis, provider lifecycle management, and progress reporting without reimplementation.

### User Stories [optional]

#### Story 1: AI-Assisted Code Migration

As a developer migrating a Java EE application to Quarkus, I want to use an AI assistant to analyze my codebase, explain migration issues, and suggest fixes.

1. Configure the AI assistant with the analyzer-mcp server (stdio transport)
2. Ask: "Analyze my project for Java EE to Quarkus migration issues"
3. The assistant calls `analyze` with the appropriate label selector
4. The assistant explains the incidents found and suggests code changes
5. Follow-up questions use `analyze_incidents` to filter by file or rule

#### Story 2: Custom Rule Development

As a rule author, I want to validate my custom rules interactively before committing them.

1. Write a new rule in my editor
2. Use an MCP client to call `validate_rules` on my rule file
3. Get immediate feedback on syntax errors or missing fields
4. Use `analyze` to test the rule on a sample project
5. Iterate based on results without leaving my workflow

#### Story 3: CI/CD Integration

As a DevOps engineer, I want to run analysis on pull requests.

1. Deploy MCP server with HTTP transport
2. Configure bearer token authentication
3. CI job calls `analyze` on the PR branch
4. Parse results and post comments on PRs with migration issues

### Implementation Details/Notes/Constraints [optional]

**Architecture:**
```
                MCP Client (Claude Code, VS Code, CI/CD)
                         |
                         | MCP Protocol (stdio or HTTP)
                         |
                  analyzer-mcp binary
                  (MCP tool handlers)
                         |
                         | In-process Go import
                         |
                  kai-analyzer-rpc
                  service.Analyzer
                  (caching, providers, progress)
                         |
                         | Uses internally
                         |
                  analyzer-lsp
                  (engine, providers, parser, output types)
```

**Repository layout:**
```
analyzer-mcp/
├── go.mod
├── go.sum
├── Makefile
├── Dockerfile
└── cmd/
    └── analyzer-mcp/
        ├── main.go               # CLI entry: cobra flags, transport selection
        ├── server.go             # NewMCPServer(), Run(), bearerAuthMiddleware()
        ├── service.go            # AnalyzerService interface + types
        ├── analyzer_service.go   # kaiAnalyzerService: wraps kai-analyzer
        ├── tools.go              # 9 MCP tool handler registrations
        ├── helpers.go            # Shared utilities
        ├── main_test.go          # CLI flag tests
        ├── tools_test.go         # Unit tests with mockAnalyzerService
        ├── helpers_test.go       # Helper function tests
        ├── service_test.go       # kaiAnalyzerService unit tests
        └── integration_test.go   # Full workflow + HTTP + auth tests
```

**Key design decisions:**

1. **Separate repo**: Avoids circular Go module dependencies between analyzer-lsp and kai-analyzer. The MCP server imports both freely.
2. **In-process kai-analyzer**: `service.NewPipeAnalyzer()` is called with `nil` for `*rpc.Client`. Single binary, no IPC.
3. **Delegation over reimplementation**: Caching, incremental analysis, provider lifecycle, and progress reporting all come from kai-analyzer.
4. **Interface-based testing**: An `AnalyzerService` interface allows mocking the entire analysis backend for unit tests.

**What kai-analyzer provides (used directly):**
- Full analysis with `Analyze()`
- Caching via internal `IncidentsCache`
- Incremental analysis via `IncludedPaths`
- File change notification via `NotifyFileChanges()`
- Provider lifecycle via `NewPipeAnalyzer()` and `Stop()`
- Progress reporting via `ProgressReporter` interface
- Label selectors, path scoping, cache reset

**Upstream changes required (merged):**
- [konveyor/kai#926](https://github.com/konveyor/kai/pull/926): Export 4 accessor methods on the `Analyzer` interface (`Providers()`, `RuleSets()`, `Cache()`, `CachedRuleSets()`) so the MCP server can query live provider state, rule metadata, and cached results.

### Security, Risks, and Mitigations

1. **Path Traversal**: Paths are validated by the underlying analyzer-lsp engine. The MCP server passes paths through without additional processing.

2. **Resource Exhaustion**: Large codebases could consume excessive resources. Mitigated by kai-analyzer's existing limits and HTTP request timeouts.

3. **Authentication**: HTTP transport uses bearer token authentication. Stdio transport relies on process-level access control.

4. **Information Disclosure**: Error messages are mapped to MCP error codes. Detailed errors only in debug mode.

## Design Details

### Test Plan

**Unit tests (59 tests, 96% coverage):**
- All 9 tool handlers tested with `mockAnalyzerService`
- Parameter validation, error handling, output format verification
- Helper functions (`filterByIncidentSelector`, `incidentVariables`, etc.)
- `kaiAnalyzerService` delegation tests

**Integration tests:**
- Full MCP session lifecycle via in-memory transport
- HTTP server with bearer token authentication
- Multi-tool workflows (analyze then query results)
- Error propagation through the MCP protocol

**E2E tests (via koncur):**
- [konveyor/ci#207](https://github.com/konveyor/ci/pull/207): CI action that builds images, starts providers, extracts the binary, and runs koncur tests against the MCP target
- [konveyor/koncur#63](https://github.com/konveyor/koncur/pull/63): MCPTarget implementation that connects to the server via stdio or HTTP and runs the standard koncur test suite

### Upgrade / Downgrade Strategy

- MCP server is a new standalone component, no upgrade concerns for existing installations
- Can be deployed alongside existing kantra/analyzer-lsp installations
- No changes to existing CLI or LSP functionality
- Tool input schemas will be versioned in descriptions; breaking changes result in new tool names

## Implementation History

- 2026-01-23: Enhancement proposal created
- 2026-03-20: Initial prototype in analyzer-lsp ([konveyor/analyzer-lsp#1146](https://github.com/konveyor/analyzer-lsp/pull/1146))
- 2026-04-07: Team meeting decided to move to separate repo wrapping kai-analyzer-rpc
- 2026-04-07: Closed analyzer-lsp PR, created [konveyor-ecosystem/analyzer-mcp](https://github.com/konveyor-ecosystem/analyzer-mcp) repo
- 2026-04-16: Upstream kai-analyzer exports merged ([konveyor/kai#926](https://github.com/konveyor/kai/pull/926))
- 2026-04-16: Initial implementation PR ([konveyor-ecosystem/analyzer-mcp#1](https://github.com/konveyor-ecosystem/analyzer-mcp/pull/1)) - 9 tools, 59 tests, stdio + HTTP transport
- 2026-04-16: Koncur MCP/RPC target integration ([konveyor/koncur#63](https://github.com/konveyor/koncur/pull/63))
- 2026-04-16: CI e2e infrastructure ([konveyor/ci#207](https://github.com/konveyor/ci/pull/207))

## Drawbacks

1. **Maintenance Burden**: Additional component to maintain alongside kantra and IDE extensions. Mitigated by delegation to kai-analyzer (minimal wrapper code) and comprehensive test coverage.

2. **Dependency on External Protocol**: Tied to MCP specification evolution. Mitigated by MCP's growing ecosystem and stable SDK.

## Alternatives

### Alternative 1: MCP server inside analyzer-lsp

The initial approach ([analyzer-lsp#1146](https://github.com/konveyor/analyzer-lsp/pull/1146)). Rejected because circular Go module dependencies between analyzer-lsp and kai-analyzer forced reimplementation of caching and analysis orchestration.

### Alternative 2: Stdio-Only (No HTTP)

Only implement stdio transport, skip HTTP server.

**Pros:** Simpler, no authentication concerns, smaller security surface.
**Cons:** No remote access for CI/CD, can't deploy as a service.

## Infrastructure Needed

- [x] New repository: [konveyor-ecosystem/analyzer-mcp](https://github.com/konveyor-ecosystem/analyzer-mcp)
- [ ] Container image: `quay.io/konveyor/analyzer-mcp`
- [ ] CI integration: Added to [konveyor/ci](https://github.com/konveyor/ci) image build matrix and koncur e2e tests
