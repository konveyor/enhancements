---
title: distributed-language-extensions
authors:
  - "@djzager"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2025-01-10
last-updated: 2025-01-10
status: provisional
see-also:
  - "https://github.com/konveyor/editor-extensions"
  - "https://github.com/konveyor/kai"
replaces: []
superseded-by: []
---

# Distributed Language Extensions Architecture

Decompose the Konveyor VSCode extension into a core orchestrator extension and
language-specific extensions (Java, etc.) where each language extension manages
its own analyzer provider lifecycle while the core extension maintains the
analyzer RPC client and coordinates analysis across languages.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. **Provider Registration API**: Should language extensions register providers synchronously during activation or asynchronously after validating language server availability?
2. ~**Activation Events**: Should language extensions use the same activation events as their dependent language servers (e.g., `onLanguage:java`) or custom Konveyor-specific events?~
3. ~**Asset Distribution**: Should rulesets be bundled per-language or remain centralized in core with language extensions contributing additional rulesets?~
4. ~**Version Compatibility**: How do we enforce compatible version ranges between core extension and language extensions (e.g., core 1.x only works with java extension 1.y-1.z)?~

## Summary

The current Konveyor extension tightly couples Java-specific functionality
(JDTLS integration, Java extension dependency, Java rulesets) with the core
analysis engine, making it difficult to support multiple programming languages
within a single workspace. This enhancement proposes splitting the extension
into:

1. **Core Extension** (`konveyor`): Provides the analyzer RPC client, webview UI, GenAI integration, orchestration API, and multi-language result aggregation
2. **Language Extensions** (`konveyor-java`, `konveyor-python`, etc.): Register language-specific analyzer providers, manage language server integration, bundle language-specific rulesets, and handle language-specific configuration

This architecture enables:
- **Multi-language workspace support**
- **Independent language evolution**
- **Smaller installation footprint**: Users only install language support they need
- **Language-specific configuration**: Each language extension owns its settings (JVM args, virtual env paths, etc.)
- **Crash isolation**: One analyzer extension's failure does not affect Java analysis

## Motivation

### Current State

The Konveyor extension currently hardcodes Java support:

**package.json**:
```json
{
  "extensionDependencies": ["redhat.java"],
  "javaExtensions": ["../downloaded_assets/jdtls-bundles/bundle.jar"]
}
```

**extension.ts**:
```typescript
private checkJavaExtensionInstalled(): void {
  const javaExt = vscode.extensions.getExtension("redhat.java");
  // Version checking: must be <= 1.45.0
}
```

**analyzerClient.ts**:
```typescript
this.analyzerRpcConnection.onRequest("workspace/executeCommand", async (params) => {
  return await vscode.commands.executeCommand(
    "java.execute.workspaceCommand",  // Hardcoded Java LSP integration
    params.command,
    params.arguments
  );
});
```

**Problems**:
1. Forces all users to install Red Hat Java extension even if analyzing non-Java code
2. Cannot analyze workspaces with multiple languages (Java + Python)
3. Java-specific crashes (JDTLS issues) can break entire extension
4. Adds 15MB of JDTLS bundles to every installation
5. Python/Go/JavaScript support requires forking or massive refactoring

### Goals

1. **Enable multi-language workspaces**: Analyze Java backend + Javascript frontend in the same project
2. **Modular language support**
3. **Language-agnostic core**: Core extension has zero knowledge of Java/Python/etc.
6. **Extensibility**: Third-party developers can create `konveyor-golang`, `konveyor-rust` extensions

### Non-Goals

1. **Language detection heuristics**: Extensions explicitly declare supported file extensions; no magic detection
2. **Backward compatibility**: This is a breaking change requiring users to install language-specific extensions
3. **Rule authoring in extensions**: Rulesets remain YAML files; extensions bundle them

## Proposal

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  konveyor (core extension)                                  │
│  ├── AnalyzerClient (manages kai-analyzer-rpc lifecycle)    │
│  ├── WebView UI (aggregates results from all languages)     │
│  ├── GenAI Integration (language-agnostic)                  │
│  ├── Solution Server Client                                 │
│  └── Orchestrator API (coordinates language extensions)     │
└─────────────────────────────────────────────────────────────┘
       ▲
       │ registers provider
       │
