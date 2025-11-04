---
title: distributed-language-extensions
authors:
  - "@djzager"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2025-01-10
last-updated: 2025-01-14
status: implementable 
see-also:
  - "https://github.com/konveyor/editor-extensions"
  - "https://github.com/konveyor/kai"
  - "https://github.com/konveyor/analyzer-lsp"
replaces: []
superseded-by: []
---

# Distributed Language Extensions Architecture

Decompose the Konveyor VSCode extension into a core orchestrator extension and
language-specific extensions (Java, etc.) where each language extension
spawns an external provider subprocess and manages communication pipes between
kai-analyzer-rpc, the provider, and the language server.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. **Provider startup timing**: Should java-external-provider start eagerly (at extension activation) or lazily (when first analysis is requested)?
  Answer: Eagerly. We will need it to be available so the konveyor-java-extension can use it to fulfill requests from konveyor-extension related to configuration, rulesets, etc.
2. **Error handling**: What happens if java-external-provider crashes mid-analysis? How do we recover?
  Answer: Stop analysis, alert with pointers to relevant logs, and option to retry.
3. **Binary distribution**: What's the best way to package and distribute java-external-provider binary with the VSCode extension?
  Answer: We pull it from the analyzer-lsp GitHub Releases like we do other assets.

## Summary

This enhancement extracts language-specific provider functionality from
kai-analyzer-rpc into separate VSCode extensions. The key architectural change
establishes **clear communication boundaries** between components:

1. **konveyor-core ↔ kai-analyzer-rpc**: Orchestration (start/stop analysis, get results)
2. **konveyor-core ↔ konveyor-java**: Ruleset metadata, provider configuration, source/target info
3. **konveyor-java ↔ java-external-provider**: LSP proxy requests to JDTLS
4. **kai-analyzer-rpc ↔ java-external-provider**: Provider evaluation calls during rule execution

Communication uses gRPC over named pipes/UDS for inter-process communication, with specific implementation details (number of pipes, API vs pipe for extension metadata) determined during implementation for optimal performance.

This allows:
- **Single JDTLS instance**: Eliminates duplicate JDTLS processes by sharing Red Hat Java extension's JDTLS
- **Smaller kai-analyzer-rpc**: Removes compiled-in language provider code
- **Multi-language workspaces**: Enables analyzing Java + Javascript projects together
- **Independent versioning**: Language extensions can be updated without touching kai-analyzer-rpc
- **Faster iteration**: Easier to experiment with new language support without rebuilding the entire stack

## Motivation

### Current State Problems

**Duplicate JDTLS Instances**:
- Red Hat Java extension runs JDTLS (loaded with konveyor analyzer bundles via `javaExtensions`)
- kai-analyzer-rpc's compiled-in java-external-provider spawns ANOTHER JDTLS instance
- Result: Two JDTLS processes consuming memory, potentially with different state

**Monolithic kai-analyzer-rpc**:
- java-external-provider code is compiled into kai-analyzer-rpc binary
- Adding Javascript support requires recompiling kai-analyzer-rpc
- Cannot analyze multi-language workspaces (Java + Python together)

**Tight coupling**:
- Updating Java provider functionality requires rebuilding entire kai-analyzer-rpc
- Language-specific changes (e.g., Java version compatibility) affect all users

### Goals

1. **Eliminate duplicate JDTLS**: Reuse the JDTLS instance managed by Red Hat Java extension
2. **Modular providers**: Extract language support into separate VSCode extensions
3. **Multi-language analysis**: Enable analyzing Java + Python workspaces simultaneously
4. **Smaller kai-analyzer-rpc binary**: Remove compiled-in language provider code
5. **Independent versioning**: Allow language extensions to evolve independently

### Non-Goals

1. **Custom language server implementation**: Still depend on existing language servers (JDTLS, Pyright)
2. **Dynamic provider loading**: Providers register at extension activation, not at runtime
4. **Breaking changes to analysis rules**: Rule syntax and semantics remain unchanged

## Proposal

### Distributed Architecture with Clear Communication Boundaries

