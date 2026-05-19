---
title: provider-specific-rule-packaging
authors:
  - "@jmle"
reviewers:
  - "@jortel"
  - "@eemcmullan"
  - "@shawn-hurley"
  - "@djzager"
  - "@pranavgaikwad"
approvers:
  - TBD
creation-date: 2026-05-14
last-updated: 2026-05-15
status: provisional
---

# Provider-Specific Rule Packaging

This enhancement proposes a mechanism for each provider to package and load only its language-specific rules, rather than all rules from the rulesets repository.

## Summary

Currently, all analysis rules from the rulesets repository are pulled into each provider, regardless of whether the
provider can actually execute them. This causes issues such as the C# provider attempting to run Java rules with "run always"
selectors, leading to unnecessary processing and potential errors. This enhancement proposes a solution where each provider
packages only the rules applicable to its target language, while maintaining the rulesets repository as the central source of
truth for rule definitions.

## Motivation

The current approach of distributing all rules to all providers has several drawbacks:

1. **Incorrect rule execution**: Providers attempt to execute rules that are not applicable to their target language 
   (e.g., C# provider running Java rules)
2. **Inefficiency**: Each provider loads and processes a large number of irrelevant rules
3. **Bloated distributions**: Provider containers and packages include unnecessary rule files

By enabling provider-specific rule packaging, we can eliminate these issues while maintaining ease of contribution
and consistency across the IDE, Hub, and CLI environments.

### Goals

- Enable each provider to package and load only the rules for its specific target language
- Maintain the rulesets repository as the single source of truth for all rule definitions
- Ensure the solution works seamlessly across IDE, Hub, and CLI (kantra) deployments
- Make it easy for contributors to add or modify rules without needing to understand complex distribution mechanics
- Provide clear guidelines for which rules belong to which provider

### Non-Goals

- Support for rules that require multiple providers (beyond the builtin + one language provider combination)
- Migration of existing C# provider rules from their current repository to the rulesets repository (can be addressed separately)

## Proposal

This proposal establishes a clear provider-to-ruleset mapping and a build/runtime mechanism to distribute
only relevant rules to each provider.

### Provider-to-Ruleset Mapping

The following mapping will be established:

| Provider | Ruleset Directory | Notes |
|----------|------------------|-------|
| Java | `rulesets/stable/java/` | Most mature ruleset |
| .NET (C#) | Currently in provider repo | Future: migrate to `rulesets/stable/dotnet/` |
| Node.js | `rulesets/stable/nodejs/` | Existing ruleset |
| Python | `rulesets/stable/python/` (future) | No rules yet |
| Go | `rulesets/stable/go/` (future) | No rules yet |
| YQ | `rulesets/stable/yq/` (future) | No rules yet |
| Builtin | N/A | Helper provider, no dedicated rules |

The builtin provider is special - it doesn't have its own rules but provides conditions
(like `builtin.filecontent`, `builtin.xml`, `builtin.file`) that can be used within any language-specific rule.
Rules can combine conditions from both the builtin provider and their language-specific provider.

### User Stories

#### Story 1: Java Provider User
As a user analyzing a Java application, I want the Java provider to only evaluate Java-specific rules so that I don't
see irrelevant warnings or errors from rules meant for other languages.

#### Story 2: Rule Contributor
As a contributor adding a new Java migration rule, I want to add my rule to the `rulesets/stable/java/` directory and
have it automatically available in the Java provider without needing to modify multiple repositories or configuration files.

#### Story 3: Multi-Language Project User
As a user with a project containing both Java and Node.js code (for instance), I want to run both providers and have each one evaluate
only the rules relevant to its language, while both can leverage builtin provider conditions.

#### Story 4: IDE User
As a user of the Konveyor IDE extension, I want provider-specific rule packaging to work transparently, with no
additional configuration required on my part.

#### Story 5: Kantra User
As a user running kantra for Java analysis, I want the bundled Java provider to include all Java rules without requiring
external network access or additional downloads.

#### Story 6: Custom Rules User
As a user with organization-specific migration rules, I want to provide my custom rules in addition to the default
bundled rules so that my analysis includes both standard and custom checks.

### Implementation Details/Notes/Constraints

#### Current Rule Distribution Architecture

Before describing the proposed changes, it's important to understand how rules are currently distributed:

**Current Flow:**
1. **rulesets repository**: Rules are authored here organized by language (java/, nodejs/, dotnet/)
2. **tackle2-seed repository**: Seeding process adds metadata (UUID, dependencies, checksums) to rules
3. **Distribution to deployments**:
   - **Hub**: Clones tackle2-seed during container build, copies `resources/` to `/tmp/seed`, seeds database with ALL rulesets
   - **Kantra**: Clones tackle2-seed during container build, copies `resources/rulesets/` to `/opt/rulesets` in the runner container image with ALL rulesets. At runtime, kantra extracts these from the runner container to the host filesystem
   - **Providers (containers)**: Do NOT bundle rules themselves; they receive rules from Hub or kantra at runtime

**Key Point**: Currently, ALL seeded rules are bundled into both Hub and kantra containers, regardless of which providers are actually used. Individual provider containers (java-external-provider, dotnet-external-provider, etc.) do NOT contain any rules.

**Current Architecture (Simplified):**
```
tackle2-seed/resources/rulesets/
├── java/
├── nodejs/
└── dotnet/

        ↓ (bundled at build time)

┌─────────────────────────────────────┐
│ Hub Container                       │
│ /tmp/seed/rulesets/                 │
│   ├── java/ (ALL)                   │
│   ├── nodejs/ (ALL)                 │
│   └── dotnet/ (ALL)                 │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Kantra Runner Container             │
│ /opt/rulesets/                      │
│   ├── java/ (ALL)                   │
│   ├── nodejs/ (ALL)                 │
│   └── dotnet/ (ALL)                 │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Provider Containers                 │
│ (java/nodejs/dotnet-provider)       │
│                                     │
│ NO RULES BUNDLED                    │
└─────────────────────────────────────┘
```

**Proposed Architecture:**
```
tackle2-seed/resources/rulesets/
├── java/
├── nodejs/
└── dotnet/

        ↓ (language-specific bundling in providers)

┌───────────────────────────────────────┐
│ Hub Container                         │
│ /tmp/seed/rulesets/                   │
│   ├── java/ (ALL - for DB/UI)         │
│   ├── nodejs/ (ALL)                   │
│   └── dotnet/ (ALL)                   │
│                                       │
│ Spawns provider containers            │
│ (rules bundled in provider images)    │
│                                       │
│ Optional: mount custom rules          │
│  -v /custom/rules:/opt/custom-rulesets:ro
└───────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Kantra Distribution (ZIP)           │
│ rulesets/java/ (for containerless)  │
│                                     │
│ Optional: custom rules on host      │
│ mounted to provider containers      │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Java Provider Container             │
│ /opt/rulesets/ (Java only - bundled)│
│ /opt/custom-rulesets/ (optional)    │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Node.js Provider Container          │
│ /opt/rulesets/ (Node.js - bundled)  │
│ /opt/custom-rulesets/ (optional)    │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ .NET Provider Container             │
│ /opt/rulesets/ (dotnet - bundled)   │
│ /opt/custom-rulesets/ (optional)    │
└─────────────────────────────────────┘
```

#### Rule Source of Truth

The `rulesets` repository will remain the authoritative source for all rule definitions. This provides:
- Centralized rule management and versioning
- Single location for rule contributions and reviews
- Consistent rule quality and standards
- Clear visibility into all available rules

The `tackle2-seed` repository will also remain the single source of truth for the distribution of rules. The provider-specific rule packaging
will occur **after** the seeding process, meaning providers will fetch their language-specific rules from the
seeded output in tackle2-seed, not directly from the rulesets repository.

#### Required Changes to Current Architecture

To achieve provider-specific rule packaging, the following changes are needed:

**Provider Container Changes (Core Change):**
- Currently: Provider containers do NOT bundle any rules
- Proposed: Each provider bundles its language-specific seeded rules from tackle2-seed
  - Dockerfile copies relevant ruleset directory: e.g., `COPY --from=rulesets /tackle2-seed/resources/rulesets/java/ /opt/rulesets/`
- **Rule Loading**: Providers load rules from `/opt/rulesets/` (bundled at build time)
- All providers are self-contained with their rules bundled

**Hub Changes:**
- Continues to seed database with ALL rulesets for management/UI purposes
- When spawning providers, simply runs the provider containers (which already have rules bundled)
- No rule mounting needed - providers are self-contained

**Kantra Changes:**
- **Distribution (ZIP)**: Bundles Java rules for containerless mode
  - Java rules packaged in kantra distribution at `rulesets/java/` for native Java provider execution
- **Containerized providers**: Use their bundled rules directly (bundled at `/opt/rulesets/` in each provider image)
- Simpler architecture: all providers are self-contained

**tackle2-seed Changes:**
- Ensure seeding metadata (rulesets.yaml index) supports provider/language filtering
- No structural changes needed; providers will consume existing language-organized directories

#### Rule Distribution Mechanism

Providers will use **build-time bundling**:

**Build Time:**
- Provider Dockerfile clones tackle2-seed at a specific version/tag
- Copies language-specific seeded rules to `/opt/rulesets/` in the container image
- Example: `COPY --from=rulesets /tackle2-seed/resources/rulesets/java/ /opt/rulesets/`

**Runtime:**
- Providers load rules from `/opt/rulesets/` (bundled in the image)
- All providers are self-contained with no external dependencies for rules

**Deployment Modes:**
- **Hub deployment**: Runs provider containers which use their bundled rules at `/opt/rulesets/`
- **Kantra deployment**: 
  - Containerless Java: Uses Java rules from kantra distribution (extracted to host filesystem)
  - Containerized providers: Use their bundled rules at `/opt/rulesets/`

#### Custom Rules Support

In addition to the bundled default rules, users can provide their own custom rules for organization-specific
migration scenarios, proprietary framework migrations, or other specialized analysis needs.

**Custom Rules Mechanism:**
- Providers load rules from two locations:
  1. `/opt/rulesets/` - Default bundled rules (always loaded from the container image)
  2. `/opt/custom-rulesets/` - Custom user-provided rules (optional, via volume mount)
- Custom rules are loaded **in addition to** default rules, not as a replacement
- Custom rules can be unstructured (do not need to follow the language-specific directory organization of default rules)
- Override behavior: The existing rule loading logic for handling duplicate rule IDs is preserved

**Deployment-Specific Custom Rules:**

**Hub:**
- Custom rules provided via volume mount when spawning provider containers
- Example: `-v /path/to/custom/rules:/opt/custom-rulesets:ro`
- Hub can manage custom rulesets through its storage mechanisms and mount them to providers

**Kantra:**
- Custom rules provided via volume mount from host filesystem to provider containers
- Example: User places custom rules in a directory, kantra mounts it when spawning providers
- For containerless Java provider, custom rules can be placed in a directory that the native provider reads

**IDE:**
- Custom rules support mechanism TBD (likely workspace or project-local directory)

**Custom Rules Structure:**
- Custom rules do not need to follow the `java/`, `nodejs/`, `dotnet/` directory structure
- They can be organized however the user prefers
- Provider loads all rule files recursively from `/opt/custom-rulesets/`

#### Kantra-Specific Handling

Kantra has unique requirements because:
- It bundles all providers but runs them in different ways
- Java provider can run without containers (direct binary execution)
- Other providers run in containers

**Current State:**
- Kantra's Dockerfile clones tackle2-seed and copies ALL `resources/rulesets/` to `/opt/rulesets` in the runner container
- At runtime, kantra extracts rulesets from the runner container to the host filesystem (e.g., `.rulesets-latest/`)
- The extraction happens in `pkg/provider/env_container.go:extractDefaultRulesets()` which copies `/opt/rulesets` from the runner container

**Proposed Changes:**
For kantra distribution:
- **Java rules (containerless mode)**: Bundle `tackle2-seed/resources/rulesets/java/` in the kantra ZIP distribution
  - At runtime, extract Java rules from the distribution to the host filesystem (e.g., `.rulesets-latest/java/`)
  - The Java provider runs natively and reads rules from the extracted location
- **Other provider rules (containerized mode)**: Bundled in their respective provider container images at `/opt/rulesets/`
  - Node.js provider container bundles `/opt/rulesets/` with Node.js rules
  - .NET provider container bundles `/opt/rulesets/` with .NET rules
- No longer bundle all rulesets in the runner container - only Java rules in the distribution for native execution
- Update extraction logic to extract Java rules from the kantra distribution (not from a container)

#### Builtin Provider Integration

Many rules combine conditions from the builtin provider with language-specific providers. For example:

```yaml
- ruleID: example-java-rule
  when:
    and:
      - builtin.filecontent:
          pattern: "SomeJavaPattern"
          filePattern: ".*\\.java"
      - java.referenced:
          pattern: "com.example.OldAPI"
```

This pattern will continue to work. The key principle:
- Rules are categorized by their primary language (Java, .NET, Node.js, etc.)
- Rules can use builtin provider conditions regardless of language
- The builtin provider itself doesn't have language-specific rules

#### Rulesets Repository Structure

The rulesets repository already has the desired structure:

```
rulesets/
├── stable/
│   ├── java/
│   │   ├── azure/
│   │   ├── camel3/
│   │   ├── eap7/
│   │   └── ...
│   ├── nodejs/
│   └── dotnet/
└── preview/
    └── (similar structure)
```

This structure will be leveraged with clear documentation about which directory maps to which provider.

#### Provider Configuration

Each provider Dockerfile will specify:
- tackle2-seed repository version/tag to use during build
- Which language-specific directory to bundle from `resources/rulesets/`
- Bundled rule location: `/opt/rulesets/`

Example Dockerfile snippet:
```dockerfile
FROM registry.access.redhat.com/ubi9-minimal as rulesets
ARG SEED_REF=v1.2.3
RUN git clone --branch ${SEED_REF} https://github.com/konveyor/tackle2-seed

FROM <base-image>
COPY --from=rulesets /tackle2-seed/resources/rulesets/java/ /opt/rulesets/
```

#### Migration Path for C# Provider

The C# (.NET) provider currently maintains rules in its own repository. The migration strategy:
1. This enhancement focuses on the general mechanism
2. C# provider continues with current approach initially
3. Separate effort to migrate C# rules to `rulesets/stable/dotnet/`
4. After migration, C# provider adopts the standard approach

### Security, Risks, and Mitigations

#### Risk: Rule Version Skew
**Description**: Different providers might run with different versions of rules, leading to inconsistent analysis results.

**Mitigation**: 
- Use explicit version pinning in provider builds
- Document recommended version combinations
- Consider version compatibility matrix in CI/CD

#### Risk: Breaking Changes in Rule Format
**Description**: Changes to rule schema or format could break providers.

**Mitigation**:
- Version the rule schema
- Implement schema validation in providers
- Clear deprecation and migration policies

#### Risk: Build Process Complexity
**Description**: Adding ruleset fetching to provider builds adds complexity.

**Mitigation**:
- Provide scripts/tooling to automate rule fetching
- Clear documentation for provider maintainers
- CI/CD integration to verify rule packaging

#### Risk: Contributor Confusion
**Description**: Contributors might not know where to add rules.

**Mitigation**:
- Clear documentation in rulesets repository README
- Directory structure matches provider names
- Contribution guidelines with examples

#### Security: Untrusted Ruleset Sources
**Description**: If runtime fetching is implemented, fetching from untrusted sources could be risky.

**Mitigation**:
- Default to official rulesets repository
- Verify signatures/checksums for fetched rules
- Use HTTPS and verify certificates
- Allow air-gapped deployments with pre-packaged rules

## Design Details

### Provider Changes

Each provider container image will need:
1. **Dockerfile updates**: 
   - Add build stage to clone tackle2-seed
   - Copy language-specific seeded rules to `/opt/rulesets/`
   - Example for Java: `COPY --from=rulesets /tackle2-seed/resources/rulesets/java/ /opt/rulesets/`
2. **Rule loader logic**: Load rules from two locations
   - `/opt/rulesets/` - Default bundled rules (always present in the container image)
   - `/opt/custom-rulesets/` - Custom user-provided rules (optional, via volume mount)
   - Custom rules are loaded in addition to default rules
3. **Documentation**: Document bundled rule location (`/opt/rulesets/`) and custom rules location (`/opt/custom-rulesets/`)

### Rulesets Repository Changes

The rulesets repository will need:
1. **Documentation**: Clear README explaining directory-to-provider mapping
2. **Contribution guide**: Instructions for adding rules to correct directory
3. **Validation**: CI/CD checks to ensure rules are in correct directories
4. **Tagging strategy**: Semantic versioning for rulesets releases

### Tackle2-Seed Repository Changes

The tackle2-seed repository will need:
1. **Versioning coordination**: Ensure tackle2-seed versions align with rulesets versions
2. **Index updates**: Add provider/language tags to `rulesets.yaml` if needed for filtering
3. **Documentation**: Clarify that language-organized directories are consumed by respective providers

### Kantra Changes

Kantra will need:
1. **Distribution packaging**:
   - Bundle Java rules from `tackle2-seed/resources/rulesets/java/` in the kantra ZIP distribution
   - No longer bundle all rulesets in the runner container
2. **Ruleset extraction logic** (`pkg/provider/env_container.go`):
   - Update `extractDefaultRulesets()` to extract Java rules from the kantra distribution (not from a container)
   - Extract to host filesystem for native Java provider execution
   - Containerized providers use their bundled rules at `/opt/rulesets/`, no extraction needed
3. **Custom rules support**:
   - Allow users to specify a custom rules directory (e.g., via CLI flag or configuration)
   - Mount custom rules from host filesystem to provider containers at `/opt/custom-rulesets:ro`
   - For containerless Java provider, provide custom rules directory path to the native provider
4. **Documentation**: Explain that Java rules are in the distribution for containerless mode, and containerized providers are self-contained with rules bundled at `/opt/rulesets/`, plus optional custom rules support

### Hub Changes

Tackle2-hub will need:
1. **Seeding process**: Continue seeding database with ALL rulesets for management/UI purposes
2. **Provider spawning**: When launching provider containers, simply run them
   - Providers use their bundled rules at `/opt/rulesets/`
   - No rule mounting needed for default rules - providers are self-contained
3. **Custom rules support**: Optionally mount custom rules when spawning providers
   - Example: `-v /path/to/custom/rules:/opt/custom-rulesets:ro`
   - Hub can manage custom rulesets and provide them to providers as needed
4. **API enhancements**: Add provider/language filtering to ruleset queries (optional)
5. **Documentation**: Update to reflect that providers are self-contained with bundled rules, with optional custom rules support

### Test Plan

#### Unit Tests
- Each provider correctly loads only its language-specific rules
- Builtin conditions work within language-specific rules
- Rule loading fails gracefully if rules are missing or malformed
- Providers correctly load custom rules from `/opt/custom-rulesets/`
- Custom rules are loaded in addition to default rules

#### Integration Tests
- Java provider runs only Java rules, not C# or Node.js rules
- Multi-provider analysis works correctly with separate rulesets
- Kantra correctly packages and runs Java rules
- Custom rules are correctly mounted and loaded in Hub deployments
- Custom rules are correctly mounted and loaded in Kantra deployments
- Custom rules work with both containerized and containerless (Java) providers

#### End-to-End Tests
- Analyze sample projects with each provider
- Verify only relevant rules are executed
- Test in IDE, Hub, and kantra environments
- Test custom rules workflow in Hub and Kantra
- Verify custom rules execute alongside default rules

#### Regression Tests
- Existing analysis results remain consistent after migration
- All current rules still execute correctly
- No loss of functionality

### Upgrade / Downgrade Strategy

#### Upgrading to Provider-Specific Rules

1. **Phase 1: Provider Updates**
   - Update each provider to support rule filtering/loading
   - Add backward compatibility to load all rules if new mechanism not configured
   - Deploy updated providers

2. **Phase 2: Rule Packaging**
   - Package language-specific rules with each provider
   - Providers still have access to all rules but prefer packaged ones

3. **Phase 3: Enforcement**
   - Remove all-rules packaging
   - Providers only have access to their language-specific rules

#### Downgrading

- If issues arise, providers can fall back to loading all rules
- Keep all-rules packaging available for one release cycle
- Document downgrade procedure in release notes

## Implementation History

- 2026-05-14: Initial proposal created

## Drawbacks

1. **Increased build complexity**: Each provider build must now fetch and package rules
2. **Potential duplication**: Rules might be duplicated across provider distributions
3. **Version coordination**: Need to coordinate rulesets version with provider versions
4. **Migration effort**: Existing providers need updates to adopt new mechanism
5. **Testing burden**: Must test rule packaging for each provider separately

## Alternatives

### Alternative 1: Rules in Each Provider Repository

**Description**: Move rules into each provider's repository instead of keeping them centralized.

**Pros**: 
- Simpler build process (no fetching needed)
- Tight coupling of provider and rules versions
- Clear ownership

**Cons**:
- Scattered rule definitions across many repositories
- Difficult to get overview of all available rules
- Harder for contributors to find where to add rules
- Potential for drift in rule formats and standards
- No single source of truth

**Decision**: Rejected because maintaining a central rulesets repository provides better contributor experience and rule consistency.

### Alternative 2: Dynamic Rule Filtering at Runtime

**Description**: Continue loading all rules but filter them at runtime based on provider type.

**Pros**:
- No build process changes needed
- Easy to implement
- Flexible filtering logic

**Cons**:
- Doesn't solve distribution bloat
- Still loads irrelevant rules into memory
- Filtering logic could be complex
- Doesn't prevent "run always" rules from executing incorrectly

**Decision**: Rejected because it doesn't address the root cause of distributing irrelevant rules.

### Alternative 3: Monorepo for Providers and Rules

**Description**: Combine all providers and rules into a single monorepo.

**Pros**:
- Atomic changes across providers and rules
- Simplified versioning
- Single CI/CD pipeline

**Cons**:
- Massive repository restructuring needed
- Harder to maintain separate provider release cycles
- Increased repository size and complexity
- Conflicts with current repository structure

**Decision**: Rejected due to significant restructuring cost and loss of modularity.

## Infrastructure Needed

1. **Rulesets Repository Tagging**: Establish semantic versioning and release process for rulesets
2. **Tackle2-Seed Repository Tagging**: Coordinate versioning with rulesets repository releases
3. **Seeding Pipeline**: Ensure seeding process runs after rulesets changes to update tackle2-seed
4. **Provider Build Tooling**: Scripts or tools to fetch and package seeded rules from tackle2-seed during provider builds
5. **CI/CD Updates**: Modify provider CI/CD pipelines to include rule packaging steps from tackle2-seed
6. **Documentation Site Updates**: Ensure documentation reflects provider-to-ruleset mapping, the seeding process, and custom rules support
7. **Testing Infrastructure**: 
   - Ability to test each provider with its specific ruleset in isolation
   - Test fixtures for custom rules scenarios
   - Integration tests for custom rules mounting in Hub and Kantra
