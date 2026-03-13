---
title: generic-provider-to-specific-providers
authors:
  - "@jmle"
  - "@pranavgaikwad"
  - "@savitharaghunathan"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2025-03-11
last-updated: 2025-03-11
status: provisional
see-also:
  - "https://github.com/konveyor/analyzer-lsp/issues/1013"
  - "docs/refactoring-generic-provider-to-specific-providers.md"
---

# Refactor Generic Service Client to Language-Specific Providers

This enhancement proposes splitting the current "generic external provider" in analyzer-lsp into separate, language-specific providers (e.g. Go, Python, Node.js), each with its own binary and initialization path. The goal is to align with the reality that LSP server initialization and behavior differ too much across languages to be reliably abstracted behind a single generic client.

The YAML `title` is lowercased with spaces replaced by `-`.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. **YAML strategy:** Default container settings use `yq-external-provider` for YAML, not the generic provider's `yaml_language_server` config. Should we (a) introduce a dedicated yaml-external-provider that uses the current yaml_language_server code, or (b) leave YAML to yq only and deprecate/remove the yaml_language_server backend from the generic provider?

2. **Image topology:** Should we ship one container image per language (e.g. go-provider, python-provider, nodejs-provider) or a single image that includes multiple provider binaries and an entrypoint that selects by argument/env? The former simplifies per-language ownership and sizing; the latter may reduce CI and deployment surface.

3. **Backward compatibility:** Should we retain a "generic" binary that supports only the current "generic" (gopls) client so that existing configs using `binaryPath: ".../generic-external-provider"` with `lspServerName: "generic"` continue to work without change, or is a breaking change to provider config acceptable?

## Summary

Today, analyzer-lsp uses a single binary, `generic-external-provider`, to serve multiple LSP-backed languages (Go/gopls, Python/pylsp, Node.js, and optionally YAML). The same binary is started multiple times with different `--name` flags; internally, a generic provider type dispatches to language-specific "service client builders" via a map and a runtime switch on `lspServerName`. Experience has shown that a single generic client cannot reliably interface with every language server because initialization and behavior differ too much (e.g. custom LSP handlers for YAML, custom reference-evaluation for Node.js, different capability sets). This enhancement proposes replacing that design with one provider per language—each with its own binary and no runtime dispatcher—following the pattern already used by `java-external-provider`. Shared logic (e.g. `lsp/base_service_client`) would remain; only the multi-language dispatcher and the convention of "one binary, many names" would go away. Users and operators would configure separate provider entries (and binaries/images) per language, improving clarity, maintainability, and reliability of each language backend.

## Motivation

### Goals

- **Reliability:** Each language provider owns its LSP initialization and capabilities so that language-specific quirks (e.g. YAML `workspace/configuration` handling, Node.js batching) are implemented in one place without conditional logic in a shared dispatcher.
- **Clarity:** One binary per language (or one clear entrypoint per language) so that configuration, debugging, and ownership are straightforward.
- **Consistency:** Align Go, Python, and Node.js providers with the existing pattern used by the Java provider (dedicated binary and package).
- **Maintainability:** Remove the generic provider's runtime switch on `lspServerName` and the `SupportedLanguages` map so that adding or changing a language does not touch a central dispatcher.

Success will be evident when: (1) Go, Python, and Node.js (and optionally YAML) are each served by a dedicated provider binary/package; (2) no single process dispatches to multiple LSP types via `lspServerName`; (3) provider settings and docs reference distinct binaries/images per language; (4) existing tests and demos pass with the new layout.

### Non-Goals

- Changing the analyzer-lsp provider gRPC API or the `BaseClient`/`ServiceClient` interfaces.
- Removing or rewriting the shared `lsp/base_service_client` layer; it remains the foundation for LSP-based providers.
- Changing how the builtin provider or non-generic providers (e.g. Java, yq) are implemented or configured.
- Supporting multiple LSP types in a single process as a feature; the refactor explicitly moves away from that model.

## Proposal

### User Stories

#### Story 1: Operator configures Go analysis