```
┌──────────────────────────────────────────────────────────────────┐
│  VSCode Extension Host Process                                   │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ konveyor-core Extension                                    │  │
│  │                                                            │  │
│  │ 1. Creates main orchestration pipe:                        │  │
│  │    /tmp/main-orchestration.sock                            │  │
│  │                                                            │  │
│  │ 2. Collects provider pipes from language extensions        │  │
│  │                                                            │  │
│  │ 3. Spawns kai-analyzer-rpc with TWO pipes:                 │  │
│  │    kai-rpc \                                               │  │
│  │      -pipePath /tmp/main-orchestration.sock \              │  │
│  │      -java-pipe /tmp/java-provider.sock                    │  │
│  │                                                            │  │
│  │ 4. Connects to main pipe as client                         │  │
│  │    - Sends: analysis_engine.Analyze                        │  │
│  └────────────────────────────────────────────────────────────┘  │
│                 ↕ Main Pipe                                      │
│           (orchestration commands)                               │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ konveyor-java Extension                                    │  │
│  │                                                            │  │
│  │ 1. Creates ONE bidirectional named pipe:                   │  │
│  │    /tmp/java-provider.sock                                 │  │
│  │                                                            │  │
│  │ 2. Spawns java-external-provider subprocess:               │  │
│  │    java-external-provider \                                │  │
│  │      -provider-pipe /tmp/java-provider.sock                │  │
│  │                                                            │  │
│  │ 3. Handles bidirectional gRPC communication over pipe:     │  │
│  │    a) Receives LSP proxy requests from                     │  │
│  │       java-external-provider, forwards to JDTLS:           │  │
│  │       vscode.commands.executeCommand(                      │  │
│  │         "java.execute.workspaceCommand",                   │  │
│  │         "workspace/getAllSymbols", ...)                    │  │
│  │    b) (Future) Can send notifications to provider          │  │
│  │                                                            │  │
│  │ 4. Registers with core via Extension API (not a pipe):     │  │
│  │    coreApi.registerProvider({                              │  │
│  │      name: "java",                                         │  │
│  │      providerConfig: provider.Config{...},                 │  │
│  │      getBundleMetadata: () => ({sources, targets}),        │  │
│  │      supportsFileExtensions: [".java", ".jar"]             │  │
│  │    })                                                      │  │
│  └────────────────────────────────────────────────────────────┘  │
│              ↕ Provider Pipe (bidirectional gRPC over UDS)       │
│         (provider.Evaluate calls + LSP proxy requests)           │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ java-external-provider subprocess                          │  │
│  │                                                            │  │
│  │ - Connects to /tmp/java-provider.sock (bidirectional)      │  │
│  │                                                            │  │
│  │ - SERVER role: Receives provider.Evaluate() from           │  │
│  │   kai-analyzer-rpc via gRPC                                │  │
│  │                                                            │  │
│  │ - CLIENT role: Sends LSP proxy requests to                 │  │
│  │   konveyor-java extension over same pipe:                  │  │
│  │   • workspace/executeCommand("io.konveyor.tackle...")      │  │
│  │   • workspace/getAllSymbols                                │  │
│  │   • textDocument/references                                │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
                 ↕ Main Pipe        ↕ Provider Pipe
┌──────────────────────────────────────────────────────────────────┐
│  kai-analyzer-rpc Process                                        │
│                                                                  │
│  Main Pipe: Connected to konveyor-core                           │
│    - Receives: analysis_engine.Analyze                           │
│                                                                  │
│  Provider Pipe: Connected to java-external-provider              │
│    - Sends: provider.Evaluate(condition)                         │
│    - Receives: ProviderEvaluateResponse with incidents           │
└──────────────────────────────────────────────────────────────────┘
```

**Communication Channels**:

The architecture requires communication between the following components:

1. **konveyor-core ↔ kai-analyzer-rpc** (Orchestration)
   - Purpose: Start/stop analysis, get results
   - Messages: `analysis_engine.Analyze`, `analysis_engine.NotifyFileChanges`
   - Example: `/tmp/main-orchestration.sock`

