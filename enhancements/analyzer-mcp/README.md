---
title: mcp-server-integration
authors:
  - "@JonahSussman"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-01-23
last-updated: 2026-01-23
status: implementable
see-also:
  - "https://modelcontextprotocol.io/"
  - "https://github.com/modelcontextprotocol/specification"
replaces:
  - N/A
superseded-by:
  - N/A
---

# MCP Server Integration

Add a Model Context Protocol (MCP) server to analyzer-lsp to expose analysis capabilities to AI assistants and other MCP clients, enabling programmatic access to rule analysis, dependency extraction, and code inspection features.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

1. Should we support additional transports beyond stdio and HTTP (e.g., WebSocket)?
2. What is the long-term strategy for versioning the MCP tools as the analyzer-lsp API evolves?
3. Should we implement caching for frequently-run analyses to improve performance?

## Summary

This enhancement adds a Model Context Protocol (MCP) server implementation to analyzer-lsp, exposing six core analysis tools through a standardized protocol. The MCP server enables AI assistants (like Claude Desktop), IDEs, and other automation tools to programmatically analyze codebases, validate rules, extract dependencies, and query analysis results.

The implementation supports both stdio transport (for local CLI usage and IDE integration) and HTTP transport (for remote access and CI/CD integration), with OAuth 2.1 authentication for the HTTP endpoint. This unlocks new use cases including AI-assisted code migration, automated rule development, and integrated analysis workflows.

Key capabilities exposed through MCP:
- **analyze_run**: Execute full analysis on codebases with configurable rules and filters
- **rules_list**: Enumerate available rules with metadata
- **rules_validate**: Validate rule syntax and structure
- **dependencies_get**: Extract dependency trees from projects
- **providers_list**: List available analysis providers and their capabilities
- **analyze_incidents**: Query and filter incidents from analysis results

## Motivation

Current analyzer-lsp usage requires direct CLI invocation or LSP server integration, limiting automation and AI-assisted workflows. Users who want to integrate analysis into custom tooling, CI/CD pipelines, or AI agents must parse command-line output or interact with the LSP protocol, which is designed for IDEs rather than programmatic access.

The Model Context Protocol provides a standardized way for AI assistants and automation tools to interact with external services. By implementing an MCP server, we enable:

1. **AI-Assisted Migration**: AI assistants can run analyses, interpret results, and suggest code fixes
2. **Custom Workflows**: Developers can build automation tools that leverage analyzer-lsp capabilities
3. **CI/CD Integration**: Remote HTTP access enables analysis in containerized and distributed environments
4. **Interactive Exploration**: Users can query rules, validate syntax, and explore dependencies conversationally
5. **Cross-Platform Access**: Standard protocol works across different tools and platforms

### Goals

- Expose core analyzer-lsp functionality through MCP protocol
- Support both local (stdio) and remote (HTTP) access patterns
- Maintain security through OAuth 2.1 authentication for HTTP
- Provide clear documentation for integration with Claude Desktop and other MCP clients
- Maintain API compatibility with existing analyzer-lsp commands
- Enable seamless integration with AI assistants and automation tools

### Non-Goals

- Replace the existing CLI interface (MCP is complementary)
- Replace the LSP server for IDE integration
- Implement real-time streaming of analysis progress
- Support WebSocket or other transports in initial implementation
- Provide a web UI for the MCP server
- Cache or persist analysis results (stateless operation)
- Modify core analyzer-lsp analysis engine behavior

## Proposal

Implement an MCP server as a new command (`cmd/analyzer/mcp/`) that wraps existing analyzer-lsp functionality and exposes it through the Model Context Protocol. The server will support two transports: stdio for local usage and HTTP for remote access.

### User Stories [optional]

#### Story 1: AI-Assisted Code Migration

As a developer migrating a Java EE application to Quarkus, I want to use Claude Desktop to analyze my codebase, explain migration issues, and suggest fixes.

1. Configure Claude Desktop with the analyzer-lsp MCP server
2. Ask Claude: "Analyze my project for Java EE to Quarkus migration issues"
3. Claude uses `rules_list` to find relevant rules, then `analyze_run` to execute analysis
4. Claude explains the incidents found and suggests code changes
5. I can ask follow-up questions like "Show me all logging framework issues"
6. Claude uses `analyze_incidents` to filter and explain specific issues