As an operator, I want to configure the analyzer to use a Go provider by pointing to a dedicated Go provider binary (e.g. `go-external-provider` or a renamed generic binary that only supports gopls), so that I do not rely on `lspServerName` in providerSpecificConfig and the behavior of the Go provider is isolated from other languages.

#### Story 2: Operator configures Python and Node.js analysis

As an operator, I want to configure Python and Node.js providers using distinct binaries (e.g. `python-external-provider`, `nodejs-external-provider`) and provider entries in my settings file, so that each language has its own process and I can scale or debug them independently.

#### Story 3: Developer adds or changes a language provider

As a developer, I want to add or modify support for a single language (e.g. Python) without editing a central "generic" provider or a shared `SupportedLanguages` map, so that changes are localized to that language’s provider package and binary.

### Implementation Details/Notes/Constraints

- **Current architecture (brief):** The analyzer loads provider configs (e.g. `provider_container_settings.json`); for each config with a `binaryPath`, the gRPC client layer runs that binary with `--port` and `--name` (where `--name` is taken from `initConfig[0].providerSpecificConfig.lspServerName`, default `"generic"`). The generic-external-provider binary constructs a `genericProvider` keyed by `--name`, which looks up a builder in `SupportedLanguages` and later may switch builders in `Init()` if config sends a different `lspServerName`. Each builder builds LSP `InitializeParams` and handlers in language-specific ways and returns a `ServiceClient` that embeds `LSPServiceClientBase` and an evaluator.

- **Target architecture:** Each of Go, Python, and Node.js (and optionally YAML) gets its own provider package and binary. Each binary implements `BaseClient` (e.g. `Capabilities()`, `Init()`) and constructs exactly one service client type (no map, no switch on `lspServerName`). The analyzer continues to spawn one process per provider config; configs will reference different `binaryPath` values per language. Shared code in `lsp/base_service_client` (and shared capability helpers where applicable) is reused by each provider.

- **Reference implementation:** `java-external-provider` is the pattern: own main, own package, no dependency on the generic provider or `SupportedLanguages`.

- **Key files to change or introduce:**
  - **Analyzer core:** `provider/grpc/provider.go` — `start()` today passes `--name` derived from `lspServerName`; for language-specific binaries this can be simplified or kept for backward compat. No change to `BaseClient`/`ServiceClient` interfaces.
  - **Generic provider (remove or shrink):** `external-providers/generic-external-provider/main.go`, `pkg/generic_external_provider/provider.go`, `pkg/server_configurations/constants.go`; move `server_configurations/generic`, `pylsp`, `nodejs`, (and optionally `yaml_language_server`) into new provider modules.
  - **New providers:** New directories, e.g. `go-external-provider`, `python-external-provider`, `nodejs-external-provider`, each with its own `main.go`, provider type, and one server configuration (no dispatcher).
  - **Config and build:** Update `provider_container_settings.json`, Makefile, root Dockerfile, generic-provider Dockerfile (or replace with per-language Dockerfiles), `.github/workflows/image-build.yaml`, `.github/workflows/demo-testing.yml`, and testdata/docs as needed.
  - **Tests:** Move or duplicate `server_configurations/*/service_client_test.go` into the new provider packages; adjust `provider/provider_test.go` if it asserts on generic-provider-specific behavior.

- **Constraint:** The analyzer calls `Capabilities()` before `Init()`, so each process must advertise the correct capabilities at startup. With one binary per language, this is natural; no runtime switch is required.

### Security, Risks, and Mitigations

- **Risk: More binaries/images.** More artifacts to build, sign, and distribute. Mitigation: Document the new layout clearly; consider a single image with multiple binaries and an entrypoint if the community prefers to limit the number of images.
- **Risk: Breaking existing configs.** Users who currently use one `binaryPath` and multiple `initConfig`/`lspServerName` entries will need to point each language to its own binary. Mitigation: Provide a migration path (e.g. retain a "generic" binary for gopls-only) and update default settings and docs.
- **Risk: Regressions in behavior.** Each language’s logic is moved into a new package. Mitigation: Preserve existing tests (move or re-run), run existing e2e/demos (golang-e2e, python-e2e, nodejs-e2e) against the new providers, and add any missing integration tests.
- **Security:** No new network exposure or auth model; providers remain out-of-process gRPC services. Binary execution is unchanged (analyzer still spawns one process per provider config). Review by maintainers familiar with the provider and analyzer-lsp security model is sufficient.