2. **konveyor-core ↔ konveyor-java** (Metadata & Configuration)
   - Purpose: Provider configuration, ruleset metadata, source/target info
   - Implementation: VSCode extension API or gRPC (determined during implementation)
   - Data: Provider configs, bundle metadata, supported file extensions

3. **konveyor-java ↔ java-external-provider** (LSP Proxy)
   - Purpose: Forward LSP requests from provider to JDTLS
   - Messages: `workspace/executeCommand`, `workspace/getAllSymbols`, `textDocument/references`
   - Implementation: Potentially shared pipe with provider communication (determined during implementation)

4. **kai-analyzer-rpc ↔ java-external-provider** (Provider Evaluation)
   - Purpose: Rule condition evaluation during analysis
   - Messages: `provider.Evaluate()`, `provider.GetDependencies()`
   - Example: `/tmp/java-provider.sock`

**Analysis Flow**:

1. User clicks "Run Analysis"
2. konveyor-core → kai-analyzer-rpc: `analysis_engine.Analyze({ label: "..." })`
3. kai-analyzer-rpc runs rules engine, encounters Java rule condition
4. kai-analyzer-rpc → java-external-provider: `provider.Evaluate({ condition: "java.referenced({pattern: '...'}" })`
5. java-external-provider → konveyor-java extension: LSP proxy request
```javascript
   workspace/executeCommand({
     command: "io.konveyor.tackle.ruleEntry",
     arguments: { query: "...", location: "...", ... }
   })
```
6. konveyor-java extension → JDTLS (via VSCode API):
```javascript
   vscode.commands.executeCommand(
     "java.execute.workspaceCommand",
     "io.konveyor.tackle.ruleEntry",
     arguments
   )
```
7. Results flow back: `JDTLS → konveyor-java → java-external-provider → kai-analyzer-rpc`

### Implementation Details/Notes/Constraints

#### konveyor-java Extension Implementation

The konveyor-java extension is responsible for:
1. Spawning and managing the java-external-provider subprocess
2. Providing LSP proxy functionality to forward requests from java-external-provider to JDTLS
3. Registering with konveyor-core to provide metadata and configuration

```typescript
// vscode/java/src/extension.ts

export async function activate(context: vscode.ExtensionContext) {
  // Get core extension API
  const coreExtension = vscode.extensions.getExtension("konveyor.konveyor");
  await coreExtension.activate();
  const coreApi = coreExtension.exports as KonveyorCoreApi;

  // 1. Create pipe(s) for provider communication
  const providerPipe = rpc.generateRandomPipeName();

  // 2. Spawn java-external-provider subprocess
  const externalProvider = new JavaExternalProvider(
    context.extensionPath,
    providerPipe
  );

  // 3. Set up LSP proxy handler - forwards LSP requests to JDTLS
  externalProvider.onLspProxyRequest("workspace/executeCommand", async (params) => {
    return vscode.commands.executeCommand(
      "java.execute.workspaceCommand",
      params.command,
      params.arguments
    );
  });
  await externalProvider.start();

  // 4. Register with core - provide metadata and configuration
  // Implementation note: This could be VSCode extension API or gRPC
  const disposable = coreApi.registerProvider({
    name: "java",
    providerConfig: {
      name: "java",
      address: providerPipe,  // Where kai-analyzer-rpc connects
      contextLines: 10,
    },
    getBundleMetadata: () => ({
      sources: ["eap6", "eap7", "jakarta-ee"],
      targets: ["eap8", "quarkus", "springboot"],
    }),
    supportsFileExtensions: [".java", ".jar", ".war"],
    rulesetsPaths: [path.join(context.extensionPath, "rulesets")],
  });

  context.subscriptions.push(disposable, externalProvider);
}
```

#### kai-analyzer-rpc Changes

The kai-analyzer-rpc must be updated to:
1. Accept provider configurations instead of spawning providers itself
2. Connect to external provider processes via the provided addresses
3. Remove embedded provider code (java-external-provider, etc.)