#### Story 2: Custom Rule Development Workflow

As a rule author, I want to validate my custom rules interactively before committing them.

1. Write a new rule in my editor
2. Use an MCP client to call `rules_validate` on my rule file
3. Get immediate feedback on syntax errors or missing fields
4. Use `analyze_run` with `incident_limit=10` to test the rule on a sample project
5. Iterate on the rule based on results without leaving my workflow

#### Story 3: CI/CD Integration

As a DevOps engineer, I want to run analysis on pull requests in our CI pipeline.

1. Deploy MCP server with HTTP transport in our Kubernetes cluster
2. Configure OAuth 2.1 authentication with our identity provider
3. CI job calls HTTP endpoint with `analyze_run` on the PR branch
4. Parse JSON results and post comments on PRs with migration issues
5. Block merges if critical issues are found

#### Story 4: Dependency Analysis Automation

As a security engineer, I want to track open-source dependencies across multiple projects.

1. Script calls MCP server with `dependencies_get` for each project
2. Aggregate dependency data into a central database
3. Cross-reference with vulnerability databases
4. Generate reports on dependency usage and risk

### Implementation Details/Notes/Constraints [optional]

**Architecture:**
```
cmd/analyzer/mcp/
├── main.go              # CLI entry point with transport selection
├── server.go            # MCP server wrapper and tool registration
├── analyze.go           # analyze_run tool implementation
├── rules.go             # rules_list and rules_validate implementations
├── dependencies.go      # dependencies_get implementation
├── providers.go         # providers_list implementation
├── incidents.go         # analyze_incidents implementation
└── transport_http.go    # HTTP server with OAuth 2.1
```

**Key Design Decisions:**

1. **Reuse Existing Code**: All tools delegate to existing analyzer-lsp libraries (`parser`, `engine`, `provider`, etc.) rather than reimplementing logic
2. **Stateless Operation**: Each tool invocation is independent; no session state is maintained
3. **Error Mapping**: analyzer-lsp errors are mapped to appropriate MCP error codes (-32602 for invalid params, -32603 for internal errors)
4. **Logging**: All logs go to stderr to keep stdout clean for MCP protocol (stdio transport)
5. **Provider Lifecycle**: Providers are initialized per-request and cleaned up immediately to avoid resource leaks

**Tool Parameter Design:**

All tools accept JSON parameters validated against schemas. Common patterns:
- Path parameters are validated for existence before processing
- Optional parameters have sensible defaults (e.g., `incident_limit: 1500`)
- Label selectors use analyzer-lsp expression syntax
- Output formats are limited to `json` and `yaml`

**HTTP Transport Security:**

- OAuth 2.1 token validation on all MCP endpoints
- CORS headers configurable (default: allow all origins for development)
- Health endpoint (`/health`) is unauthenticated for load balancers
- TLS termination expected at reverse proxy layer

### Security, Risks, and Mitigations

**Security Considerations:**

1. **Path Traversal Risk**
   - Risk: Malicious clients could provide paths like `../../../etc/passwd`
   - Mitigation: Validate all paths exist and reject suspicious patterns

2. **Resource Exhaustion**
   - Risk: Large codebases or complex rules could consume excessive memory/CPU
   - Mitigation: Respect existing analyzer-lsp limits (`incident_limit`), HTTP request timeouts

3. **Authentication Bypass**
   - Risk: HTTP transport could be accessed without valid tokens
   - Mitigation: OAuth 2.1 token validation on all protected endpoints

4. **Information Disclosure**
   - Risk: Error messages could leak sensitive file paths or code content
   - Mitigation: Generic error messages, detailed errors only in debug mode

5. **Dependency Vulnerabilities**
   - Risk: MCP SDK or OAuth libraries could have vulnerabilities
   - Mitigation: Regular dependency updates, security scanning in CI

**Deployment Risks:**

1. **Performance Impact**: MCP server adds overhead to each analysis
   - Mitigation: Minimal wrapper code, direct delegation to existing functions