┌──────┴──────────────────┐
│ konveyor-java           │
│ ├── JavaProvider        │
│ ├── JDTLS Integration   │
│ ├── Java Rulesets       │
│ └── Java Config         │
└─────────────────────────┘
       │
       ▼ depends on
┌─────────────────────────┐
│ redhat.java <= 1.45.0   │
└─────────────────────────┘
```

### Key Architectural Decision: Core Owns Analyzer Client

The analyzer RPC client (`AnalyzerClient`) remains in the **core extension**, not duplicated in each language extension.

**Rationale**:
1. **Single Source of Truth**: Only one kai-analyzer-rpc process runs, configured with multiple providers
2. **Simplified IPC**: Language extensions don't manage RPC connections; they register providers via API
3. **Resource Efficiency**: One analyzer process vs. N processes (one per language)
4. **Coordinated Lifecycle**: Core extension controls when analysis starts/stops across all languages
5. **Result Aggregation**: Core receives results directly from analyzer, no inter-extension messaging

**How It Works**:

Core extension (konveyor):

```typescript
export class AnalyzerClient {
  private registeredProviders = new Map<string, ProviderConfig>();

  registerProvider(config: ProviderConfig): void {
    this.registeredProviders.set(config.languageId, config);
    // When analyzer starts, enable this provider
  }

  async start(): Promise<void> {
    // Start kai-analyzer-rpc with all registered providers
    const providerFlags = Array.from(this.registeredProviders.values())
      .map(p => `--provider=${p.name}`)
      .join(" ");

    // kai-analyzer-rpc --provider=java --provider=python
    this.analyzerProcess = spawn(kaiPath, providerFlags.split(" "));
  }
}
```

Language extension (konveyor-java):
```typescript
export async function activate(context: vscode.ExtensionContext) {
  const coreApi = await getCoreExtension();

  coreApi.registerProvider({
    languageId: "java",
    name: "java",  // Corresponds to kai provider name
    supportsFileExtensions: [".java", ".class"],
    executeCommand: async (cmd: string, args: any[]) => {
      // Delegate to Java LSP
      return await vscode.commands.executeCommand(
        "java.execute.workspaceCommand",
        cmd,
        ...args
      );
    },
    getAssetPaths: () => ({
      bundles: context.asAbsolutePath("./jdtls-bundles"),
      rulesets: context.asAbsolutePath("./rulesets"),
    }),
  });
}
```

### User Stories

#### Story 1: Full-Stack Developer (New Use Case)

**As a** full-stack developer with Java backend + Python scripts
**I want to** analyze both codebases for migration issues
**So that** I can modernize my entire application stack

**Flow**:
1. Install `konveyor`, `konveyor-java`, `konveyor-python`
2. Open workspace:
   ```
   project/
   ├── backend/  (Java Spring Boot)
   └── scripts/  (Python 2)
   ```
3. Create profile:
   ```json
   {
     "name": "Modernize Full Stack",
     "labelSelector": "(konveyor.io/source=java-ee || konveyor.io/source=python2) && (konveyor.io/target=quarkus || konveyor.io/target=python3)"
   }
   ```
4. Run analysis
5. Both providers activate in single analyzer process
6. Results merged in UI: "Found 42 Java issues, 15 Python issues"

### Implementation Details/Notes/Constraints

#### Core Extension API

```typescript
// konveyor/src/api/types.ts

export interface ProviderConfig {
  /** Unique identifier (e.g., "java", "python") */
  languageId: string;

  /** Provider name as recognized by kai-analyzer-rpc */
  name: string;

  /** File extensions this provider handles */
  supportsFileExtensions: string[];

  /** Execute language server commands (for kai RPC callbacks) */
  executeCommand(command: string, args: any[]): Promise<any>;

  /** Get paths to language-specific assets */
  getAssetPaths(): ProviderAssetPaths;

  /** Check if language server is ready */
  isReady(): Promise<boolean>;
}

export interface ProviderAssetPaths {
  /** Path to language-specific bundles (e.g., JDTLS jars) */
  bundles?: string;

  /** Path to language-specific rulesets */
  rulesets?: string;

