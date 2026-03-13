# Refactoring: Generic Service Client to Specific Providers

**Related:** [GitHub Issue #1013](https://github.com/konveyor/analyzer-lsp/issues/1013) — *[refactor]: Generic Service Client to have specific providers.*

This document describes the current architecture of the generic external provider, the rationale for the proposed refactor, and a detailed inventory of every component that would be affected by splitting the generic provider into language-specific providers.

---

## 1. Executive Summary

**Current state:** A single binary, `generic-external-provider`, serves multiple LSP-backed languages (generic/gopls, Python/pylsp, Node.js, YAML). The same binary is started multiple times with different `--name` flags (e.g. `--name generic`, `--name pylsp`, `--name nodejs`). Internally, a single “generic provider” type dispatches to language-specific *service client builders* via a map and a runtime switch on `lspServerName`.

**Proposed change:** Stop using one generic binary as a multi-language dispatcher. Instead, give each language its own provider (and typically its own binary), similar to `java-external-provider`. Each provider would own its LSP initialization and capabilities without sharing a generic dispatcher.

**Reason (from the issue):** *“We are not finding it the case, that we can have a generic client that reliably interfaces with any generic language server. The initialization of those language servers are too different to make this fully compatible.”*

---

## 2. Current Architecture — How the Generic Provider Fits In

### 2.1 Provider discovery and spawning (analyzer-lsp core)

- **Config loading:** `provider.GetConfig(settingsFile)` reads provider configs (e.g. `provider_container_settings.json` or YAML). Each entry has `name`, `binaryPath` or `address`, and `initConfig[]`.
- **Client creation:** For every config with `name != "builtin"`, `provider/lib.GetProviderClient(config, log)` is used. That function delegates to `grpc.NewGRPCClient(config, log)` for all non-builtin providers.
- **Binary startup (generic only):** In `provider/grpc/provider.go`, `start()` is called when `config.BinaryPath != ""`. It:
  1. Reserves a free port.
  2. Reads `lspServerName` from `config.InitConfig[0].ProviderSpecificConfig["lspServerName"]` (default `"generic"`).
  3. Runs: `exec.CommandContext(ctx, config.BinaryPath, "--port", port, "--name", name)`.
  4. Dials `localhost:{port}` and returns the gRPC connection and stdout pipe.

So for a settings file that defines three “providers” (e.g. go, python, nodejs) all using the same `binaryPath: "/usr/local/bin/generic-external-provider"`, the analyzer spawns **three separate processes** of the same binary, each with a different `--name` (e.g. `generic`, `pylsp`, `nodejs`).

**Files:**

| Path | Role |
|------|------|
| `provider/provider.go` | `Config`, `InitConfig`, `GetConfig()`, `validateAndUpdateProviderSpecificConfig()`, `BaseClient`, `ServiceClient`, `InternalProviderClient` interfaces. |
| `provider/lib/lib.go` | `GetProviderClient(config, log)` — branches on `config.Name == "builtin"` → builtin, else → `grpc.NewGRPCClient`. |
| `provider/grpc/provider.go` | `NewGRPCClient()`, `start()` (spawns binary with `--port` and `--name` from `lspServerName`), `Init()`, `Capabilities()`, `Evaluate()`, etc. |
| `provider/server.go` | gRPC server used by external provider processes: `NewServer(client BaseClient, ...)`, `Capabilities()`, `Init()`, `Evaluate()`, `Stop()`, etc. |
| `cmd/analyzer/main.go` | Loads configs, builds `finalConfigs`, calls `lib.GetProviderClient(config, log)` per config, starts providers, runs engine. |
| `cmd/dep/main.go` | Same pattern for dep command. |

### 2.2 Generic external provider binary and process layout

- **Entrypoint:** `external-providers/generic-external-provider/main.go`
  - Parses flags: `--port`, `--socket`, `--name` (lsp server name), `--log-level`, TLS, etc.
  - If `--name` is empty, defaults to `"generic"`.
  - Builds one provider: `client := generic_external_provider.NewGenericProvider(*lspServerName, log)`.
  - Starts the gRPC server: `provider.NewServer(client, *port, ...)`.

So each process is “locked” at startup to one LSP type via `--name`; that name is used to choose the initial `ServiceClientBuilder` and the capability set advertised to the analyzer.

**Files:**

| Path | Role |
|------|------|
| `external-providers/generic-external-provider/main.go` | Flag parsing, `NewGenericProvider(*lspServerName, log)`, `provider.NewServer(client, ...)`. |

### 2.3 Generic provider type and dispatch

- **Type:** `genericProvider` in `pkg/generic_external_provider/provider.go`
  - Holds: `lspServerName`, `serviceClientBuilder` (a `serverconf.ServiceClientBuilder`), and a list of `capabilities` derived from that builder.
- **Construction:** `NewGenericProvider(lspServerName, log)` looks up `serverconf.SupportedLanguages[lspServerName]`; if missing, falls back to `"generic"`. It then stores that builder and builds the capability list from `ctor.GetGenericServiceClientCapabilities(log)`.
- **Capabilities():** Returns the stored capabilities (so each process advertises only the capabilities for its LSP).
- **Init():** Receives `provider.InitConfig` from the analyzer (via gRPC). It reads `c.ProviderSpecificConfig["lspServerName"]` again; if it differs from the provider’s current `lspServerName`, it **reassigns** `serviceClientBuilder` via a switch on the known constants (`generic`, `pylsp`, `nodejs`, `yaml_language_server`). Then it calls `p.serviceClientBuilder.Init(ctx, log, c)` and returns the resulting `ServiceClient`.

So:
- Capabilities are fixed per process (by `--name`).
- Init can still “switch” the builder at runtime if the config says a different `lspServerName`, which keeps the door open for one process to serve multiple LSP types in theory, but in practice the analyzer spawns one process per provider config and passes matching `lspServerName` in initConfig.

**Files:**

| Path | Role |
|------|------|
| `external-providers/generic-external-provider/pkg/generic_external_provider/provider.go` | `genericProvider`, `NewGenericProvider()`, `Capabilities()`, `Init()` (switch on `lspServerName`, delegate to builder). |
| `external-providers/generic-external-provider/pkg/server_configurations/constants.go` | `GenericClient`, `PythonClient`, `YamlClient`, `NodeClient`; `ServiceClientBuilder` interface; `SupportedLanguages` map. |

### 2.4 Server configurations (per-language “backends”)

Each language has a package under `server_configurations/` that implements:

- A **config struct** (embeds `base.LSPServiceClientConfig`).
- A **service client struct** (embeds `*base.LSPServiceClientBase` and `*base.LSPServiceClientEvaluator[T]`).
- A **builder** (e.g. `GenericServiceClientBuilder`) with:
  - `Init(ctx, log, c provider.InitConfig) (provider.ServiceClient, error)`
  - `GetGenericServiceClientCapabilities(log) []base.LSPServiceClientCapability`

`Init()` in each builder:

- Unmarshals `c.ProviderSpecificConfig` into the client’s config.
- Builds LSP `InitializeParams` (RootURI, WorkspaceFolders, Capabilities, InitializationOptions, etc.) in **language-specific ways**.
- Calls `base.NewLSPServiceClientBase(ctx, log, c, initializeHandler, params)` — the **handler** and **params** differ by language.
- Builds the evaluator from the capability list and attaches it to the client.

Differences that matter for “initialization is too different”:

| Language | Notable Init / behavior |
|----------|-------------------------|
| **generic** | Standard `InitializeParams`; `base.LogHandler(log)`; capabilities: referenced, dependency (no-op), echo. |
| **pylsp** | Same pattern as generic for Init; capabilities: referenced only. |
| **nodejs** | Same handler; custom `EvaluateReferenced` (didOpen/didClose batching, custom symbol/reference handling). Capabilities: referenced. |
| **yaml_language_server** | **Custom handler** for `workspace/configuration` (workaround for yaml-language-server behavior). Builds `params` similarly; capabilities: referenced; custom `EvaluateReferenced` using schema + diagnostics. |

So: YAML has a custom jsonrpc2 handler; Node has a custom `EvaluateReferenced` and different LSP usage; generic has extra capabilities (dependency, echo). There is no single “generic” init that fits all.

**Files:**

| Path | Role |
|------|------|
| `external-providers/generic-external-provider/pkg/server_configurations/generic/service_client.go` | Generic LSP client; referenced, dependency, echo capabilities. |
| `external-providers/generic-external-provider/pkg/server_configurations/pylsp/service_client.go` | Pylsp client; referenced only. |
| `external-providers/generic-external-provider/pkg/server_configurations/nodejs/service_client.go` | Node client; custom `EvaluateReferenced` (batching, didOpen/didClose). |
| `external-providers/generic-external-provider/pkg/server_configurations/yaml_language_server/service_client.go` | YAML client; custom handler for `workspace/configuration`; custom `EvaluateReferenced` (schema + publishDiagnostics). |

### 2.5 Base LSP client and capabilities (shared)

- **Base client:** `lsp/base_service_client/base_service_client.go`
  - `LSPServiceClientBase`: holds context, log, config, dialer, jsonrpc2 connection, caches, server capabilities.
  - `NewLSPServiceClientBase(ctx, log, c, initializeHandler, initializeParams)`: unmarshals config, validates `lspServerPath`, builds temp dir if needed, creates `CmdDialer`, dials LSP, sends `initialize`, sends `initialized`, returns base.
  - Methods: `Stop()`, `GetAllDeclarations()`, `GetAllReferences()`, `GetDependencies()` (calls external dependency binary), `Handle()` (e.g. publishDiagnostics), etc.
- **Capabilities:** `lsp/base_service_client/base_capabilities.go`
  - Generic `EvaluateReferenced[T]` and `EvaluateNoOp[T]` used by generic and pylsp; nodejs and yaml use their own `EvaluateReferenced` implementations.

**Files:**

| Path | Role |
|------|------|
| `lsp/base_service_client/base_service_client.go` | `LSPServiceClientBase`, `NewLSPServiceClientBase()`, LSP lifecycle and helpers. |
| `lsp/base_service_client/base_capabilities.go` | Shared capability logic (e.g. `EvaluateReferenced[T]`). |

### 2.6 Provider interfaces (core)

- **BaseClient:** `Capabilities() []Capability`, `Init(ctx, log, InitConfig) (ServiceClient, InitConfig, error)`.
- **ServiceClient:** `Evaluate()`, `Stop()`, `GetDependencies()`, `GetDependenciesDAG()`, `NotifyFileChanges()`.
- **InternalProviderClient:** extends the client interface used by the analyzer (Init, Evaluate, deps, etc.).

The generic provider’s `genericProvider` implements `BaseClient`; each builder’s `Init()` returns a type that implements `ServiceClient` (often by embedding the base and the evaluator).

---

## 3. Configuration and deployment touchpoints

### 3.1 Provider settings (which binary, which name)

- **Example config:** `provider_container_settings.json`
  - **go:** `binaryPath: "/usr/local/bin/generic-external-provider"`, `providerSpecificConfig.lspServerName: "generic"`.
  - **python:** Same binaryPath, `lspServerName: "pylsp"`.
  - **nodejs:** Same binaryPath, `lspServerName: "nodejs"`.
  - **yaml:** Uses `yq-external-provider` (different binary); the generic provider’s YAML server config is still present in code but not used by this file.
  - **java:** `binaryPath: "/usr/local/bin/java-external-provider"` (separate binary).

So today, go/python/nodejs are distinguished only by config (and the `--name` passed when the same binary is spawned).

### 3.2 Build and images

- **Makefile**
  - `external-generic` builds the generic provider binary.
  - `build-generic-provider` builds the generic provider image (with sed for golang-dependency-provider image ref).
  - `run-external-providers-local` / `run-external-providers-pod`: run one container for java, one for yq, then **three** containers from the same generic provider image (golang-provider, nodejs, python) with different `--name` and ports.
- **Dockerfile (repo root)**
  - `make build` builds analyzer and providers; copies `generic-external-provider` binary to `/usr/local/bin/generic-external-provider`.
- **external-providers/generic-external-provider/Dockerfile**
  - Multi-stage: build generic-external-provider and gopls; base image has Node, Python, pylsp, typescript-language-server; copies in generic-external-provider and golang-dependency-provider; entrypoint is the generic binary.
- **CI / demo**
  - `.github/workflows/image-build.yaml`: builds generic-external-provider image.
  - `.github/workflows/demo-testing.yml`: uses `IMG_GENERIC_PROVIDER` for go/python/nodejs demos; e2e paths under `external-providers/generic-external-provider/e2e-tests/` (golang-e2e, python-e2e, nodejs-e2e).

**Files:**

| Path | Role |
|------|------|
| `Makefile` | Build generic binary and image; run multiple generic containers with different `--name`. |
| `Dockerfile` | Copy generic-external-provider binary into main image. |
| `external-providers/generic-external-provider/Dockerfile` | Image that runs generic-external-provider (with runtimes for go/python/node). |
| `.github/workflows/image-build.yaml` | Build generic-external-provider image. |
| `.github/workflows/demo-testing.yml` | Demo/e2e using generic provider for go, python, nodejs. |
| `provider_container_settings.json` | Default provider configs (binaryPath and initConfig per provider name). |

### 3.3 Tests

- Unit tests that instantiate the generic provider and/or builders:
  - `external-providers/generic-external-provider/pkg/server_configurations/generic/service_client_test.go`
  - `external-providers/generic-external-provider/pkg/server_configurations/pylsp/service_client_test.go`
  - `external-providers/generic-external-provider/pkg/server_configurations/nodejs/service_client_test.go`
  - `external-providers/generic-external-provider/pkg/server_configurations/yaml_language_server/service_client_test.go`
- `provider/provider_test.go` uses `lspServerName` in expected provider config (generic).

---

## 4. What “breaking out to specific providers” would mean

Conceptually:

- **One provider per language** (e.g. go, python, nodejs, and optionally yaml) with its own:
  - Binary (or at least its own `main` and provider type), and/or
  - Clear entrypoint and no runtime switch on `lspServerName` inside one process.
- **No generic dispatcher:** Remove the single `genericProvider` that holds a map and switches on `lspServerName` in `Init()`.
- **Shared code:** Keep using `lsp/base_service_client` (and optionally shared capability helpers) where it makes sense; each specific provider can still embed the base and the evaluator.

Reference implementation: **java-external-provider** — its own binary, own package, `java.NewJavaProvider(...)`, no dependency on the generic provider or `SupportedLanguages` map.

---

## 5. Affected components — full checklist

### 5.1 Analyzer-lsp core (no interface change, but config and spawn logic)

| Component | Current behavior | Effect of refactor |
|-----------|------------------|---------------------|
| `provider/lib/lib.go` — `GetProviderClient()` | Any non-builtin config → `grpc.NewGRPCClient(config, log)`. | No change to interface. Each “language” could be a different config with a different `binaryPath` (e.g. go-provider, python-provider, nodejs-provider). |
| `provider/grpc/provider.go` — `start()` | Uses `config.BinaryPath` and `config.InitConfig[0].ProviderSpecificConfig["lspServerName"]` to set `--name` for the generic binary. | For specific providers, either: (a) keep passing a name for backward compat, or (b) remove the `--name`/`lspServerName` convention for those binaries and have each binary fixed to one language. |
| `provider/provider.go` | Defines `BaseClient`, `ServiceClient`, `InitConfig`, `ProviderSpecificConfig`. | No change to interfaces. Provider-specific config can remain per-provider (e.g. python-specific keys without `lspServerName`). |
| `provider/server.go` | Accepts any `BaseClient`; calls `Capabilities()` and `Init()`. | No change; each specific provider still implements `BaseClient`. |
| `cmd/analyzer/main.go` / `cmd/dep/main.go` | Load configs and call `GetProviderClient` per config. | Config files would list separate provider entries with distinct `binaryPath` (and optionally different `name`) for go, python, nodejs. |

### 5.2 Generic external provider (to be split or removed)

| Component | Current behavior | Effect of refactor |
|-----------|------------------|---------------------|
| `external-providers/generic-external-provider/main.go` | Parses `--name`, calls `NewGenericProvider(*lspServerName, log)`, starts server. | **Remove or repurpose.** Each new provider (e.g. python-external-provider) would have its own main that constructs one provider type (e.g. Python only). |
| `external-providers/generic-external-provider/pkg/generic_external_provider/provider.go` | Dispatches by `lspServerName`; holds one builder; returns capabilities from that builder. | **Eliminate.** Logic moves into each language-specific provider (no runtime switch). |
| `external-providers/generic-external-provider/pkg/server_configurations/constants.go` | `SupportedLanguages` map and builder interface. | **Split or remove.** Each specific provider only knows its own builder; no shared map. |
| `external-providers/generic-external-provider/pkg/server_configurations/generic/` | Generic (gopls) service client and builder. | Becomes the core of a **go-external-provider** (or stays as “generic” if one binary remains for generic LSPs). |
| `external-providers/generic-external-provider/pkg/server_configurations/pylsp/` | Pylsp service client and builder. | Becomes the core of **python-external-provider** (or pylsp-external-provider). |
| `external-providers/generic-external-provider/pkg/server_configurations/nodejs/` | Node service client and builder. | Becomes the core of **nodejs-external-provider**. |
| `external-providers/generic-external-provider/pkg/server_configurations/yaml_language_server/` | YAML service client and builder. | Either a **yaml-external-provider** (if not using yq) or removed if yq remains the only YAML path. |

### 5.3 Base LSP client (shared)

| Component | Current behavior | Effect of refactor |
|-----------|------------------|---------------------|
| `lsp/base_service_client/base_service_client.go` | Shared base for all LSP-based service clients. | **Keep.** All specific providers can keep embedding and using it. |
| `lsp/base_service_client/base_capabilities.go` | Shared capability functions. | **Keep.** Language providers that use the same behavior can keep calling these. |

### 5.4 Configuration and deployment

| Component | Current behavior | Effect of refactor |
|-----------|------------------|---------------------|
| `provider_container_settings.json` | go, python, nodejs share `binaryPath: ".../generic-external-provider"` and differ by `lspServerName`. | **Update.** Point go to go-external-provider binary, python to python-external-provider, nodejs to nodejs-external-provider (or keep one “generic” binary for go only and add python/nodejs binaries). |
| `Makefile` | Builds one generic binary; runs multiple containers with same image and different `--name`. | **Update.** Build and run separate images/containers per provider (or one image with multiple binaries). |
| Root `Dockerfile` | Copies single generic-external-provider binary. | **Update.** Copy each new provider binary (or a single multi-binary image). |
| `external-providers/generic-external-provider/Dockerfile` | Single image with generic binary + runtimes. | **Option A:** Replace with separate Dockerfiles per language. **Option B:** One image that contains multiple binaries (go, python, nodejs) and an entrypoint that selects by env/arg. |
| `.github/workflows/image-build.yaml` | Builds generic-external-provider image. | **Update.** Build new provider images (or one multi-provider image). |
| `.github/workflows/demo-testing.yml` | Uses IMG_GENERIC_PROVIDER for go/python/nodejs. | **Update.** Use the appropriate image per demo (or one image with multiple binaries). |
| `provider/testdata/*.yaml` / `*.json` | Some reference generic-external-provider binaryPath. | **Update** if tests assume a single generic binary; otherwise leave as-is for backward compat tests. |
| `debug/debug.Dockerfile` | Builds and copies generic-external-provider. | **Update** if debug image should include specific providers. |

### 5.5 Tests

| Component | Current behavior | Effect of refactor |
|-----------|------------------|---------------------|
| `external-providers/generic-external-provider/pkg/server_configurations/*/service_client_test.go` | Use `generic_external_provider.NewGenericProvider("...", log)` and/or builders. | **Move** to the corresponding new provider packages; remove dependency on generic provider and constants map. |
| `provider/provider_test.go` | Expects `lspServerName` in provider config. | **Adjust** if tests are generic-provider-specific; keep or duplicate for new providers as needed. |

### 5.6 Documentation

| Component | Current behavior | Effect of refactor |
|-----------|------------------|---------------------|
| `external-providers/generic-external-provider/docs/README.md` | Describes generic provider and how to add new service clients via the map. | **Rewrite** to describe either (a) the remaining “generic” provider (e.g. go only) or (b) the pattern for language-specific providers. |
| `docs/experimental/python-provider.md` | References generic-external-provider and pylsp. | **Update** to point to python-external-provider (or new layout). |
| Any other docs that reference “generic provider” or “SupportedLanguages” | — | **Update** to reflect new provider layout. |

---

## 6. Intricacies and risks

1. **Capabilities before Init:** The analyzer calls `Capabilities()` on the gRPC client (which forwards to the provider process) before calling `Init()`. So each process must advertise the right capabilities at startup. Today that’s done by locking the generic provider to one `lspServerName` via `--name`. With separate binaries, each binary naturally advertises one set of capabilities.

2. **InitConfig and lspServerName:** Today, `Init()` can change the builder if `ProviderSpecificConfig["lspServerName"]` differs from the provider’s `lspServerName`. After the refactor, that switch is unnecessary for single-language providers; config can be simplified (e.g. drop `lspServerName` for python/nodejs/go-specific providers).

3. **Shared base and evaluator:** All four server configs depend on `lsp/base_service_client` and the evaluator pattern. Moving code into new repos/packages should preserve this; only the “dispatcher” (generic provider + constants map) goes away.

4. **YAML:** The default container settings use `yq-external-provider` for yaml, not the generic provider’s yaml_language_server. So “breaking out” YAML could mean either (a) a new yaml-external-provider that uses the current yaml_language_server code, or (b) leaving YAML to yq only and deprecating the yaml_language_server config in the generic provider.

5. **Backward compatibility:** If we keep a “generic” binary that only supports the current “generic” (gopls) client, existing configs that use `binaryPath: ".../generic-external-provider"` with `lspServerName: "generic"` could keep working. Configs that use the same binary with `pylsp` or `nodejs` would need to be updated to new binaries.

6. **Build and CI:** Splitting into multiple binaries/images increases the number of build artifacts and possibly CI jobs. Alternative: one image with multiple binaries and an entrypoint that chooses by `--name` or env (minimal change to run scripts but still allows separate codebases per language).

---

## 7. Summary table of file-level impact

| Area | Files to modify / move / remove |
|------|----------------------------------|
| **Provider core** | `provider/grpc/provider.go` (start logic for `--name`); optionally `provider/provider.go` if any generic-specific validation exists. |
| **Generic provider** | Remove or shrink `generic-external-provider`: `main.go`, `pkg/generic_external_provider/provider.go`, `pkg/server_configurations/constants.go`; move each server_configurations/* into a corresponding new provider. |
| **New providers** | New dirs: e.g. `go-external-provider`, `python-external-provider`, `nodejs-external-provider` (each with main, provider type, one server config). |
| **Base** | `lsp/base_service_client/*` — keep as-is. |
| **Config** | `provider_container_settings.json`; testdata in `provider/testdata`. |
| **Build / CI** | `Makefile`, root `Dockerfile`, `external-providers/generic-external-provider/Dockerfile`, `.github/workflows/image-build.yaml`, `.github/workflows/demo-testing.yml`, `debug/debug.Dockerfile`. |
| **Tests** | All `server_configurations/*/service_client_test.go`; `provider/provider_test.go` if it asserts on generic behavior. |
| **Docs** | `external-providers/generic-external-provider/docs/README.md`, `docs/experimental/python-provider.md`, any other references to generic provider or SupportedLanguages. |

This document is intended as the single reference for the refactoring scope and touchpoints; it does not prescribe an order of implementation or a specific design for the new provider binaries/images (e.g. one image vs many).