2. **API Stability**: MCP protocol or analyzer-lsp internals could change
   - Mitigation: Pin MCP SDK version, maintain compatibility layer
   - Versioning strategy documented in tool descriptions

3. **Multi-Tenancy**: HTTP deployment requires isolation between requests
   - Mitigation: Stateless design, provider cleanup after each request
   - Each request gets fresh provider instances

## Design Details

### Test Plan

**Unit Tests:**
- Tool parameter validation (all required/optional fields)
- Error handling for invalid inputs (non-existent paths, invalid formats)
- Output format validation (JSON and YAML)
- Helper function coverage (label parsing, category extraction)
- Provider lifecycle management

**Integration Tests:**
- Full HTTP server lifecycle (startup, request handling, graceful shutdown)
- OAuth authentication flow (valid tokens, invalid tokens, missing tokens)
- CORS header validation
- All 6 tools end-to-end with real fixtures
- Concurrent request handling
- Context cancellation and timeout behavior

**Test Fixtures:**
- Sample rules (`testdata/rules/test_rules.yaml`)
- Invalid rules for error testing
- Sample Java/Go projects for analysis
- Provider configuration files
- Analysis result files for incident querying

**Load Testing (Future):**
- Concurrent requests to HTTP endpoint
- Large codebase analysis
- Memory and CPU profiling under load

### Upgrade / Downgrade Strategy

**Initial Release:**
- MCP server is a new optional component, no upgrade concerns
- Can be deployed alongside existing analyzer-lsp installations
- No changes to existing CLI or LSP functionality

**Future Version Compatibility:**
- Tool input schemas will be versioned in descriptions
- Breaking changes will result in new tool names (e.g., `analyze_run_v2`)
- Deprecated tools will remain available for 2+ releases
- HTTP API will use URL versioning if needed (`/v2/mcp`)

**Configuration Migration:**
- MCP server reads existing `provider_settings.json`
- No new configuration files required
- OAuth settings provided via environment variables or flags

## Implementation History

- 2026-01-23: Enhancement proposal created
- 2026-01-23: Initial implementation
- 2026-01-23: Initial tests suite created

## Drawbacks

1. **Maintenance Burden**: Additional API surface to maintain alongside CLI and LSP
   - Counterpoint: Minimal code due to delegation to existing libraries
   - Mitigation: Comprehensive test coverage catches regressions

2. **Security Surface**: HTTP transport introduces authentication and network security concerns
   - Counterpoint: OAuth 2.1 is industry standard, well-understood
   - Mitigation: Security review before HTTP deployment, TLS required in production

3. **Performance Overhead**: MCP protocol adds JSON serialization/deserialization
   - Counterpoint: Overhead is minimal (<5%) and acceptable for automation use cases
   - Mitigation: Direct CLI/LSP remains available for performance-critical usage

4. **Dependency on External Protocol**: Tied to MCP specification evolution
   - Counterpoint: MCP is backed by Anthropic and growing ecosystem
   - Mitigation: MCP SDK is stable, minimal breaking changes expected

## Alternatives

### Alternative 1: Stdio-Only (No HTTP)

Only implement stdio transport, skip HTTP server.

**Pros:**
- Simpler implementation
- No authentication concerns
- Smaller security surface

**Cons:**
- No remote access for CI/CD
- Can't deploy as a service
- Limited to local usage only

## Infrastructure Needed [optional]

**Development:**
- No new infrastructure required
- Existing analyzer-lsp repository and CI pipeline sufficient

**Testing:**
- Integration with Claude Desktop requires desktop environment (manual testing)
- HTTP integration tests run on existing CI infrastructure

**Production Deployment (for users):**
- Users deploying HTTP transport need:
  - Container orchestration (Kubernetes, Docker, etc.) - user-provided
  - OAuth 2.1 identity provider - user-provided
  - TLS termination (reverse proxy) - user-provided
  - Monitoring/observability - user-provided

**Documentation:**
- Usage examples in main README
- Detailed MCP server documentation in `cmd/analyzer/mcp/README.md`
- Claude Desktop integration guide
- HTTP deployment guide with security best practices