  /** Additional provider-specific assets */
  [key: string]: string | undefined;
}

export interface KonveyorCoreApi {
  /** Register a language provider (called during language extension activation) */
  registerProvider(config: ProviderConfig): Disposable;

  /** Get currently registered providers */
  getProviders(): ProviderConfig[];

  /** Subscribe to analysis lifecycle events */
  onAnalysisStarted(listener: () => void): Disposable;
  onAnalysisCompleted(listener: (results: AnalysisResults) => void): Disposable;
  onAnalysisFailed(listener: (error: Error) => void): Disposable;
}
```

#### Analyzer Client Changes

**Current**: Hardcoded Java LSP integration

```typescript
// OLD: vscode/src/client/analyzerClient.ts
this.analyzerRpcConnection.onRequest(
  "workspace/executeCommand",
  async (params: WorkspaceCommandParams) => {
    return await vscode.commands.executeCommand(
      "java.execute.workspaceCommand",  // HARDCODED
      params.command,
      params.arguments![0],
    );
  }
);
```

**New**: Provider-based delegation

```typescript
// NEW: vscode/src/client/analyzerClient.ts
export class AnalyzerClient {
  private providers = new Map<string, ProviderConfig>();

  registerProvider(config: ProviderConfig): Disposable {
    this.providers.set(config.languageId, config);
    this.logger.info(`Registered ${config.languageId} provider`);

    return {
      dispose: () => {
        this.providers.delete(config.languageId);
      }
    };
  }

  private setupRpcHandlers(): void {
    this.analyzerRpcConnection.onRequest(
      "workspace/executeCommand",
      async (params: WorkspaceCommandParams) => {
        // Determine which provider should handle this command
        const provider = this.getProviderForCommand(params.command);

        if (!provider) {
          throw new Error(`No provider registered for command: ${params.command}`);
        }

        // Delegate to language-specific provider
        return await provider.executeCommand(params.command, params.arguments);
      }
    );
  }

  private getProviderForCommand(command: string): ProviderConfig | undefined {
    // Commands are prefixed with language (e.g., "java.project.getAll")
    const languageId = command.split(".")[0];
    return this.providers.get(languageId);
  }

  async start(): Promise<void> {
    // Build provider flags for kai-analyzer-rpc
    const providerArgs: string[] = [];
    const assetPaths: Record<string, any> = {};

    for (const [languageId, provider] of this.providers) {
      providerArgs.push(`--provider=${provider.name}`);

      const assets = provider.getAssetPaths();
      if (assets.bundles) {
        assetPaths[`${languageId}_bundles`] = assets.bundles;
      }
      if (assets.rulesets) {
        assetPaths[`${languageId}_rulesets`] = assets.rulesets;
      }
    }

    // Start kai-analyzer-rpc with dynamic provider configuration
    const args = [
      ...providerArgs,
      `--asset-paths=${JSON.stringify(assetPaths)}`,
      // ... other args
    ];

    this.analyzerProcess = spawn(this.assetPaths.kaiAnalyzer, args);
  }
}
```

#### Language Extension Structure

```
editor-extensions/
├── agentic/           # No change
├── shared/            # No change
├── webview-ui/        # No change
├── tests/             # No change
└── vscode/            # Reorganize internally
    ├── lib/           # NEW: Shared utilities package
    │   ├── package.json
    │   ├── src/
    │   │   ├── pathUtils.ts      # Moved from vscode/src/utilities
    │   │   ├── ideUtils.ts       # Moved from vscode/src/utilities
    │   │   ├── fileUtils.ts      # Moved from vscode/src/utilities
    │   │   ├── Extension.ts      # Moved from vscode/src/helpers
    │   │   └── index.ts          # Export barrel
    │   └── tsconfig.json
    │
    ├── core/          # RENAMED: Was the root of vscode/src
    │   ├── package.json          # This IS the konveyor extension
    │   ├── src/
    │   │   ├── api/              # Extension API
    │   │   ├── client/           # AnalyzerClient
    │   │   ├── utilities/        # Core-specific (config, auth, profiles)
    │   │   ├── modelProvider/
    │   │   ├── issueView/
    │   │   └── extension.ts
    │   └── tsconfig.json
    │
    └── java/          # NEW: Java language extension
        ├── package.json          # This IS konveyor-java extension
        │   ├── "activationEvents": ["onLanguage:java"]  ← Same as redhat.java
        │   ├── "extensionDependencies": ["konveyor.konveyor", "redhat.java"]
        │   └── "contributes":
        │       ├── "configuration": { konveyor.java.* settings }
        │       └── "commands": { konveyor.java.* commands }
        ├── src/
        │   ├── extension.ts          # Activation & provider registration
        │   ├── javaProvider.ts        # ProviderConfig implementation
        │   └── javaExtensionCheck.ts  # Version validation for redhat.java
        ├── assets/
        └── tsconfig.json
