---
title: ai-assisted-rules-generation
authors:
  - "@savitharaghunathan"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-04-03
last-updated: 2026-04-14
status: implementable
replaces:
  - N/A
superseded-by:
  - N/A
see-also:
  - https://github.com/konveyor/enhancements/pull/259
  - https://github.com/konveyor/enhancements/pull/274
---

# AI-Assisted Rules Generation

Extract migration patterns from documentation via LLM and generate structurally valid Konveyor rules, with interactive IDE support (MCP server) and batch/CI pipelines (CLI + Agent Skill).

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. **Test data generation validity**: LLM-generated test data is biased toward the LLM's understanding of the rule. Is synthetic test data useful beyond smoke testing?

2. **Degraded mode without LLM key**: Should the Agent Skill be usable without an LLM API key (agent does extraction, MCP tools handle construction/validation, no test generation)?

3. **Binary distribution**: How should the `rulegen` binary be distributed to users? GitHub Releases, Homebrew, container image, or bundled with the editor extension?

4. **Merge with analyzer-lsp MCP?**: The analyzer-lsp MCP (PR #259) already exposes `rules_validate`. Should rule construction tools (`construct_rule`, `construct_ruleset`, `get_help`) live in the analyzer-lsp MCP server instead of a separate server, avoiding duplication and giving users a single MCP server for all rule operations?

## Summary

Konveyor supports multiple languages, but only Java has a substantial ruleset. Creating rules requires both
domain-specific migration knowledge and Konveyor rule syntax expertise, which is a high barrier that limits adoption. Konveyor and Kai are only useful if rules exist to support the migration. Even for custom libraries within enterprises,custom rules are needed.

This enhancement proposes `rulegen`, a Go binary that extracts migration patterns from documentation (migration guides, changelogs, code snippets) via LLM and generates structurally valid Konveyor rules. The core engine handles ingestion, pattern extraction, rule generation, validation, and test data generation. It is exposed through two interfaces:

- **Interface A: MCP Server** (`rulegen serve`) -- 4 deterministic tools for interactive rule construction in IDEs. No server-side LLM needed.
- **Interface B: CLI + Agent Skill** (`rulegen generate/validate/test/pipeline` + `SKILL.md`) -- Full pipeline access for batch/CI and agentic workflows.

Both interfaces share the same core library. The architecture is designed to integrate complementary engines (e.g.,
[semver-analyzer](https://github.com/shawn-hurley/semver-analyzer) for deterministic rule generation from code diffs) in the future.

## Motivation

### The Rules Gap

- Only Java has comprehensive rulesets (inherited from Windup)
- Go, Node.js, C#, Python, and frontend frameworks have minimal coverage
- Enterprise-specific rules (custom libraries, internal frameworks) don't exist
- Creating rules requires understanding both the migration domain AND Konveyor's YAML rule syntax

### Goals

- **Lower the barrier to creating rules**: A developer with migration knowledge but no Konveyor syntax expertise should be able to generate rules
- **Expand language/framework coverage**: Enable rule generation for any language or framework where migration
  documentation exists
- **Verify AI-generated rules**: Validate rules structurally (syntax) and functionally (kantra)
- **Integrate with the Konveyor ecosystem**: Work with kantra, Kai, existing rulesets, and developer tooling

### Non-Goals

- **Replace human rule authors**: AI generates draft rules; humans review and decide whether to commit
- **Guarantee rule correctness**: AI-generated rules are proposals, not verified facts
- **Build a new rule engine**: Rules use existing Konveyor YAML format, validated by existing kantra infrastructure

## Proposal

### User Stories

#### Story 1: Generate rules from a migration guide

A developer needs Konveyor rules for migrating Spring Boot 3 to 4. They have the official migration guide URL. They run `rulegen generate` pointing at the guide. The tool ingests the guide, extracts migration patterns (deprecated APIs, renamed packages, removed configurations), and generates a validated rule for each. The developer runs `rulegen test` to verify the rules match expected patterns via kantra. The developer reviews and submits a PR to konveyor/rulesets.

#### Story 2: Interactive rule authoring in IDE

A developer asks their IDE agent: "Create Konveyor rules from this Spring Boot migration guide" and pastes a URL. The agent, connected to the `rulegen` MCP server, reads the guide, identifies migration patterns, and builds a rule for each one. The server validates each rule and returns valid YAML. The developer reviews the generated rules, makes adjustments, and saves them.

#### Story 3: Agentic workflow via Agent Skill

A developer invokes the rules generation skill in OpenCode, Goose, or another agentic CLI tool. The skill asks: "What is the migration? Do you have a migration guide or documentation?" The developer provides a URL.

The skill generates rules from the guide, validates them, and asks: "Generated 12 rules. Want me to run tests?" The
developer agrees. All 12 rules pass kantra validation. The developer reviews the rules, makes edits, and commits.

#### Story 4: CI pipeline

A CI job monitors framework changelog feeds (Spring Boot, Quarkus, Jakarta EE). When a new version is released, the job generates candidate rules from the changelog and validates them against kantra. Rules that pass are included in a PR to konveyor/rulesets for human review.

### Implementation Details/Notes/Constraints

#### Architecture

```text
┌─────────────────────────────────────────────────────────────────────────┐
│ rulegen binary                                                          │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │ Core library(internal/)                                           │  │
│  │                                                                   │  │
│  │  ingestion/   • URL fetch + HTML→markdown conversion              │  │
│  │               • File/text intake, chunking                        │  │
│  │                                                                   │  │
│  │  extraction/  • LLM pattern extraction from content chunks        │  │
│  │               • Cross-chunk deduplication, metadata detection     │  │
│  │                                                                   │  │
│  │  generation/  • Pattern→Rule mapping (12 condition types)         │  │
│  │               • LLM message generation, concern-based grouping    │  │
│  │                                                                   │  │
│  │  rules/       • Rule/Ruleset types, condition builders            │  │
│  │               • Structural validator, YAML serializer             │  │
│  │                                                                   │  │
│  │  testgen/     • LLM test data generation, kantra runner           │  │
│  │               • Fix loop (kantra test → analyze failures → hints) │  │
│  │                                                                   │  │
│  │  llm/         • llm interface                                     │  │
│  │               • Anthropic, OpenAI, Gemini, Ollama providers       │  │
│  │                                                                   │  │
│  │  workspace/   • Output directory management                       │  │
│  │  templates/   • Embedded LLM prompt templates (go:embed)          │  │
│  └──────────────────────┬────────────────────┬───────────────────────┘  │
│                         │                    │                          │
│              ┌──────────┘                    └──────────┐               │
│              ▼                                          ▼               │
│  ┌──────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │ Interface A: MCP Server      │  │ Interface B: CLI + Agent Skill  │  │
│  │                              │  │                                 │  │
│  │ rulegen serve                │  │ rulegen generate                │  │
│  │ • stdio / Streamable HTTP    │  │ • Full LLM-driven pipeline      │  │
│  │ • no API keys                │  │                                 │  │
│  │                              │  │ rulegen validate                │  │
│  │ Tools:                       │  │ • Structural validation         │  │
│  │ • construct_rule             │  │                                 │  │
│  │ • construct_ruleset          │  │ rulegen test                    │  │
│  │ • validate_rules             │  │ • Test data gen + fix loop      │  │
│  │ • get_help                   │  │                                 │  │
│  │                              │  │ rulegen pipeline                │  │
│  │                              │  │ • generate → test → report      │  │
│  │                              │  │                                 │  │
│  │                              │  │ + SKILL.md                      │  │
│  └──────────────┬───────────────┘  └────────────────┬────────────────┘  │
│                 │                                   │                   │
│              MCP Protocol                     Shell / CI / Skill        │
│                 │                                   │                   │
│                 ▼                                   ▼                   │
│  ┌──────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │ MCP Clients                  │  │ Consumers:                      │  │
│  │ Claude Code, Cursor, VS Code │  │ Shell, CI/CD, Agent Skills      │  │
│  │ Goose.                       │  │                                 │  │
│  └──────────────────────────────┘  └─────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Core Engine

##### Generation Pipeline

```text
Input (URL/file/text)
  │
  ▼ [ingestion]
Chunks (markdown sections)
  │
  ▼ [extraction] ← LLM
MigrationPatterns (deduplicated)
  │
  ▼ [generation] ← deterministic mapping + LLM messages
Rules (grouped by concern) + Ruleset
  │
  ▼ [validation] ← deterministic
Valid rules written to workspace
```

**Step 1: Ingestion** — Accepts URLs, file paths, or raw text. URLs are fetched and converted from HTML to Markdown.
Content is chunked by document structure (headers) to fit within LLM context windows.

**Step 2: Extraction** — Each chunk is sent to the LLM to extract migration patterns (deprecated APIs, renamed packages, removed configurations, etc.). Patterns are deduplicated across chunks. If `--source`/`--target` are not provided, metadata is auto-detected from the content.

Each pattern captures: what to detect (source API/class/config), what replaces it (if anything), why migration is needed (rationale, category, complexity), and how to match it (provider type, code location, file pattern). Optional fields include code examples and documentation URLs.

**Step 3: Generation** — Each pattern maps to a Konveyor rule. The provider type determines the condition type (e.g., java → `java.referenced`, go → `go.referenced`). Complexity maps to effort (trivial=1 through expert=9). Rule messages are generated via LLM with fallback to a simple template. Rules are grouped by concern (e.g., ejb, security, web) into separate YAML files.

**Step 4: Validation** — Deterministic structural checks: required fields, valid categories, condition-specific
requirements (patterns, locations, regex syntax), duplicate rule IDs. Returns `{ valid, errors, warnings, rule_count }`.

##### Rule Types (`internal/rules/`)

**Rule struct**:

```yaml
- ruleID: spring-boot-3-to-4-00010
  description: "Short description"
  when:
    java.referenced:
      pattern: javax.servlet.http.HttpServlet
      location: IMPORT
  message: "Migration guidance with Before/After examples"
  category: mandatory
  effort: 3
  labels:
    - konveyor.io/source=spring-boot-3
    - konveyor.io/target=spring-boot-4
  links:
    - title: "Migration Guide"
      url: "https://example.com/guide"
  tag:                       # optional, alternative to message
    - "tag-value"
```

**Ruleset struct**:

```yaml
name: spring-boot-3/spring-boot-4
description: "Rules for migrating from spring-boot-3 to spring-boot-4"
labels:
  - konveyor.io/source=spring-boot-3
  - konveyor.io/target=spring-boot-4
```

##### Supported Condition Types

| Language | Condition Types |
|----------|----------------|
| Java | `java.referenced`, `java.dependency` |
| Go | `go.referenced`, `go.dependency` |
| Node.js/TypeScript | `nodejs.referenced` |
| C# | `csharp.referenced` |
| Any | `builtin.filecontent`, `builtin.file`, `builtin.xml`, `builtin.json`, `builtin.hasTags`, `builtin.xmlPublicID` |

Each condition type has specific fields (see `construct_rule` tool inputs under Interface A for details). Conditions can be combined with `Or` and `And` combinators.

##### Test Data Generation and Fix Loop (`internal/testgen/`)

**Test data generation**: For each rule file (grouped by concern), the LLM generates compilable application code that triggers the rules. Supports Java, Go, Node.js/TypeScript, and C# with language-appropriate build files and dependency resolution. A `.test.yaml` file is generated for kantra with test entries expecting at least one incident per rule.

**Fix loop** (up to `--max-iterations`, default 3):

1. **Kantra tests**: Run kantra against the test data. Report passed/total.
2. **Fix test data** (if failures remain): For each failing rule, analyze the kantra debug output to extract the pattern and provider. Generate a code hint via LLM explaining what the test data needs to trigger the rule. Regenerate test data with hints. Continue to next iteration. Note: this phase fixes the test data, not the rules themselves.

#### Interface A: MCP Server

`rulegen serve` starts an MCP server exposing 4 deterministic tools. The server requires no LLM, no API keys, and no
external dependencies. The client's LLM does all reasoning; the server ensures structural correctness.

**Important limitation**: Only construction and validation tools are exposed. The full generation pipeline and test data generation require server-side LLM access. MCP sampling (where the client provides LLM completion) is [not widely supported by MCP clients](https://modelcontextprotocol.io/clients), so these capabilities remain CLI-only.

**Interactive session flow** (Story 2: bulk rule creation from migration guide):

```text
Developer                Client LLM               MCP Server
    │                        │                        │
    │  "Create Konveyor      │                        │
    │   rules from this      │                        │
    │   migration guide:     │                        │
    │   <url>"               │                        │
    │───────────────────────>│                        │
    │                        │                        │
    │                        │  get_help("all")       │
    │                        │───────────────────────>│
    │                        │  rule syntax docs      │
    │                        │<───────────────────────│
    │                        │                        │
    │                        │  (fetches URL, reads   │
    │                        │   migration guide,     │
    │                        │   identifies patterns) │
    │                        │                        │
    │                        │  construct_rule(       │
    │                        │    ruleID, pattern,    │
    │                        │    condition_type,     │
    │                        │    message, ...)       │
    │                        │───────────────────────>│
    │                        │  { yaml, valid: true } │
    │                        │<───────────────────────│
    │                        │                        │
    │                        │  ... repeat per        │
    │                        │      pattern ...       │
    │                        │                        │
    │                        │  construct_ruleset(    │
    │                        │    name, labels)       │
    │                        │───────────────────────>│
    │                        │  ruleset YAML          │
    │                        │<───────────────────────│
    │                        │                        │
    │                        │  validate_rules(       │
    │                        │    "./rules/")         │
    │                        │───────────────────────>│
    │                        │  { valid, errors,      │
    │                        │    warnings }          │
    │                        │<───────────────────────│
    │                        │                        │
    │  "Here are 12 rules    │                        │
    │   for Spring Boot 3→4. │                        │
    │   All valid."          │                        │
    │<───────────────────────│                        │
    │                        │                        │
    │  (reviews, edits,      │                        │
    │   commits)             │                        │
```

**Tools**:

**`construct_rule`**: Build a single Konveyor rule from parameters. Validates all inputs and returns valid rule YAML. Required input: `ruleID`, `condition_type` (enum of 12 types), `message`, `category` (enum:
mandatory/optional/potential), `effort` (integer). Condition-specific input:

- `java.referenced`: `pattern` (required), `location`
- `java.dependency` / `go.dependency`: `name` or `nameRegex` (one required), `lowerbound`, `upperbound`
- `go.referenced` / `nodejs.referenced`: `pattern` (required)
- `csharp.referenced`: `pattern` (required), `location`
- `builtin.filecontent`: `pattern` (required), `filePattern`
- `builtin.xml` / `builtin.json`: `xpath` (required)
- `builtin.file`: `pattern` (required)
- `builtin.hasTags`: `tags` (required, non-empty array)
- `builtin.xmlPublicID`: `regex` (required)

Optional input: `description`, `labels`, `links`. Returns `{ yaml, valid, errors }`.

**`construct_ruleset`**: Input: `name` (required), `description`, `labels`. Output: Valid ruleset YAML.

**`validate_rules`**: Input: `rules_path` (file or directory). Loads rules (skips `ruleset.yaml`), runs all structural checks. Output: `{ valid, errors, warnings, rule_count }`.

**`get_help`**: Input: `topic` -- one of `condition_types`, `locations`, `labels`, `categories`, `rule_format`,
`ruleset_format`, `examples`, `all` (default). Returns hardcoded documentation content.

**Transport**: stdio (recommended for local IDE integration) and Streamable HTTP (remote/multi-client).

#### Interface B: CLI + Agent Skill

The CLI binary provides full pipeline access. An Agent Skill (`SKILL.md`) following the [Agent Skills](https://agentskills.io) open standard wraps the CLI for agentic workflows in IDEs.

**CLI Architecture**:

```text
┌──────────────────────────────────────────────────────────┐
│  Agent Skill (SKILL.md)                                  │
│  (Claude Code, Cursor, VS Code/Copilot, Gemini CLI,      │
│   JetBrains Junie, OpenAI Codex, Goose, etc.)            │
│                                                          │
│  Defines multi-step workflow:                            │
│  context → generation → validation                       │
│  → iteration → save                                      │
└────────────────────────┬─────────────────────────────────┘
                         │ invokes
                         ▼
┌──────────────────────────────────────────────────────────┐
│  CLI Binary (rulegen)                                    │
│                                                          │
│  rulegen generate   ← Full LLM-driven pipeline           │
│  rulegen validate   ← Structural validation              │
│  rulegen test       ← Test data gen + kantra + fix loop  │
│  rulegen pipeline   ← generate → test → report (unified) │
└──────────────────────────────────────────────────────────┘
```

**Full CLI pipeline** (generate → test):

```text
                          rulegen generate
┌───────────┐     ┌─────────────────────────────────────────────────┐
│           │     │                                                 │
│ Input     │     │  ┌────────────┐   ┌─────────────┐   ┌────────┐  │     ┌──────────┐
│ URL / file│────>│  │ ingestion  │──>│ extraction  │──>│ gener- │  │────>│ output/  │
│ / text    │     │  │            │   │             │   │ ation  │  │     │  rules/  │
│           │     │  │ fetch,     │   │ LLM pattern │   │        │  │     │  *.yaml  │
└───────────┘     │  │ HTML→MD,   │   │ extraction, │   │ map to │  │     │          │
                  │  │ chunk      │   │ deduplicate │   │ rules, │  │     │ ruleset  │
                  │  └────────────┘   └─────────────┘   │ LLM    │  │     │ .yaml    │
                  │                                     │ msgs,  │  │     └─────┬────┘
                  │              validate ◄──────────── │ group  │  │           │
                  │                                     └────────┘  │           │
                  └─────────────────────────────────────────────────            │
                                                                                │
                          rulegen test                                          │
                   ┌─────────────────────────────────────────────────┐          │
                   │                                                 ◄──────────┘
                   │  ┌────────────┐   ┌─────────────┐               │     ┌─────────┐
                   │  │ generate   │──>│ run kantra  │               │────>│ output/ │
                   │  │ test data  │   │ tests       │               │     │  tests/ │
                   │  │            │   │             │               │     │         │
                   │  │ LLM gen    │   │ report      │               │     └─────────┘
                   │  │ test app   │   │ passed/total│               │
                   │  └────────────┘   └──────┬──────┘               │
                   │                          │                      │
                   │  fix test data via LLM  ◄┘                      │
                   │  analyze failures, generate hints,              │
                   │  regenerate test data, iterate                  │
                   │  (up to --max-iterations)                       │
                   └─────────────────────────────────────────────────┘
```

**CLI Commands**:

**`rulegen generate`**:

```sh
rulegen generate \
  --input https://example.com/spring-boot-migration-guide \
  --source spring-boot-3 --target spring-boot-4 \
  --language java --output ./output --provider anthropic
```

- Flag: `--input`; Required: Yes; Default: ; Description: URL, file path, or text content
- Flag: `--source`; Required: No; Default: auto-detected; Description: Source technology
- Flag: `--target`; Required: No; Default: auto-detected; Description: Target technology
- Flag: `--language`; Required: No; Default: auto-detected; Description: Programming language (java, go, nodejs, csharp)
- Flag: `--output`; Required: No; Default: `output`; Description: Output directory
- Flag: `--provider`; Required: No; Default: `RULEGEN_LLM_PROVIDER` env var; Description: LLM provider: anthropic,
  openai, gemini, ollama. Required via flag or env var.

Runs the full pipeline: ingest -> chunk -> extract -> generate -> validate -> write.

**`rulegen validate`**:

```sh
rulegen validate --rules ./rules/
```

- Flag: `--rules`; Required: Yes; Default: ; Description: Path to rules directory or file

Returns JSON `ValidationResult`.

**`rulegen test`**:

```sh
rulegen test --rules ./output/rules/ --provider anthropic --max-iterations 3
```

- Flag: `--rules`; Required: Yes; Default: ; Description: Path to rules directory
- Flag: `--output`; Required: No; Default: ; Description: Output directory (parent of rules/)
- Flag: `--language`; Required: No; Default: auto-detected from rules; Description: Programming language
- Flag: `--source`; Required: No; Default: ; Description: Source technology (used in test data generation prompts)
- Flag: `--target`; Required: No; Default: ; Description: Target technology (used in test data generation prompts)
- Flag: `--provider`; Required: No; Default: `RULEGEN_LLM_PROVIDER` env var; Description: LLM provider. Required via
  flag or env var.
- Flag: `--max-iterations`; Required: No; Default: `3`; Description: Max fix loop iterations

Runs the fix loop: generate test data, run kantra, analyze failures, regenerate with hints.

**`rulegen pipeline`**:

```sh
rulegen pipeline \
  --input https://example.com/spring-boot-migration-guide \
  --source spring-boot-3 --target spring-boot-4 \
  --language java --output ./output --provider gemini
```

Accepts all flags from `generate` plus `--max-iterations` (default: `3`). Runs the full end-to-end pipeline: generate rules → generate test data → run kantra fix loop → write summary report. Equivalent to running `rulegen generate` followed by `rulegen test`, but as a single command for CI/batch use.

**Agent Skill (`SKILL.md`)**: Portable markdown file defining the agentic workflow. Works across 30+ agents via the
[Agent Skills](https://agentskills.io) open standard.

Workflow:

1. Gather context: What is the migration? Docs available? Code available?
2. Route to appropriate engine:
   - Documentation available -> `rulegen generate`
   - Both codebases available -> semver-analyzer (future integration)
   - Both -> semver-analyzer first (higher confidence), rulegen fills gaps
3. Run generation
4. Validate output (`rulegen validate`)
5. Optionally generate test data (`rulegen test`)
6. Present results for human review

**Agent Skill workflow** (Story 3: agentic workflow in IDE):

```text
Developer                    Agent (via SKILL.md)              rulegen CLI
    │                              │                               │
    │  "Generate migration         │                               │
    │   rules for Spring           │                               │
    │   Boot 3 to 4"               │                               │
    │─────────────────────────────>│                               │
    │                              │                               │
    │  "Do you have a migration    │                               │
    │   guide URL or docs?"        │                               │
    │<─────────────────────────────│                               │
    │                              │                               │
    │  "Here: <url>"               │                               │
    │─────────────────────────────>│                               │
    │                              │                               │
    │                              │  (web fetch, read guide,      │
    │                              │   determine: docs available   │
    │                              │   → route to rulegen)         │
    │                              │                               │
    │                              │  rulegen generate             │
    │                              │  --input <url>                │
    │                              │  --provider anthropic         │
    │                              │──────────────────────────────>│
    │                              │  output/rules/*.yaml   <──────│
    │                              │                               │
    │                              │  rulegen validate             │
    │                              │  --rules ./output/rules/      │
    │                              │──────────────────────────────>│
    │                              │  { valid: true }   <──────────│
    │                              │                               │
    │  "Generated 12 rules.        │                               │
    │   Want me to run tests?"     │                               │
    │<─────────────────────────────│                               │
    │                              │                               │
    │  "Yes"                       │                               │
    │─────────────────────────────>│                               │
    │                              │                               │
    │                              │  rulegen test                 │
    │                              │  --rules ./output/rules/      │
    │                              │──────────────────────────────>│
    │                              │  12/12 passed      <──────────│
    │                              │                               │
    │  "All 12 rules passed        │                               │
    │   kantra. Files saved        │                               │
    │   to output/."               │                               │
    │<─────────────────────────────│                               │
    │                              │                               │
    │  (reviews, edits, commits)   │                               │
```

Key properties: Zero infrastructure (just a markdown file), leverages agent's native capabilities (web search, file I/O, user interaction), embeds domain knowledge in the prompt, multi-step with reasoning, invokes CLI for all heavy lifting.

**Example `SKILL.md`**:

`````markdown
---
name: konveyor-rules-generation
description: >
  Generate Konveyor analyzer rules from migration guides and documentation.
  Use when creating rules for a new migration path or expanding coverage
  for an existing one. Requires the rulegen binary and an LLM API key.
---

# Konveyor Rules Generation

## Prerequisites
- `rulegen` binary installed and on PATH
- LLM provider configured: `export RULEGEN_LLM_PROVIDER=anthropic`
- Corresponding API key set (e.g., `export ANTHROPIC_API_KEY=<key>`)
- `kantra` installed (optional, for test data generation)

## Workflow

1. Ask the user: What migration are they targeting? Do they have a
   migration guide URL or documentation?

2. Run the full pipeline (generate + test in one step):
   ```sh
   rulegen pipeline \
     --input <url-or-file> \
     --source <source> --target <target> \
     --provider $RULEGEN_LLM_PROVIDER \
     --output ./output
   ```

   Or run steps individually for more control:
   ```sh
   rulegen generate --input <url-or-file> --source <source> \
     --target <target> --provider $RULEGEN_LLM_PROVIDER --output ./output
   rulegen validate --rules ./output/rules/
   rulegen test --rules ./output/rules/ --provider $RULEGEN_LLM_PROVIDER
   ```

3. Present results: number of rules generated, validation status,
   and test pass rate. Let the user review, edit, and commit.
`````

#### Existing Work

- Project: [konveyor/rulesets](https://github.com/konveyor/rulesets)
  Description: Existing rulesets repository
  Relevance: Target for generated rules

- Enhancement: [analyzer-lsp MCP](https://github.com/konveyor/enhancements/pull/259)
  Description: MCP server for analyzer-lsp (analysis engine)
  Relevance: Complementary: exposes `analyze_run`, `rules_validate`, `dependencies_get` etc.

- Project: [analyzer-rule-generator](https://github.com/konveyor-ecosystem/analyzer-rule-generator)
  Description: Python-based AI rules generator
  Relevance: migration guides, Claude skill, E2E pipeline

- Project: [scribe](https://github.com/sshaaf/scribe)
  Description: Java-based MCP server for rules 
  Relevance: MCP tool approach

- Project: [semver-analyzer](https://github.com/shawn-hurley/semver-analyzer)
  Description: Rust-based deterministic API diff
  Relevance: Future integration as complementary engine

### Security, Risks, and Mitigations

- **LLM API keys**: Configured via environment variables (CLI only). Keys are never logged, committed, or included in rule output. Raw user input (URLs, file content) is not logged — only input type and size are recorded. 
- **Prompt injection**: Migration guides from URLs could contain adversarial content. Input sanitization and prompt
  guardrails required.
- **URL ingestion**: URLs are fetched with a hardened HTTP client with SSRF mitigation that blocks loopback and private IP addresses.
- **File system access**: `validate_rules` (MCP) reads files at client-provided paths. Path traversal is restricted to the working directory by resolving and rejecting paths that escape it (e.g., `../../../etc/passwd`). Workspace directory names are sanitized (no `..`, `/`, `\`).
- **Supply chain**: Generated rules affect how kantra analyzes applications. Human review before committing to rulesets is essential.
- **LLM hallucination**: Structural validation catches invalid regex and wrong condition types. Semantic errors require human review.

## Design Details

### Test Plan

- **Unit tests**: Core library functions (ingestion, rule construction, condition builders, validation, serialization). No LLM or kantra needed.
- **Integration tests**: Full pipeline with mock LLM.
- **E2E tests**: Real LLM + kantra. Requires API keys and kantra binary
- **MCP protocol tests**: Tool discovery, invocation, response format

### Upgrade / Downgrade Strategy

**Upgrade**: `rulegen` is a new standalone binary. No existing installations or data to migrate. Generated rules use the standard Konveyor YAML format and work with any version of kantra that supports the condition types used.

**Downgrade**: No persistent state — no databases, caches, or config files. Replacing the binary with an older version has no side effects. Generated output files are plain YAML and do not depend on the generating version.

**MCP tool compatibility**: Breaking changes to tool inputs will use new tool names (e.g., `construct_rule_v2`).

## Implementation History

## Drawbacks

- **LLM dependency**: Pattern extraction requires an LLM. Quality varies by provider. Mitigation: support multiple
  providers including local models (Ollama).
- **Validation gap**: No fully trustworthy automated validation for AI-generated rules. Synthetic test data is  biased. Human review remains necessary.
- **Multiple interfaces**: MCP and CLI+Skill share a core library but each needs maintenance.

## Alternatives

### CLI Only

Just the binary. Any agent can shell out to it. No server, no skill, no protocol.

- **Pros**: Simplest to build and maintain. Works in CI/CD natively. Single binary distribution.
- **Cons**: No IDE integration beyond shell invocation. No tool discovery or schema enforcement. Agents must parse CLI output as unstructured text.

### MCP Only

MCP server without CLI commands. Provides tool discovery and schema enforcement.

- **Pros**: Standardized protocol for IDE integration. Schema-enforced inputs prevent malformed rules. Works across all MCP-compatible clients.
- **Cons**: No batch/CI mode. Test data generation requires server-side LLM (MCP sampling not widely supported). No Agent Skill portability for non-MCP agents.

### Agent Skill using MCP

Agent Skill orchestrates the workflow, using MCP tools (`construct_rule`, `validate_rules`) for rule construction and
validation. The agent's own LLM handles pattern extraction — no server-side LLM or API keys needed.

- **Pros**: No API key management for the skill. Schema-enforced rule construction via MCP tools. Agent extracts patterns natively.
- **Cons**: No batch/CI mode. No test data generation or kantra fix loop. Extraction quality depends on the client LLM.

### Agent Skill Only

Purely prompt-based approach with no binary or MCP server. Relies entirely on the agent's LLM for rule construction and validation.

- **Pros**: Zero infrastructure — just a markdown file. No binary to install or distribute. Works anywhere an agent runs.
- **Cons**: No structural validation — quality depends entirely on the client LLM. No batch mode. No kantra integration for functional testing. Rules may have syntax errors that only surface at analysis time.

## Future Extensions

- **semver-analyzer integration**: Deterministic rule generation from API surface diffs between two codebases. The skill can route to it when both codebases are available (`v1` and `v2`). Currently TypeScript only, designed for
  multi-language extensibility.
- **Research agents**: LLM-driven discovery of migration paths, breaking changes, and migration guides
- **GitHub mining**: Discover migration patterns from real-world repositories; feed discovered repos into
  semver-analyzer
- **Knowledge database**: Accumulate migration intelligence over time and use that knowledge to generate rules.

## Infrastructure Needed

- **Kantra**
- **CI**: GitHub Actions for unit, integration, and E2E tests
- **LLM API access**: API keys for test generation, E2E test runs

