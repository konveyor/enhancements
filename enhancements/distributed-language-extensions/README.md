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
2. **Error handling**: What happens if java-external-provider crashes mid-analysis? How do we recover?
3. **Binary distribution**: What's the best way to package and distribute java-external-provider binary with the VSCode extension?

## Summary

This enhancement extracts language-specific provider functionality from
kai-analyzer-rpc into separate VSCode extensions. The key innovation is a
**three-pipe architecture**:

1. **Main Orchestration Pipe**: konveyor-core extension ↔ kai-analyzer-rpc for analysis commands
2. **Provider Pipe**: kai-analyzer-rpc ↔ java-external-provider subprocess for provider.Evaluate() calls
3. **JDTLS Pipe**: java-external-provider ↔ konveyor-java extension for LSP queries to JDTLS

This allows:
- **Single JDTLS instance**: Eliminates duplicate JDTLS processes by sharing Red Hat Java extension's JDTLS
- **Smaller kai-analyzer-rpc**: Removes compiled-in language provider code
- **Multi-language workspaces**: Enables analyzing Java + Javascript projects together
- **Independent versioning**: Language extensions can be updated without touching kai-analyzer-rpc

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
3. **GRPC migration**: This enhancement uses named pipes (though GRPC is a valid alternative)
4. **Breaking changes to analysis rules**: Rule syntax and semantics remain unchanged

## Proposal

### Three Named Pipes Architecture

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
│  │ 1. Creates TWO named pipes:                                │  │
│  │    a) /tmp/java-provider.sock (for kai-analyzer-rpc)       │  │
│  │    b) /tmp/java-jdtls.sock (for java-external-provider)    │  │
│  │                                                            │  │
│  │ 2. Spawns java-external-provider subprocess:               │  │
│  │    java-external-provider \                                │  │
│  │      -pipe /tmp/java-jdtls.sock \                          │  │
│  │      -provider-pipe /tmp/java-provider.sock                │  │
│  │                                                            │  │
│  │ 3. Listens on /tmp/java-jdtls.sock as SERVER:              │  │
│  │    - Handles LSP requests from java-external-provider      │  │
│  │    - Forwards to JDTLS via:                                │  │
│  │      vscode.commands.executeCommand(                       │  │
│  │        "java.execute.workspaceCommand", ...)               │  │
│  │                                                            │  │
│  │ 4. Registers with core:                                    │  │
│  │    coreApi.registerProvider({                              │  │
│  │      providerPipePath: "/tmp/java-provider.sock"           │  │
│  │    })                                                      │  │
│  └────────────────────────────────────────────────────────────┘  │
│         ↕ JDTLS Pipe                  ↕ Provider Pipe            │
│    (LSP messages)              (provider.Evaluate calls)         │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ java-external-provider subprocess                          │  │
│  │                                                            │  │
│  │ - Listens on /tmp/java-provider.sock as SERVER             │  │
│  │ - Handles provider.Evaluate() from kai-analyzer-rpc        │  │
│  │                                                            │  │
│  │ - Connects to /tmp/java-jdtls.sock as CLIENT               │  │
│  │ - Sends LSP messages to konveyor-java extension:           │  │
│  │   • initialize                                             │  │
│  │   • workspace/executeCommand("io.konveyor.tackle...")      │  │
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

**The Three Pipes**:

1. **Main Orchestration Pipe** (`/tmp/main-orchestration.sock`)
   - konveyor-core extension (client) ↔ kai-analyzer-rpc (server)
   - Purpose: Start/stop analysis, get results
   - Messages: `analysis_engine.Analyze`, `analysis_engine.NotifyFileChanges`

2. **Provider Pipe** (`/tmp/java-provider.sock`)
   - kai-analyzer-rpc (client) ↔ java-external-provider (server)
   - Purpose: Provider queries during rule evaluation
   - Messages: `provider.Evaluate()`, `provider.GetDependencies()`

3. **JDTLS Pipe** (`/tmp/java-jdtls.sock`)
   - java-external-provider (client) ↔ konveyor-java extension (server)
   - Purpose: LSP queries to JDTLS
   - Messages: `initialize`, `workspace/executeCommand`, `textDocument/references`, `shutdown`

**Analysis Flow**:

1. User clicks "Run Analysis"
2. konveyor-core → kai-analyzer-rpc (main pipe): `analysis_engine.Analyze({ label: "..." })`
3. kai-analyzer-rpc runs rules engine - Encounters Java rule condition
4. kai-analyzer-rpc → java-external-provider (provider pipe): `provider.Evaluate({ condition: "java.referenced({pattern: '...'}" })`
5. java-external-provider → konveyor-java extension (JDTLS pipe)
```javascript
   workspace/executeCommand({
     command: "io.konveyor.tackle.ruleEntry",
     arguments: { query: "...", location: "...", ... }
   })
```
6. konveyor-java extension → JDTLS
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