```

**vscode/java/src/extension.ts**:

```typescript
import * as vscode from "vscode";
import { KonveyorCoreApi } from "@konveyor/core-api";
import { JavaProvider } from "./javaProvider";
import { checkJavaExtensionVersion } from "./javaExtensionCheck";

export async function activate(context: vscode.ExtensionContext) {
  // Wait for core extension
  const coreExt = vscode.extensions.getExtension("konveyor.konveyor");
  if (!coreExt) {
    throw new Error("Konveyor core extension not found. Please install it first.");
  }

  const coreApi: KonveyorCoreApi = await coreExt.activate();

  const javaExt = vscode.extensions.getExtension("redhat.java");
  if (!javaExt) {
    vscode.window.showErrorMessage(
      "Red Hat Java extension is required for Konveyor Java support."
    );
    return;
  }

  // any reqs on the java extension
  checkJavaExtensionVersion(javaExt);

  // Wait for Java extension to activate
  await javaExt.activate();

  // Create and register Java provider
  const javaProvider = new JavaProvider(context);
  const registration = coreApi.registerProvider(javaProvider.getConfig());

  context.subscriptions.push(registration);
}
```

**konveyor-java/src/javaProvider.ts**:

```typescript
import * as vscode from "vscode";
import { ProviderConfig, ProviderAssetPaths } from "@konveyor/core-api";

export class JavaProvider {
  constructor(private context: vscode.ExtensionContext) {}

