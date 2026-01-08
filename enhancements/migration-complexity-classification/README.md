---
title: migration-complexity-classification
authors:
  - "@tsanders-rh"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2025-11-02
last-updated: 2025-12-12
status: provisional
see-also:
  - "https://github.com/konveyor/enhancements/issues/254"
  - "https://github.com/tsanders-rh/konveyor-iq"
  - "https://tsanders-rh.github.io/konveyor-iq/"
replaces:
  - N/A
superseded-by:
  - N/A
---

# Migration Complexity Classification

Add a standardized complexity classification system to Konveyor rules to improve AI automation success rates, set appropriate expectations for different migration scenarios, and enable better migration planning and reporting.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. ~~Should the `migration_complexity` field be added to the Konveyor ruleset schema itself, or kept as evaluation-only metadata?~~ **Resolved:** Add to schema as optional field (follows pattern from AppCAT enhancement [#254](https://github.com/konveyor/enhancements/issues/254))
2. ~~Should we provide tooling to auto-classify existing rules, or require manual classification by rule authors?~~ **Resolved:** Hybrid approach with auto-classification + manual override
3. Are there other complexity factors we should consider beyond the five-level system (e.g., multi-file migrations, database schema changes)?
4. ~~How should this integrate with existing rule metadata (categories, severity levels, effort/story points)?~~ **Resolved:** Complexity is a complementary, orthogonal dimension (see Alternative #1 for detailed explanation of relationship to story points)
5. Should different complexity levels trigger different automation workflows or review processes in Konveyor Hub/CLI, or should this be left entirely to downstream tool implementers?
6. ~~How do we handle custom rules authored by end customers that aren't in the default ruleset?~~ **Resolved:** See "Custom Rules" section below

## Summary

This enhancement proposes adding a **migration complexity classification** field to Konveyor rules, using a five-level system (TRIVIAL, LOW, MEDIUM, HIGH, EXPERT) that categorizes migrations based on their difficulty and suitability for AI automation. This classification enables segmented reporting of AI success rates, better migration planning, and complexity-aware automation strategies.

Currently, AI performance on migrations is evaluated as a single metric (e.g., "50% success rate"), which obscures the reality that namespace changes succeed at 95%+ rates while security configurations may only succeed 30% of the time. By classifying migrations by complexity, we can demonstrate AI's true value at each difficulty level, focus automation on suitable tasks, and set appropriate expectations for migration tooling.

## Motivation

### Goals

1. **Maximize AI automation ROI** by identifying which migrations are suitable for full automation vs. requiring human review
2. **Enable better migration planning** by showing the distribution of complexity types and informing automation strategy decisions
3. **Provide segmented reporting** that shows AI value at each difficulty level rather than a single averaged metric
4. **Set appropriate expectations** for different migration types (automation for TRIVIAL, scaffolding for EXPERT)
5. **Focus human expertise** where it's truly needed by routing simple migrations to automation pipelines
6. **Demonstrate clear ROI** for AI-assisted migration tooling with data-driven insights

### Non-Goals

1. Requiring manual classification for all existing rules (we provide automated classification with review/override capability)
2. Replacing existing rule metadata (severity, category, effort/story points) - complexity is complementary
3. Mandating specific automation workflows based on complexity (this is left to implementers)
4. Classifying non-migration rules or analysis-only rules
5. Creating complexity-specific migration prompts or guidance (future enhancement)

### Current Problem

When evaluating AI-generated migrations as a single aggregate metric, we obscure important patterns:

**Before classification:**
```
Overall Pass Rate: 50% (116/234 failures)
```
❌ Conclusion: "AI isn't ready for production migrations"

**After classification:**
```
TRIVIAL: 95% pass rate (19/20)   ← Automate these!
LOW:     82% pass rate (41/50)   ← Automate with review
MEDIUM:  64% pass rate (96/150)  ← Good acceleration
HIGH:    45% pass rate (8/18)    ← Useful scaffolding
EXPERT:  20% pass rate (1/5)     ← AI generates checklist
```
✅ Conclusion: "AI excels at 60% of migrations, assists with the rest"

### Proof of Concept Results

The classification system has been validated on **2,680+ Konveyor rules** with AI-assisted migration testing. The results demonstrate clear differentiation between complexity levels:

| Complexity | Rules Classified | Representative Examples |
|------------|------------------|-------------------------|
| **TRIVIAL** | 847 (32%) | `javax.*` → `jakarta.*` namespace changes |
| **LOW** | 1,203 (45%) | `@Stateless` → `@ApplicationScoped`, simple annotation swaps |
| **MEDIUM** | 456 (17%) | JMS → Reactive Messaging, context-aware API migrations |
| **HIGH** | 142 (5%) | Spring Security → Quarkus Security, architectural changes |
| **EXPERT** | 32 (1%) | Custom security realms, distributed transaction coordinators |

**Key Insights:**
- **77% of rules** (TRIVIAL + LOW) are highly suitable for AI automation
- **17% of rules** (MEDIUM) benefit from AI acceleration with developer guidance
- **6% of rules** (HIGH + EXPERT) require significant human expertise but AI can scaffold

**Interactive Demo:** https://tsanders-rh.github.io/konveyor-iq/

This distribution pattern has been validated across multiple migration scenarios (Spring Boot → Quarkus, Java EE → Jakarta EE, Patternfly 5 → 6) with consistent results.

## Proposal

### User Stories

#### Story 1: Migration Planner
As a **migration planner**, I want to see the complexity distribution of rules affecting my application so that I can estimate the effort required and determine which migrations can be automated vs. requiring expert review.

#### Story 2: AI Tool Developer
As an **AI migration tool developer**, I want to know the complexity level of each rule so that I can route TRIVIAL/LOW migrations to automated pipelines and flag HIGH/EXPERT migrations for human review.

#### Story 3: Migration Report Consumer
As a **stakeholder reviewing migration reports**, I want to see AI success rates segmented by complexity so that I can understand where AI provides value and justify investment in migration tooling.

#### Story 4: Rule Author
As a **rule author**, I want to classify my rules by migration complexity so that users understand the expected difficulty and AI tooling can handle them appropriately.

### Implementation Details/Notes/Constraints

#### Alignment with AppCAT Metadata Enhancement

This proposal follows the same optional metadata pattern established by the AppCAT enhancement ([#254](https://github.com/konveyor/enhancements/issues/254)), which adds optional `message`, `domain`, `category`, and `metadata.confidence` fields to improve rule precision and reduce false positives. Both enhancements share key principles:

- **Optional Fields**: `migration_complexity` is optional; existing rules without it remain fully valid
- **Backward Compatible**: No breaking changes; tools that don't support the field simply ignore it
- **Single Source of Truth**: Konveyor remains the upstream authority while enabling richer downstream tooling
- **Gradual Adoption**: Allows ecosystem tools to adopt at their own pace

Like the AppCAT enhancement focuses on reducing false positives through better detection confidence, this enhancement focuses on improving AI automation success rates through complexity-aware routing—complementary goals that enrich Konveyor's metadata without fragmenting the schema.

**Note on Confidence**: While AppCAT's `metadata.confidence` field indicates detection reliability (how confident the rule correctly matches a pattern), our classification process uses confidence scores internally in the tooling layer to guide review workflows, but does not expose them in the rule schema. See "Classification Review Tooling" below for details.

#### Five-Level Classification System

| Level | Description | Example Rules | Expected AI Success | Use Case |
|-------|-------------|---------------|---------------------|----------|
| **TRIVIAL** | Mechanical find/replace operations | `javax.*` → `jakarta.*`<br/>`java.util.Date` → `java.time.LocalDate` | 95%+ | Full automation |
| **LOW** | Straightforward API equivalents | `@Stateless` → `@ApplicationScoped`<br/>`@WebServlet` → `@Path` | 80%+ | Automation with light review |
| **MEDIUM** | Requires understanding context | JMS → Reactive Messaging<br/>`web.xml` → annotations | 60%+ | AI acceleration + developer completion |
| **HIGH** | Architectural changes | Spring Security → Quarkus Security<br/>EJB transactions → CDI interceptors | 30-50% | AI scaffolding + expert review |
| **EXPERT** | Domain-specific expertise required | Custom security realms<br/>Performance-critical algorithms<br/>Distributed transaction coordinators | <30% | AI checklist + human migration |

#### Schema Changes

Add an optional `migration_complexity` field to the rule schema:

```yaml
rules:
  - id: "javax-to-jakarta-00001"
    description: "Replace javax.* imports with jakarta.*"
    migration_complexity: trivial  # OPTIONAL - enhances AI automation routing & planning
    category: mandatory
    # ... rest of rule definition

  - id: "legacy-rule-without-complexity"
    description: "Some older rule without complexity classification"
    # migration_complexity field not present - defaults to MEDIUM in tooling
    category: mandatory
    # ... rest of rule definition
```

**Key Points:**
- Field is **optional** - existing rules remain valid without it
- Tools that don't support complexity classification simply ignore the field
- Unclassified rules default to MEDIUM complexity (conservative, safe default)

#### Classification Algorithm

An automated classification tool will analyze rules using a heuristic-based approach. The detailed algorithm will be refined during implementation, but the general strategy includes:

1. **Keyword Analysis**: Scan rule descriptions, code snippets, and expected fixes for complexity indicators
   - Security-related keywords (`security`, `authentication`, `authorization`, `encryption`) → HIGH/EXPERT
   - Messaging patterns (`JMS`, `MQ`, `messaging`, `reactive`) → MEDIUM/HIGH
   - Simple namespace changes (`javax` → `jakarta`, package renames) → TRIVIAL
   - Configuration keywords (`xml`, `properties`, `yaml` transformations) → LOW/MEDIUM

2. **Import Analysis**: Count and categorize imports in test cases
   - Pure package renames (`javax.*` → `jakarta.*`) → TRIVIAL
   - Framework migrations (Spring → Quarkus, Java EE → Jakarta EE) → MEDIUM/HIGH
   - Security/transaction/persistence imports → HIGH
   - Ratio of imports changed vs. logic changed → complexity indicator

3. **Annotation Complexity**: Identify patterns in annotation changes
   - Simple 1:1 swaps (`@Stateless` → `@ApplicationScoped`) → LOW
   - Configuration annotations (`@EnableWebSecurity`, `@Configuration`) → HIGH
   - Behavioral annotations (lifecycle, transaction boundaries) → MEDIUM/HIGH

4. **Multi-file Coordination**: Detect cross-cutting concerns from rule metadata
   - Single-file changes with localized impact → TRIVIAL/LOW
   - Cross-cutting concerns (security policies, transaction strategies) → HIGH/EXPERT
   - Configuration-to-code migrations → MEDIUM

5. **Test Case Analysis**: Examine complexity of expected fixes
   - Line-for-line replacements → TRIVIAL
   - Structural changes with similar patterns → LOW/MEDIUM
   - Algorithmic or architectural changes → HIGH/EXPERT

**Note**: The classification algorithm generates confidence scores (0.0-1.0) for each suggestion. Low-confidence classifications default to MEDIUM and are flagged for manual review (see "Classification Review Tooling" below).

#### Manual Override

Rule authors can explicitly set complexity to override automated classification:

```yaml
- id: "custom-security-realm"
  migration_complexity: expert  # Manual override
```

#### Custom Rules and End-User Authored Rules

Organizations often create custom migration rules specific to their internal frameworks or patterns. These custom rules need complexity classification but aren't part of the default Konveyor ruleset.

**Hybrid Approach for Custom Rules:**

1. **Automatic Heuristic Classification**
   - Apply the same classification algorithm to custom rules
   - Analyze descriptions, code patterns, imports, and annotations
   - Generate confidence score for classification

2. **Default to MEDIUM on Low Confidence**
   - If confidence score falls below threshold (e.g., <70%), default to MEDIUM
   - Conservative assumption: neither trivial nor expert
   - Prevents over-automation of unknown patterns

3. **Explicit Override Capability**
   - Custom rule authors add `migration_complexity` field when they have domain knowledge
   - Manual classification takes precedence over heuristic
   - Recommended for high-stakes migrations (security, performance-critical)

4. **Classification Review Tooling**
   - Provide CLI/UI tool to review auto-classifications with transparency

   **Confidence Scoring (Internal Tooling Only):**
   - Classification algorithm generates confidence scores (0.0 - 1.0)
   - High confidence (>0.85): Auto-approve classification
   - Medium confidence (0.70-0.85): Flag for review
   - Low confidence (<0.70): Default to MEDIUM, require manual classification

   **Review Interface:**
   - Show confidence scores alongside suggested classifications
   - Highlight low-confidence classifications for priority review
   - Allow batch approval of high-confidence classifications
   - Track classification history and overrides
   - Report unclassified rules separately in metrics

   **Example Output:**
   ```
   Rule: javax-to-jakarta-00001
   Suggested: TRIVIAL (confidence: 0.95) ✓ Auto-approved

   Rule: custom-security-realm
   Suggested: HIGH (confidence: 0.68) ⚠️  Review needed

   Rule: unknown-pattern
   Suggested: MEDIUM (confidence: 0.45) ⚠️  Default applied
   ```

   **Important**: Confidence scores are used during classification workflow but **not stored in rule schema**, keeping rules clean and maintainable. This differs from AppCAT's `metadata.confidence` which indicates detection reliability and is stored in the schema.

**Example Custom Rule:**

```yaml
- id: "internal-cache-framework-migration"
  description: "Migrate from internal CacheManager to Quarkus Cache"
  migration_complexity: medium  # Explicit classification by author
  # Auto-classification would default to MEDIUM with ~65% confidence
  category: mandatory
  test_cases:
    - original_code: |
        @Inject InternalCacheManager cache;
      expected_fix: |
        @Inject Cache cache;
```

**Benefits:**
- Custom rules get classified automatically with minimal author effort
- Authors retain control for domain-specific knowledge
- Reporting remains consistent across default and custom rules
- Low-risk defaults prevent automation surprises

### Security, Risks, and Mitigations

**Risk: Incorrect Classification**
- **Mitigation**: Provide manual override capability, allow community review of classifications, start with conservative (higher complexity) defaults

**Risk: Over-reliance on Automation**
- **Mitigation**: Clear documentation that TRIVIAL/LOW classifications indicate higher automation suitability, not guaranteed success; always recommend review

**Risk: Schema Fragmentation**
- **Mitigation**: Propose addition to official Konveyor rule schema with community review; provide backward compatibility for rules without complexity field

**Security Impact**
- No direct security implications; this is metadata for planning and reporting
- Indirectly improves security by flagging complex security migrations for expert review

## Design Details

### Test Plan

1. **Unit Tests**
   - Test classification algorithm against known rule patterns
   - Verify each complexity level is correctly identified
   - Test manual override functionality

2. **Integration Tests**
   - Classify all existing Konveyor rules
   - Validate classifications against human expert review for sample set
   - Test reporting functionality with segmented results

3. **Validation Metrics**
   - Measure AI success rates by complexity level across multiple models
   - Verify expected success rate ranges match empirical results
   - Compare automated classification vs. manual expert classification

4. **User Acceptance Testing**
   - Collect feedback from migration practitioners on classification accuracy
   - Validate that complexity levels align with real-world difficulty

### Upgrade / Downgrade Strategy

#### Backward Compatibility Guarantees

**Konveyor Remains Single Source of Truth:**
- The `migration_complexity` field is **OPTIONAL**
- Existing rules without this field remain fully valid
- All Konveyor tools continue to function normally
- Schema change is additive only—no breaking changes

**Downstream Tool Flexibility:**
- Tools that don't support complexity classification simply ignore the field
- Tools that support it can provide enhanced automation routing and reporting
- No functional impact on rule execution or analysis
- Each tool in the ecosystem can adopt at its own pace

**Conservative Defaults:**
- Unclassified rules default to MEDIUM complexity (safe, conservative)
- Prevents over-automation of unknown patterns
- Allows gradual classification of existing rulesets
- Tools can override default behavior if needed

**Upgrade Path (Adding Complexity Classification):**
- Provide migration script to classify existing rulesets
- High-confidence auto-classifications approved automatically
- Low-confidence classifications flagged for review
- Rule authors can override any auto-classification

**Downgrade Path (Removing/Ignoring Complexity):**
- Simply stop reading the `migration_complexity` field
- No migration needed—field is ignored
- Reports fall back to aggregate metrics
- Zero disruption to existing workflows

### Phased Implementation

#### Phase 1: Schema and Classification
1. Propose `migration_complexity` field addition to Konveyor rule schema
2. Develop classification algorithm and tooling
3. Classify existing rulesets with automated tool
4. Community review of classifications

#### Phase 2: Reporting and Tooling Integration
1. Update evaluation frameworks to segment results by complexity
2. Add complexity badges to rule listings/documentation
3. Generate recommendations per complexity tier

#### Phase 3: Complexity-Aware Workflows (Optional)
1. Route TRIVIAL/LOW migrations to automated pipelines
2. Add human review checkpoints for HIGH/EXPERT
3. Customize prompts and guidance per complexity level

## Implementation History

- 2025-12-12: Major revision for clarity and impact:
  - Added "Proof of Concept Results" section with distribution data from 2,680+ classified rules
  - Enhanced five-level classification table with concrete rule examples
  - Expanded classification algorithm with 5 detailed heuristics
  - Resolved open question #4: Complexity is orthogonal to existing metadata
  - Refined open question #5: Clarified scope of automation workflow question
  - Fixed Goals #1 and #2 wording for clarity (maximize ROI, inform strategy)
  - Clarified Non-Goals #1 and #2 to avoid contradictions
  - Removed time estimates from phased implementation plan
- 2025-12-12: Added "Use Existing Effort Field (Story Points)" to Alternatives section explaining why story points are insufficient for AI automation routing
- 2025-12-12: Aligned proposal with AppCAT metadata enhancement pattern ([#254](https://github.com/konveyor/enhancements/issues/254))
- 2025-12-12: Clarified confidence scoring approach (tooling-only, not in schema)
- 2025-12-12: Resolved open question #1: Add field to schema as optional metadata
- 2025-11-02: RFE issue created: https://github.com/konveyor/enhancements/issues/245
- 2025-11-02: Initial proposal created based on konveyor-iq evaluation framework
- 2025-10-22: Proof-of-concept demonstrated with 2,680+ rules classified
- 2025-10-22: Sample report published showing segmented results: https://tsanders-rh.github.io/konveyor-iq/

## Drawbacks

1. **Additional Metadata Burden**: Rule authors must classify rules, adding overhead to rule creation
   - Counter: Automated classification tool reduces burden; manual override only when needed

2. **Subjectivity**: Complexity assessment may be subjective and vary by user skill level
   - Counter: Clear criteria and examples for each level; community review process

3. **Schema Changes**: Adding fields to rule schema requires coordination across Konveyor projects
   - Counter: Optional field maintains backward compatibility; gradual adoption possible

4. **Maintenance**: Classifications may become outdated as frameworks evolve
   - Counter: Version-specific classifications could be added in future enhancement

## Alternatives

### Alternative 1: Use Existing Effort Field (Story Points)
Konveyor rules already have an `effort` field that represents the number of expected story points to fix an issue ([Rules Development Guide](https://docs.redhat.com/en/documentation/migration_toolkit_for_applications/8.0/html-single/rules_development_guide/index#how_story_points_are_estimated_in_rules)). Story points are an abstract metric commonly used in Agile development to estimate the level of effort needed to implement a feature or change. Why not use this instead of adding a new `migration_complexity` field?

**Rejected because:**

1. **Different Purpose**: Story points measure effort/work required (human planning metric for resource allocation), while complexity measures suitability for AI automation and migration difficulty. A namespace change might have `effort: 1` (low story points for quick manual fix) and be TRIVIAL for AI, while a custom security realm might also have `effort: 1` (small code change) but be EXPERT complexity due to requiring deep domain knowledge and security expertise.

2. **No AI Success Rate Correlation**: Story point values don't correlate with AI success rates. A 3-point task could be 95% successful for AI (simple refactoring) or 20% successful (complex architecture decision), depending on the nature of the work. Story points measure human effort, not automation suitability.

3. **Abstract Metric Not Designed for AI**: Story points are intentionally abstract and designed for human sprint planning, not AI automation routing. They don't translate to automation success rates or indicate which automation strategy to use.

4. **Doesn't Indicate Automation Type**: Story points don't tell you whether to:
   - Fully automate (TRIVIAL: 95%+ success)
   - Auto-generate with review (LOW: 80%+ success)
   - Use AI for scaffolding (HIGH: 30-50% success)
   - Generate checklist only (EXPERT: <30% success)

5. **Team-Dependent Subjectivity**: Story points are relative to team skill level and environment. What's `effort: 3` for one team might be `effort: 1` for another. Complexity classification is more objective: a namespace change is TRIVIAL regardless of team experience.

6. **Missing Context on Failure Modes**: Story points don't capture WHY something is complex (security implications, architectural changes, performance-critical code, distributed systems concerns) or what makes it unsuitable for AI automation.

**Example Showing the Difference:**

```yaml
# Example 1: Low story points, low complexity
- id: "javax-to-jakarta-namespace"
  effort: 1              # 1 story point - quick mechanical change for humans
  migration_complexity: trivial  # Perfect for AI automation (95%+ success)
  # Low effort for humans, even easier for AI

# Example 2: Low story points, high complexity
- id: "custom-security-realm"
  effort: 1              # 1 story point - small code change
  migration_complexity: expert   # Requires deep expertise (20% AI success)
  # Why expert? Security implications, custom logic, testing requirements
  # Low effort for experienced human, extremely difficult for AI

# Example 3: High story points, medium complexity
- id: "jms-to-reactive-messaging"
  effort: 5              # 5 story points - many files to update
  migration_complexity: medium   # AI can help but needs guidance (60% success)
  # High effort due to scale, but pattern is understandable to AI

# Example 4: Same story points, different complexity
- id: "spring-boot-to-quarkus-controller"
  effort: 3              # 3 story points
  migration_complexity: low      # Straightforward API mapping (80% AI success)

- id: "custom-transaction-manager"
  effort: 3              # Also 3 story points
  migration_complexity: expert   # Complex distributed systems logic (15% AI success)
```

**Note on Relationship**: While story points and complexity are independent dimensions, there may be loose correlations in some cases (e.g., many TRIVIAL migrations have low story points). However, this correlation is not reliable enough to use story points as a proxy for complexity.

**Conclusion**: `effort` (story points) and `migration_complexity` are complementary fields serving different purposes:
- **Story points**: Human resource planning and sprint estimation
- **Complexity**: AI automation routing, expectation setting, and tooling strategy

### Alternative 2: Rule Categories Only
Use existing rule categories (mandatory, optional, potential) as a proxy for complexity.

**Rejected because:** Categories describe rule importance, not migration difficulty. A mandatory namespace change is TRIVIAL, while a mandatory security migration is EXPERT.

### Alternative 3: Severity Levels
Repurpose or extend existing severity levels (critical, high, medium, low) for complexity.

**Rejected because:** Severity describes impact of not migrating, not difficulty of migration. Mixing concepts creates confusion.

### Alternative 4: AI Confidence Scores
Let AI models report confidence scores instead of rule-level classification.

**Rejected because:** Model-specific and doesn't help with planning. Users need upfront understanding of difficulty, not post-hoc confidence.

### Alternative 5: Multi-Dimensional Classification
Use multiple dimensions (technical complexity, business logic complexity, testing complexity, etc.).

**Rejected because:** Too complex for initial implementation. Single dimension captures primary use case. Could be future enhancement.

### Alternative 6: Three-Level System (Easy, Medium, Hard)
Simplified three-level classification.

**Rejected because:** Doesn't provide enough granularity. TRIVIAL (95%+ automation) vs. LOW (80% automation) distinction is valuable. EXPERT (human-led) vs. HIGH (AI-scaffolded) distinction is critical.

## Infrastructure Needed

1. **GitHub Repository for Enhancement Proposal**
   - Location: `/enhancements/migration-complexity-classification/` in konveyor/enhancements

2. **Classification Tooling Repository**
   - Initially in konveyor-iq (https://github.com/tsanders-rh/konveyor-iq)
   - May migrate to official Konveyor tooling if adopted

3. **Documentation Site Updates**
   - Add complexity classification documentation to Konveyor docs
   - Update rule authoring guides

4. **Community Review Process**
   - Forum/mailing list discussion on classification criteria
   - Review meetings for schema changes

5. **Sample Classified Rulesets**
   - Reference implementations showing classified rules
   - Published reports demonstrating segmented results
