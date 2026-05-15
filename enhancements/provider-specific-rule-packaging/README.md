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
- Support the builtin provider as a helper provider that can be used by any language-specific provider
- Ensure the solution works seamlessly across IDE, Hub, and CLI (kantra) deployments
- Make it easy for contributors to add or modify rules without needing to understand complex distribution mechanics
- Provide clear guidelines for which rules belong to which provider

### Non-Goals

- Support for rules that require multiple providers (beyond the builtin + one language provider combination)
- Migration of existing C# provider rules from their current repository to the rulesets repository (can be addressed separately)
- Environment-specific distribution strategies (IDE vs Hub vs CLI) - this will be addressed in a follow-up enhancement

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
As a user with a project containing both Java and Node.js code, I want to run both providers and have each one evaluate
only the rules relevant to its language, while both can leverage builtin provider conditions.

#### Story 4: IDE User
As a user of the Konveyor IDE extension, I want provider-specific rule packaging to work transparently, with no
additional configuration required on my part.

#### Story 5: Kantra User
As a user running kantra for Java analysis, I want the bundled Java provider to include all Java rules without requiring
external network access or additional downloads.

### Implementation Details/Notes/Constraints

#### Rule Source of Truth

The `rulesets` repository will remain the authoritative source for all rule definitions. This provides:
- Centralized rule management and versioning
- Single location for rule contributions and reviews
- Consistent rule quality and standards
- Clear visibility into all available rules

The `tackle2-seed` repository will also remain the single source of truth for the distribution of rules. The provider-specific rule packaging
will occur **after** the seeding process, meaning providers will fetch their language-specific rules from the
seeded output in tackle2-seed, not directly from the rulesets repository.

#### Rule Distribution Mechanism

Rules will be distributed to providers through one of these mechanisms (to be determined before implementation):

**Option A: Build-Time Vendoring**
- During provider build process, fetch the relevant seeded ruleset directory from tackle2-seed
- Embed the rules as resources in the provider binary or container image
- Providers reference a specific tackle2-seed repository version/tag
- Pros: No runtime dependencies, offline operation, version consistency, includes all metadata
- Cons: Providers must be rebuilt to pick up rule changes

**Option B: Runtime Fetching**
- Providers fetch their seeded rules at initialization time from tackle2-seed
- Could use git clone, HTTP download, or dedicated API
- Configurable tackle2-seed repository location and version
- Pros: Rule updates without provider rebuilds
- Cons: Requires network access, potential version skew issues

**Option C: Hybrid Approach**
- Default rules embedded at build time (Option A)
- Optional runtime override mechanism (Option B)
- Best of both worlds for different deployment scenarios

The initial implementation should start with **Option A** for simplicity and reliability, with Option C as a potential future enhancement.

#### Kantra-Specific Handling

Kantra has unique requirements because:
- It bundles all providers but runs them in different ways
- Java provider can run without containers (direct binary execution)
- Other providers run in containers

For kantra:
- **Java rules**: Package directly with the kantra binary (since Java provider runs natively)
- **Other provider rules**: Include in each provider's container image
- Each provider container is responsible for including its own ruleset at build time

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

Each provider will need configuration to specify:
- Which seeded ruleset directory it consumes
- tackle2-seed repository version/tag to use (for build-time vendoring)
- Any rule filtering or inclusion/exclusion patterns

Example configuration (format TBD):
```yaml
provider:
  name: java-external-provider
  ruleset:
    source: konveyor/tackle2-seed
    version: v1.2.3
    directory: resources/rulesets/java  # seeded rules
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

Each provider repository will need:
1. **Build script updates**: Add rule fetching/vendoring step
2. **Configuration file**: Specify ruleset source and version
3. **Rule loader changes**: Load only embedded/configured rules instead of all rules
4. **Documentation**: Explain which rules the provider uses

### Rulesets Repository Changes

The rulesets repository will need:
1. **Documentation**: Clear README explaining directory-to-provider mapping
2. **Contribution guide**: Instructions for adding rules to correct directory
3. **Validation**: CI/CD checks to ensure rules are in correct directories
4. **Tagging strategy**: Semantic versioning for rulesets releases

### Tackle2-Seed Repository Changes

The tackle2-seed repository will need:
1. **Seeding process updates**: Potentially update to support language-specific seeding
2. **Index updates**: The `rulesets.yaml` index may need language/provider tags to support filtering

### Kantra Changes

Kantra will need:
1. **Build process**: Package Java rules directly
2. **Container builds**: Ensure each provider container includes its rules
3. **Documentation**: Explain bundled vs. containerized provider rule handling

### Test Plan

#### Unit Tests
- Each provider correctly loads only its language-specific rules
- Builtin conditions work within language-specific rules
- Rule loading fails gracefully if rules are missing or malformed

#### Integration Tests
- Java provider runs only Java rules, not C# or Node.js rules
- Multi-provider analysis works correctly with separate rulesets
- Kantra correctly packages and runs Java rules

#### End-to-End Tests
- Analyze sample projects with each provider
- Verify only relevant rules are executed
- Test in IDE, Hub, and kantra environments

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
6. **Documentation Site Updates**: Ensure documentation reflects provider-to-ruleset mapping and the seeding process
7. **Testing Infrastructure**: Ability to test each provider with its specific ruleset in isolation