  getConfig(): ProviderConfig {
    return {
      languageId: "java",
      name: "java",
      supportsFileExtensions: [".java", ".class", ".jar"],

      executeCommand: async (command: string, args: any[]) => {
        // Delegate to Red Hat Java extension
        return await vscode.commands.executeCommand(
          "java.execute.workspaceCommand",
          command,
          ...args
        );
      },

      getAssetPaths: (): ProviderAssetPaths => ({
        bundles: this.context.asAbsolutePath("./assets/jdtls-bundles"),
        rulesets: this.context.asAbsolutePath("./rulesets"),
      }),

      isReady: async (): Promise<boolean> => {
        try {
          const projects = await vscode.commands.executeCommand(
            "java.execute.workspaceCommand",
            "java.project.getAll"
          );
          return Array.isArray(projects) && projects.length > 0;
        } catch {
          return false;
        }
      },
    };
  }
}
```

#### Asset Distribution

**Rulesets**:
- **Current**: All rulesets bundled in core (`downloaded_assets/rulesets/`)
- **Proposed**:
  - Core: None until we have true generalized default rulesets
  - konveyor-java: Java-specific rulesets (ie. all of the rulesets repo)
  - konveyor-python: Python rulesets (python2 → python3)

**Analyzer Binary**:
- **Current**: Single kai-analyzer-rpc with all providers compiled in
- **Proposed**: **Same** - still one binary, but providers enabled dynamically via `--provider=<name>` flags
- Core extension bundles kai binary (all platforms)
- Language extensions don't duplicate the binary

**JDTLS Bundles**:
- **Current**: Bundled in core
- **Proposed**: Moved to `java` extension
- Only downloaded if user installs Java support

### Security, Risks, and Mitigations

#### Security Considerations

1. **Extension Trust Chain**
   - **Risk**: Malicious language extension could register fake provider and intercept analysis results
   - **Mitigation**: None because who cares if another extensions knows sees the results we push to what is already accessible as VScode diagnostics.

2. **Command Execution**
   - **Risk**: `executeCommand` allows arbitrary VSCode command execution
   - **Mitigation**:
     - Providers can only execute commands their language server exposes
     - Core validates command prefixes match registered languageId
     - Log all provider command executions

3. **Asset Path Injection**
   - **Risk**: Malicious provider could point to system paths outside extension
   - **Mitigation**:
     - Validate all asset paths are within extension's `extensionPath`
     - Use `context.asAbsolutePath()` to resolve paths safely

#### Technical Risks

1. **Activation Race Conditions**
   - **Risk**: Java extension activates before core, calls API that doesn't exist yet
   - **Mitigation**:
     ```typescript
     export async function activate(context: vscode.ExtensionContext) {
       const coreExt = vscode.extensions.getExtension("konveyor.konveyor");
       if (!coreExt) {
         throw new Error("Core not found");
       }

       // Wait for core to activate first
       const coreApi = await coreExt.activate();

       // Now safe to register
       coreApi.registerProvider(config);
     }
     ```

2. **Provider Registration After Analysis Started**
   - **Risk**: User starts analysis before Java extension activates
   - **Mitigation**:
     - Core extension delays analysis start by 2 seconds to allow providers to register
     - If no providers registered, show error: "No language support installed"
     - Future: Detect file types, prompt to install missing language extensions

3. **Breaking Changes in Core API**
   - **Risk**: Core extension updates break existing language extensions
   - **Mitigation**:
     - CI Checks

4. **Memory/Resource Leaks**
   - **Risk**: Multiple providers registering/unregistering could leak listeners
   - **Mitigation**:
     - Return `Disposable` from `registerProvider()`
     - Core tracks all registrations in `WeakMap`
     - Language extensions call `dispose()` on deactivation

#### User Experience Risks

1. **Installation Confusion**
   - **Risk**: Users don't understand why they need multiple extensions
   - **Mitigation**:
     - Extension pack: `konveyor-bundle` that installs core + Java
     - First-run wizard detects workspace languages, prompts to install support
     - Clear messaging: "Konveyor requires language-specific extensions"

2. **Upgrade Breakage**
   - **Risk**: Existing users upgrade core, Java support stops working
   - **Mitigation**:
     - **Migration release** (v2.0.0):
       - Core extension detects missing Java extension
       - Auto-installs `konveyor-java` on first activation
       - Shows notification: "Konveyor now uses separate language extensions"
     - Documentation: Prominent upgrade guide

## Design Details

### Repository Structure

```
konveyor-editor-extensions/
├── agentic/           # No change (platform-agnostic)
├── shared/            # No change (platform-agnostic)
├── webview-ui/        # No change (platform-agnostic)
├── tests/             # No change
│
└── vscode/            # Reorganize internally
    ├── lib/           # NEW: Shared utilities package
    │   ├── package.json
    │   ├── src/
    │   │   ├── pathUtils.ts      # Moved from vscode/src/utilities
    │   │   ├── fileUtils.ts      # Moved from vscode/src/utilities
    │   │   └── index.ts          # Export barrel
    │   └── tsconfig.json
    │
    ├── core/          # RENAMED: Was the root of vscode/src
    │   ├── package.json          # This IS the konveyor extension
    │   ├── src/
    │   │   ├── api/              # NEW: Extension API exports
    │   │   │   ├── types.ts
    │   │   │   └── index.ts
    │   │   ├── client/           # AnalyzerClient with provider support
    │   │   │   └── analyzerClient.ts  # MODIFIED: Provider registration
    │   │   ├── utilities/        # Core-specific (config, auth, profiles)
    │   │   ├── modelProvider/
    │   │   ├── issueView/
    │   │   └── extension.ts      # MODIFIED: API export
    │   └── tsconfig.json
    │
    └── java/          # NEW: Java language extension
        ├── package.json          # This IS konveyor-java extension
        │   ├── "activationEvents": ["onLanguage:java"]
        │   ├── "extensionDependencies": ["konveyor.konveyor", "redhat.java"]
        │   └── "contributes":
        │       ├── "configuration": { konveyor.java.* settings }
        │       └── "commands": { konveyor.java.* commands }
        ├── src/
        │   ├── extension.ts          # Activation & provider registration
        │   ├── javaProvider.ts        # ProviderConfig implementation
        │   └── javaExtensionCheck.ts  # Version validation for redhat.java
        ├── rulesets/                 # Java-specific rulesets
        └── assets/
            └── jdtls-bundles/