```typescript
// vscode/java/src/extension.ts

export async function activate(context: vscode.ExtensionContext) {
  // Get core extension API
  const coreExtension = vscode.extensions.getExtension("konveyor.konveyor");
  await coreExtension.activate();
  const coreApi = coreExtension.exports as KonveyorCoreApi;

  // 1. Create TWO named pipes
  const providerPipe = rpc.generateRandomPipeName();  // For kai-rpc connection
  const jdtlsPipe = rpc.generateRandomPipeName();     // For JDTLS access

  // 2. Start JDTLS proxy server (handles LSP messages from java-external-provider)
  const jdtlsProxy = new JdtlsProxyServer(jdtlsPipe, context);
  jdtlsProxy.onRequest("workspace/executeCommand", async (params) => {
    return vscode.commands.executeCommand(
      "java.execute.workspaceCommand",
      params.command,
      params.arguments
    );
  });
  await jdtlsProxy.start();

  // 3. Spawn java-external-provider subprocess
  const externalProvider = new JavaExternalProvider(
    context.extensionPath,
    providerPipe,  // Where kai-analyzer-rpc connects
    jdtlsPipe      // Where provider connects for JDTLS access
  );
  await externalProvider.start();

  // 4. Register provider pipe with core
  const disposable = coreApi.registerProvider({
    languageId: "java",
    providerPipePath: providerPipe,
    supportsFileExtensions: [".java", ".jar", ".war"],
    rulesetsPaths: [path.join(context.extensionPath, "rulesets")]
  });

  context.subscriptions.push(disposable, jdtlsProxy, externalProvider);
}
```

#### kai-analyzer-rpc Changes

```go
// kai_analyzer_rpc/main.go

func main() {
    pipePath := flag.String("pipePath", "", "Main orchestration pipe")
    javaPipe := flag.String("java-pipe", "", "Java provider pipe")
    pythonPipe := flag.String("python-pipe", "", "Python provider pipe")
    // ... other flags

    flag.Parse()

    // Build provider pipe map
    providerPipes := make(map[string]string)
    if *javaPipe != "" {
        providerPipes["java"] = *javaPipe
    }
    if *pythonPipe != "" {
        providerPipes["python"] = *pythonPipe
    }

    // Start analyzer with provider pipes
    analyzerService := service.NewPipeAnalyzer(
        ctx,
        *pipePath,
        providerPipes,  // NEW: Map of provider pipes
        *rules,
        *sourceDirectory,
        l,
    )
}

// kai_analyzer_rpc/pkg/service/pipe_analyzer.go

func NewPipeAnalyzer(
    ctx context.Context,
    pipePath string,
    providerPipes map[string]string,  // NEW
    rules, location string,
    l logr.Logger,
) (Analyzer, error) {

    providers := map[string]provider.InternalProviderClient{}

    // If Java pipe provided, use external provider
    if javaPipePath, ok := providerPipes["java"]; ok {
        jProvider, err := java.NewInternalProviderClientForPipe(
            ctx, l, contextLines, location, javaPipePath,
        )
        if err != nil {
            return nil, err
        }
        providers["java"] = jProvider
    }
    // NO fallback to spawning JDTLS

    // ... rest of initialization
}
```

#### java-external-provider Changes

```go
// analyzer-lsp/external-providers/java-external-provider/main.go

var (
    pipe          = flag.String("pipe", "", "Pipe to JDTLS (for LSP communication)")
    providerPipe  = flag.String("provider-pipe", "", "Pipe for provider communication")
    logLevel      = flag.Int("log-level", 5, "Log level")
)

func main() {
    flag.Parse()

    // Validate flags
    if *pipe == "" {
        log.Error(fmt.Errorf("pipe required"), "must specify -pipe for JDTLS")
        panic(1)
    }
    if *providerPipe == "" {
        log.Error(fmt.Errorf("provider-pipe required"), "must specify -provider-pipe")
        panic(1)
    }

    // Create Java provider (connects to JDTLS via -pipe)
    client := java.NewJavaProvider(log, "java", contextLines, provider.Config{})

    // Start provider server on provider-pipe (listens for kai-rpc)
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

1. **Increased complexity**: Three pipes instead of one, multiple subprocesses to manage
2. **Debugging difficulty**: Harder to debug issues across multiple processes and pipes
3. **Startup latency**: Spawning java-external-provider subprocess adds startup time
4. **Binary distribution**: Must package and distribute java-external-provider binary with extension (increases extension size)
5. **Cross-repo coordination**: Requires synchronized changes across analyzer-lsp, kai, and editor-extensions repositories
6. **Potential for version skew**: konveyor-java extension version must match compatible java-external-provider version

## Alternatives

### Alternative 1: GRPC Instead of Named Pipes

Use GRPC for provider communication (java-external-provider already supports GRPC mode):

```typescript
// Instead of named pipes, use GRPC
const port = await findFreePort();
spawn("java-external-provider", [
  "-port", port.toString(),
  "-pipe", jdtlsPipe
]);

coreApi.registerProvider({
  languageId: "java",
  grpcAddress: `localhost:${port}`
});
```

**Pros**:
- Better error handling, built-in retry logic
- Language-agnostic protocol
- Easier debugging with grpcurl
- Better suited for potential future remote provider scenarios

**Cons**:
- More complex setup (port management, potential TLS)
- Heavier dependency
- Named pipes already work well for local IPC

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

### Alternative 3: Single Provider Pipe with Core Proxying

Instead of kai-analyzer-rpc connecting directly to provider pipes, have core extension proxy:

```
kai-rpc → (main pipe) → core extension → (delegates) → java extension → java-external-provider
```

**Pros**:
- Only two pipes instead of three
- Core extension has visibility into all provider traffic

**Cons**:
- Core extension becomes bottleneck
- More coupling between core and providers
- Extra hop adds latency

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