```go
// kai_analyzer_rpc/main.go

func main() {
    pipePath := flag.String("pipePath", "", "Main orchestration pipe")
    providerConfigsJson := flag.String("provider-configs", "", "JSON array of provider configs")
    // ... other flags

    flag.Parse()

    // Parse provider configs - these come from language extensions
    var providerConfigs []provider.Config
    if *providerConfigsJson != "" {
        if err := json.Unmarshal([]byte(*providerConfigsJson), &providerConfigs); err != nil {
            log.Error(err, "failed to parse provider configs")
            panic(1)
        }
    }

    // Start analyzer - connect to external providers
    analyzerService := service.NewPipeAnalyzer(
        ctx,
        *pipePath,
        providerConfigs,  // Provider locations from extensions
        *rules,
        *sourceDirectory,
        l,
    )
}

// kai_analyzer_rpc/pkg/service/pipe_analyzer.go

func NewPipeAnalyzer(
    ctx context.Context,
    pipePath string,
    providerConfigs []provider.Config,
    rules, location string,
    l logr.Logger,
) (Analyzer, error) {

    providers := map[string]provider.InternalProviderClient{}

    // Connect to external providers using their configs
    for _, config := range providerConfigs {
        switch config.Name {
        case "java":
            jProvider, err := java.NewInternalProviderClientForPipe(
                ctx, l, config.ContextLines, location, config.Address,
            )
            if err != nil {
                return nil, err
            }
            providers["java"] = jProvider
        // Additional providers registered by their extensions
        }
    }
    // NO fallback to spawning providers - they come from extensions

    // ... rest of initialization
}
```

#### java-external-provider Changes

The java-external-provider needs to:
1. Be spawned by konveyor-java extension (not by kai-analyzer-rpc)
2. Handle provider.Evaluate() calls from kai-analyzer-rpc
3. Send LSP proxy requests to konveyor-java extension for JDTLS access

```go
// analyzer-lsp/external-providers/java-external-provider/main.go

var (
    providerPipe  = flag.String("provider-pipe", "", "Pipe for provider communication")
    logLevel      = flag.Int("log-level", 5, "Log level")
)

func main() {
    flag.Parse()

    if *providerPipe == "" {
        log.Error(fmt.Errorf("provider-pipe required"), "must specify -provider-pipe")
        panic(1)
    }

    // Create Java provider
    // Implementation details for LSP proxy communication determined during implementation
    client := java.NewJavaProvider(log, "java", contextLines, provider.Config{
        Address: *providerPipe,
    })

    // Start provider server
    // - Receives provider.Evaluate() from kai-analyzer-rpc
    // - Sends LSP proxy requests to konveyor-java extension for JDTLS access
    s := provider.NewPipeServer(client, *providerPipe, log)
    ctx := context.TODO()
    s.Start(ctx)
}
```

### Security, Risks, and Mitigations

**Risks**:
1. **Subprocess management complexity**: Multiple subprocesses to manage (java-external-provider, potentially python-external-provider)
   - *Mitigation*: Proper lifecycle management, cleanup on extension deactivation
2. **Pipe communication failures**: Pipes could disconnect, causing analysis failures
   - *Mitigation*: Implement retry logic, clear error messages, ability to restart providers
3. **Binary distribution**: java-external-provider binary must work across platforms (macOS, Linux, Windows)
   - *Mitigation*: Build matrix in CI/CD, package platform-specific binaries with extension

## Design Details

### Repository Structure