```

<!-- Need to review what Claude wrote here

### Migration Steps (Code Movement)

#### Phase 1: Extract API (Week 1)

**Create**: `vscode/core/src/api/`

```typescript
// vscode/core/src/api/types.ts
export interface ProviderConfig { /* ... */ }
export interface KonveyorCoreApi { /* ... */ }

// vscode/core/src/api/index.ts
export * from "./types";

// vscode/core/src/extension.ts
export function activate(context: vscode.ExtensionContext): KonveyorCoreApi {
  const orchestrator = new ProviderOrchestrator(context);
  return {
    registerProvider: (config) => orchestrator.register(config),
    getProviders: () => orchestrator.getAll(),
    onAnalysisStarted: (listener) => orchestrator.onStart(listener),
    onAnalysisCompleted: (listener) => orchestrator.onComplete(listener),
    onAnalysisFailed: (listener) => orchestrator.onFail(listener),
  };
}
```

**Modify**: `vscode/core/src/client/analyzerClient.ts`

```typescript
// Add provider registration
private providers = new Map<string, ProviderConfig>();

registerProvider(config: ProviderConfig): Disposable {
  this.providers.set(config.languageId, config);
  return { dispose: () => this.providers.delete(config.languageId) };
}

// Modify RPC handler
this.analyzerRpcConnection.onRequest("workspace/executeCommand", async (params) => {
  const provider = this.getProviderForCommand(params.command);
  return await provider.executeCommand(params.command, params.arguments);
});
```

#### Phase 2: Create Java Extension Scaffold (Week 2)

**Create**: `vscode/java/`

```bash
mkdir -p vscode/java/src
mkdir -p vscode/java/rulesets
mkdir -p vscode/java/assets/jdtls-bundles
```

**Create**: `vscode/java/package.json`

```json
{
  "name": "konveyor-java",
  "displayName": "Konveyor Java Language Support",
  "version": "1.0.0",
  "publisher": "konveyor",
  "engines": { "vscode": "^1.93.0" },
  "activationEvents": ["onLanguage:java"],
  "extensionDependencies": [
    "konveyor.konveyor",
    "redhat.java"
  ],
  "main": "./out/extension.js",
  "contributes": {
    "configuration": {
      "title": "Konveyor Java",
      "properties": {
        "konveyor.java.maxMemory": {
          "type": "string",
          "default": "4096m",
          "description": "Maximum memory for Java analyzer"
        }
      }
    }
  }
}
```

#### Phase 3: Move Java-Specific Code (Week 3)

**Move Files**:

```bash
# Java extension version check
mv vscode/src/extension.ts::checkJavaExtensionInstalled() \
   vscode/java/src/javaExtensionCheck.ts

# JDTLS integration logic (extract from analyzerClient.ts)
# Create vscode/java/src/javaProvider.ts with extracted logic
```

**Copy Assets**:

```bash
# JDTLS bundles
cp -r downloaded_assets/jdtls-bundles/ \
      vscode/java/assets/jdtls-bundles/

# Java rulesets (filter from main rulesets)
cp -r downloaded_assets/rulesets/java* \
      vscode/java/rulesets/