## Design Details

### Test Plan

- **Unit tests:** Each new provider package must have unit tests for its provider type (Capabilities, Init) and for its service client behavior where non-trivial (e.g. Node.js or YAML-specific evaluation). Existing tests under `external-providers/generic-external-provider/pkg/server_configurations/*/service_client_test.go` will be moved or reimplemented in the new packages so that behavior is preserved.
- **Integration / e2e:** Existing demo and e2e flows (e.g. golang-e2e, python-e2e, nodejs-e2e in `external-providers/generic-external-provider/e2e-tests/` and `.github/workflows/demo-testing.yml`) will be updated to use the new provider binaries/images and re-run to ensure no regressions.
- **Isolation:** Each language provider can be tested in isolation by running its binary with the same gRPC contract; the analyzer’s existing provider client logic does not need to change beyond how it starts the binary (and possibly how `--name` is passed).
- Tricky areas: (1) Ensuring capability lists and Init parameters match what the analyzer expects for each provider name. (2) YAML if we introduce a dedicated provider—tests for the yaml_language_server path should be retained or moved.

### Upgrade / Downgrade Strategy

- **Upgrade:** Users currently using the generic-external-provider for go, python, or nodejs must update their provider settings to use the new per-language binaries (and, if applicable, new image names). If we retain a "generic" binary that only supports gopls, configs that use only `lspServerName: "generic"` can keep the same binaryPath until that binary is eventually deprecated. Documentation and default `provider_container_settings.json` (or equivalent) will be updated to reflect the new layout.
- **Downgrade:** Reverting to the previous release would require restoring the single generic-external-provider binary and configs that use `lspServerName` to select the language. No schema change to the analyzer’s provider config format is required; only the number and identity of binaries and the presence of `lspServerName` in config may change.

## Implementation History

| Date       | Status / Milestone |
|-----------|---------------------|
| 2025-03-11 | Enhancement proposed (provisional); refactoring analysis document added. |

## Drawbacks

- **Operational overhead:** Operators must manage and optionally image-scan more than one provider binary/image if we ship separate images per language.
- **Duplication of structure:** Each language provider will have its own main and provider type, which repeats some boilerplate (flag parsing, server startup). This is intentional for isolation but increases code surface.
- **Breaking change for current generic users:** Anyone relying on a single generic binary with multiple `lspServerName` values will need to switch to multiple provider entries and binaries (unless we keep a generic binary for gopls-only and document it as the “Go” provider).

## Alternatives

1. **Keep the generic provider and only fix initialization per language.** We could leave the single binary and dispatcher in place and try to make initialization more pluggable (e.g. more builder options). This was considered insufficient because the issue states that a generic client cannot reliably interface with every language server; the problem is structural, not just a few missing options.

2. **Single image with multiple binaries and a router entrypoint.** We could keep one container image that contains go-external-provider, python-external-provider, nodejs-external-provider and an entrypoint that invokes the right binary based on env or args. This preserves “one image” for deployment while still allowing separate codebases and no in-process dispatcher. This is left as an option in Open Questions (image topology).

3. **Leave YAML in the generic provider only.** If we do not introduce a dedicated yaml-external-provider, we could leave the yaml_language_server code in a shrunken “generic” provider that only supports generic (gopls) and yaml. That would reduce the number of new provider modules at the cost of retaining a small dispatcher for two backends.

## Infrastructure Needed

- CI jobs (or matrix entries) to build and test each new provider binary and, if applicable, per-language images. The exact layout (one job per provider vs one job building multiple binaries) can follow the decision on image topology.
- No new repositories or subprojects are required; new providers can live under `external-providers/` in the analyzer-lsp repo, consistent with `java-external-provider` and `yq-external-provider`.