```
konveyor-editor-extensions/
├── agentic/           # Platform-agnostic
├── shared/            # Platform-agnostic
├── webview-ui/        # Platform-agnostic
├── tests/             # E2E tests
└── vscode/            # VSCode-specific code
    ├── lib/           # Shared utilities
    │   └── src/
    │       ├── pathUtils.ts
    │       └── fileUtils.ts
    │
    ├── core/          # Core extension (renamed from vscode/src)
    │   ├── package.json          # konveyor.konveyor
    │   ├── src/
    │   │   ├── api/              # NEW: Extension API exports
    │   │   │   ├── types.ts
    │   │   │   └── index.ts
    │   │   ├── client/
    │   │   │   └── analyzerClient.ts  # MODIFIED
    │   │   └── extension.ts      # MODIFIED: Export API
    │   └── tsconfig.json
    │
    └── java/          # NEW: Java language extension
        ├── package.json          # konveyor.konveyor-java
        │   ├── "activationEvents": ["onLanguage:java"]
        │   ├── "extensionDependencies": ["konveyor.konveyor", "redhat.java"]
        │   ├── "javaExtensions": ["./assets/jdtls-bundles/bundle.jar"]
        ├── src/
        │   ├── extension.ts          # Main activation logic
        │   ├── jdtlsProxyServer.ts   # LSP proxy server
        │   ├── javaExternalProvider.ts # Subprocess manager
        │   └── types.ts
        ├── rulesets/                 # Java-specific rulesets
        └── assets/
            ├── jdtls-bundles/        # Analyzer bundles for JDTLS
            └── java-external-provider # Binary from analyzer-lsp
```

## Drawbacks

1. **Increased complexity**: Multiple processes with IPC instead of single monolithic binary, more components to manage
2. **Debugging difficulty**: Harder to debug issues across multiple processes with inter-process communication
3. **Startup latency**: Spawning java-external-provider subprocess adds startup time

## Alternatives

### Alternative 1: gRPC Over TCP/IP Instead of gRPC Over UDS/Named Pipes

Use TCP/IP sockets instead of Unix Domain Sockets/Named Pipes for provider communication:

```typescript
// Instead of gRPC over named pipes/UDS, use gRPC over TCP/IP
const port = await findFreePort();
spawn("java-external-provider", [
  "-port", port.toString()
]);

coreApi.registerProvider({
  name: "java",
  providerConfig: {
    name: "java",
    address: `localhost:${port}`,  // TCP instead of UDS path
  },
  // ...
});
```

**Pros**:
- Better error handling and built-in retry logic with TCP
- Language-agnostic protocol
- Easier debugging with grpcurl
- Better suited for potential future remote provider scenarios

**Cons**:
- More complex setup (port management, potential TLS)
- Heavier dependency than UDS
- Named pipes/UDS already work well for local IPC
- Users would need permission to open ports on their machine
- Potential port conflicts with other applications

### Alternative 2: Keep Providers Compiled in kai-analyzer-rpc

Don't extract providers, maintain current architecture:

**Pros**:
- Simpler: no subprocess management, no extra pipes
- Fewer failure modes
- Easier to debug (single process)

**Cons**:
- Continues duplicate JDTLS problem
- Cannot add new languages without recompiling kai-analyzer-rpc
- Large binary size
- Tight coupling between language support and core analyzer

### Alternative 3: Core Extension Proxies Provider Communication

Instead of kai-analyzer-rpc connecting directly to providers, have core extension proxy all provider communication:

```
kai-rpc → core extension → java extension → java-external-provider
```

**Pros**:
- Core extension has visibility into all provider traffic
- Centralized control over provider communication
- Potentially simpler IPC management

**Cons**:
- Core extension becomes bottleneck for all provider queries
- More coupling between core and providers
- Extra hop adds latency to every provider.Evaluate() call
- Core must understand provider protocol to proxy it

### Alternative 4: VSCode Extension Implements Provider Protocol Directly

Skip java-external-provider subprocess entirely, implement provider protocol in TypeScript:

**Pros**:
- No subprocess management
- Fewer pipes
- All code in VSCode extension

**Cons**:
- Must reimplement all java-external-provider logic in TypeScript
- Duplicates code from analyzer-lsp
- Harder to maintain (logic exists in two places)

### Alternative 5: Use C Bindings to Embed Go Code in TypeScript

Use C bindings (cgo) to compile java-external-provider as a native module that can be loaded directly into the VSCode extension process:

**Pros**:
- No subprocess management needed
- No IPC overhead
- Single process for debugging

**Cons**:
- Complex build process requiring C toolchain
- Platform-specific binaries (different builds for macOS, Linux, Windows, ARM)
- Harder to isolate crashes (Go crash could crash entire extension)
- Node.js native module compatibility issues
- Significant development and maintenance overhead
- Violates VSCode extension sandboxing principles