```

#### Phase 4: Update Build Scripts (Week 4)

**Modify**: `scripts/collect-assets.js`

```javascript
// Remove JDTLS bundle download (lines 129-140)
// Keep only:
// - kai binaries
// - Generic rulesets (cloud-readiness, discovery)
// - opensource-labels-file
// - fernflower
```

**Create**: `scripts/collect-java-assets.js`

```javascript
// Download JDTLS bundles
// Download Java-specific rulesets
// Extract from kai-rpc-server.linux-x86_64.zip
```

**Update**: `package.json` scripts

```json
{
  "scripts": {
    "collect-assets": "node scripts/collect-assets.js",
    "collect-assets:java": "node scripts/collect-java-assets.js",
    "build": "npm run build:core && npm run build:java",
    "build:core": "npm run build -w vscode/core",
    "build:java": "npm run build -w vscode/java"
  }
}
```

### Test Plan

#### Unit Tests

**Core API Tests** (`vscode/core/src/api/__tests__/`)
```typescript
describe("ProviderOrchestrator", () => {
  it("registers providers", () => {
    const orchestrator = new ProviderOrchestrator(mockContext);
    const config = createMockProviderConfig("java");
    const disposable = orchestrator.register(config);

    expect(orchestrator.getAll()).toContain(config);
    disposable.dispose();
    expect(orchestrator.getAll()).not.toContain(config);
  });

  it("prevents duplicate languageId registration", () => {
    const orchestrator = new ProviderOrchestrator(mockContext);
    orchestrator.register({ languageId: "java", /* ... */ });

    expect(() => {
      orchestrator.register({ languageId: "java", /* ... */ });
    }).toThrow("Provider 'java' already registered");
  });
});
```

#### E2E Tests

**Installation Flow**
  - Install core extension only
  - Open Java workspace
  - Verify prompt: "Install Konveyor Java support?"
  - Click install
  - Verify `konveyor-java` installed and activated
  - Run analysis successfully

## Implementation History

- 2025-01-10: Initial proposal created (provisional)
- TBD: Design review with konveyor/kai team
- TBD: Prototype implementation (core API + Java extension)
- TBD: E2E testing with mixed Java/Python workspace
- TBD: Beta release for early adopters
- TBD: GA release (implementable → implemented)

## Drawbacks

### 1. Installation Complexity

**Before**: One extension to install
**After**: Two extensions (core + language)

**Impact**:
- Users may not understand why they need multiple extensions
- Extension marketplace listing less discoverable

**Mitigation**:
- Create `konveyor-bundle` extension pack (one-click install)
- Auto-prompt to install language support when detected
- Clear documentation

### 2. Development Overhead

**Before**: Single repository, single release cycle
**After**: Multiple packages, coordinated releases

**Impact**:
- More complex CI/CD pipelines
- Version compatibility testing between core and language extensions
- Documentation must cover multiple extensions

**Mitigation**:
- Use npm workspaces (already in place)
- Automated version compatibility checks in CI
- Shared changelog generation

## Alternatives

### Duplicate Analyzer in Each Language Extension

**Design**:
- Each language extension bundles its own kai-analyzer-rpc
- No core analyzer client
- Core extension only aggregates results

**Pros**:
- **True independence**: Java and Python teams fully own their stacks
- **Parallel execution**: Multiple analyzer processes

**Cons**:
- **Resource waste**: 3 languages = 3 analyzer processes (300+ MB RAM)
- **Binary duplication**: 3x download size (kai binary is 20 MB)
- **Result synchronization**: Complex IPC to merge analysis results
- **Configuration drift**: Each analyzer may have different settings

**Rejected Because**: Wastes resources, complicates result aggregation (current proposal keeps analyzer in core)

## Infrastructure Needed

### 1. Extension Pack

**Create**: `konveyor-bundle` extension pack

```json
{
  "name": "konveyor-bundle",
  "displayName": "Konveyor (All Languages)",
  "version": "2.0.0",
  "publisher": "konveyor",
  "extensionPack": [
    "konveyor.konveyor",
    "konveyor.konveyor-java"
  ]
}
```

**Publish to**: VSCode Marketplace as separate listing

### 2. Marketplace Listings

**Existing**:
- `konveyor.konveyor` (update to v2.0, core only)

**New**:
- `konveyor.konveyor-java`
- `konveyor.konveyor-bundle` (extension pack)

**Future**:
- `konveyor.konveyor-python`
- `konveyor.konveyor-golang`

### 3. CI/CD Pipelines

**Current**: Single workflow builds one .vsix

**Needed**:
- `.github/workflows/build-core.yml` - Builds core extension
- `.github/workflows/build-java.yml` - Builds Java extension
- `.github/workflows/build-bundle.yml` - Creates extension pack
- `.github/workflows/test-integration.yml` - Tests core + Java together

### 4. Documentation Sites

**Update**: https://github.com/konveyor/editor-extensions/blob/main/README.md

**New Sections**:
- "Installation" - Explain core vs language extensions
- "Language Support" - List available language extensions
-->
