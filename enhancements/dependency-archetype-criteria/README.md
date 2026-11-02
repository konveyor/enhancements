---
title: dependency-based-archetype-criteria
authors:
  - "@jortel"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-10-30
last-updated: 2026-10-30
status: provisional
see-also:
  - "/enhancements/assessment-module/README.md"
---

# Dependency-Based Archetype Criteria

Archetypes currently classify applications using Criteria Tags alone. An application
is a member of an archetype when the archetype's criteria tags are a subset of the
application's tags. This works well when the portfolio has been curated with tags, but
it cannot express the single most objective fact analysis already discovers about an
application: what it depends on.

This enhancement proposes **dependency criteria** — a boolean expression over the
dependencies reported by analysis — that may be used alone or in combination with
Criteria Tags to determine archetype membership.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

An archetype gains an optional dependency expression, matched against the dependencies
reported by the application's most recent analysis:

```
org.springframework.boot:spring-boot@2
  && (javax.servlet:javax.servlet-api || javax.persistence:javax.persistence-api)
  && !jakarta.servlet:jakarta.servlet-api
```

The expression language has two parts. A **dependency reference** names a dependency —
with wildcards permitted for name segments — and optionally constrains its version.
A **boolean layer** composes references with `&&`, `||`, `!`, and parentheses. An
application is a member of the archetype when its criteria tags match (as today) **and**
the dependency expression is satisfied.

The syntax is one language across all ecosystems rather than per-ecosystem dialects, and
the user never declares which ecosystem an expression targets. Version *ordering* does
genuinely differ between ecosystems, but every dependency row already carries its own
`provider`, so the comparator is selected per candidate during matching. A single
expression may therefore reference Java, Go, Python, and npm dependencies together.

## Motivation

Tag-based criteria depend on the portfolio being tagged accurately and consistently,
which in practice is the slowest and least reliable part of onboarding a large estate.
Tags are applied by people, drift as applications change, and are frequently absent
entirely for applications that have only just been discovered.

Analysis, by contrast, already produces a precise, machine-generated inventory of every
dependency of every analyzed application, refreshed on every run. Archetypes are the
mechanism by which Konveyor groups applications for assessment, review, and target
profiles — so the inability to say "every application on Spring Boot 2" or "everything
still pulling a vulnerable Jackson" forces users to maintain by hand a classification
the tool could derive.

Dependency data also stays current on its own. Discovery is triggered by an edit to an
application, so once discovery reports dependencies
([Dependency Data from Discovery](#dependency-data-from-discovery)) an application's
dependency set is refreshed whenever the application changes. Criteria built on it keep
classifying correctly across edits, where tag criteria depend on someone noticing the
change and re-tagging.

The cases users describe most often:

- **Framework and version cohorts.** "All Spring Boot 2 applications", in order to
  assess the Boot 3 migration as a group.
- **Platform coupling.** "Anything importing WebLogic or WebSphere packages", which is
  rarely tagged but always visible in the dependency list.
- **Vulnerability and EOL sweeps.** "Everything on a Jackson older than 2.13", scoped to
  a wave rather than chased application by application.
- **Mixed criteria.** "Business-critical applications (tag) that are also on Struts 2
  (dependency)" — the combination is the useful query, and neither half expresses it.

### Goals

- Define a dependency expression syntax that is readable, validatable, and
  **language-agnostic** — usable for Java, Go, Python, and npm dependencies without
  per-ecosystem dialects.
- Define version constraint semantics, including ranges and wildcards, that behave
  correctly against the version strings each ecosystem actually produces.
- Extend the Archetype model and API to store dependency criteria alongside Criteria
  Tags, without breaking existing clients.
- Make dependency data available from discovery runs, so criteria work on a portfolio
  that has been onboarded but not fully analyzed.
- Define membership semantics when dependency criteria are combined with Criteria Tags,
  and how they interact with the existing archetype specificity rules.
- Keep membership resolution within its current performance envelope for large
  portfolios.

### Non-Goals

- **A general-purpose query language.** This matches dependencies, not incidents, facts,
  or arbitrary application attributes.
- **Native ecosystem range dialects.** Maven `[1.2,1.3)`, npm `^1.2`, and PEP 440
  `>=1.2,<2.0` are explicitly not accepted. See [Alternatives](#alternatives).
- **Dependency resolution.** The hub matches what analysis reported; it does not resolve
  or walk a dependency graph.
- **SBOM ingestion** as an alternative source of dependency data.
- **Tag inference from dependencies.** Automatically tagging applications based on
  dependencies is a separate concern and is out of scope.

## Proposal

### User Stories

#### DAC001

*As an Architect, I want to define an archetype by the dependencies its applications
use, so that applications are classified without relying on manual tagging.*

#### DAC002

*As an Architect, I want to combine dependency criteria with Criteria Tags, so that I
can express "business-critical applications that are also on Struts 2".*

#### DAC003

*As an Architect, I want to constrain a dependency to a version or a range of versions,
so that I can separate applications on Spring Boot 2 from those already on 3.*

#### DAC004

*As an Architect, I want to use wildcards in dependency names, so that I can match a
whole package namespace without enumerating artifacts.*

#### DAC005

*As an Architect, I want to see which applications an archetype's criteria currently
match before I save it, so that I can tell whether the expression says what I meant.*

#### DAC006

*As an Architect working on a polyglot portfolio, I want the same expression syntax for
Java, Go, Python, and npm dependencies, so that I learn one language rather than four.*

### Expression Syntax

#### Boolean layer

```
expr = ref | '(' expr ')' | '!' expr | expr '&&' expr | expr '||' expr
```

`&&` binds tighter than `||`. This mirrors the addon/extension selector already
implemented in `internal/task/selector.go` and uses the same parser.

#### Dependency reference

```
ref        = name [ ('@' | ':') constraint ]
name       = segment (('.' | ':' | '/') segment)*
segment    = text | '*' | '**'
```

The name is a sequence of **segments**. `.`, `:`, and `/` are all segment separators,
which is what makes one syntax work across ecosystems — the hub normalizes both the
pattern and the stored dependency name the same way:

| provider | reported name | segments |
|---|---|---|
| java | `com.fasterxml.jackson.core:jackson-core` | `com fasterxml jackson core jackson-core` |
| java | `com.fasterxml.jackson.core.jackson-core` | *identical* |
| go | `github.com/jortel/go-utils/v2` | `github com jortel go-utils v2` |
| python | `zope.interface` | `zope interface` |
| npm | `@babel/core` | `@babel core` |

`*` matches exactly one segment; `**` matches zero or more. Because Maven's
group/artifact colon normalizes to the same thing as a dot, the common Java form
`group:artifact` works without the hub needing to know whether the analyzer emitted a
colon or a dot.

Either `@` or `:` introduces the version constraint. `@` is unambiguous and idiomatic
for Go and npm; a trailing `:` field is treated as a constraint only when it looks like
a version (leading digit, `v` + digit).

#### Version constraint

A constraint is a comma-separated enumeration of terms; a version satisfying any term
satisfies the constraint. Every term is built from a **version prefix**, and a prefix
denotes the entire version line beneath it:

| term | meaning | matches | does not match |
|---|---|---|---|
| `2` | the 2 line | `2.0.0`, `2.7.18` | `1.9.9`, `3.0`, `20.1` |
| `2.13` | the 2.13 line | `2.13`, `2.13.7` | `2.12.9`, `2.14.0` |
| `1.2+` | that line or greater | `1.2.0`, `9.0` | `1.1.9` |
| `..2.12` | that line or lower | `2.12.7`, `1.0` | `2.13.0` |
| `2.9..2.13` | both ends inclusive of the line | `2.9.0`, `2.13.5` | `2.8.9`, `2.14` |
| *omitted* | any version | everything | — |

Prefix semantics is the reason `3.4.*` from the originating issue is written `3.4` here.
It also makes range endpoints behave as users read them: `..2.12` covers `2.12.7`, which
an exact-endpoint reading would exclude.

**Prereleases do not belong to their line; post-releases do.** `2.13` matches `2.13-sp`
and `2.13.post1` but not `2.13.0-rc1` or `2.13-SNAPSHOT`. This matches the convention
that `>=2.13.0` excludes prereleases of `2.13.0`.

#### Examples

```
# Spring Boot 2 → 3 migration cohort
org.springframework.boot:spring-boot@2
  && org.springframework.boot:spring-boot-starter-web@2.0..2.7
  && com.fasterxml.jackson.core:jackson-databind@2.9..2.13
  && (javax.servlet:javax.servlet-api || javax.persistence:javax.persistence-api)
  && !jakarta.servlet:jakarta.servlet-api
```

```
# Legacy application server coupling
weblogic:** || com.ibm.websphere:** || org.jboss.as:** || com.sun.*:**
```

```
# Stranded on an old major — in Go the major version is part of the module path
(github.com/golang-jwt/jwt/v4 && !github.com/golang-jwt/jwt/v5)
  || (github.com/go-chi/chi && !github.com/go-chi/chi/v5)
```

```
# Vulnerable pins
pyyaml@..5.3 || jinja2@..2.11 || urllib3@..1.26 || cryptography@..3.3
```

```
# Polyglot: a Java service with a legacy React front end.
# One expression spans both ecosystems; no provider is declared.
org.springframework.boot:spring-boot@2 && react@16
```

### Model and API Changes

One column on `archetype`:

```go
type Archetype struct {
	Model
	// ... existing fields
	CriteriaTags []Tag `gorm:"many2many:ArchetypeCriteriaTags;constraint:OnDelete:CASCADE"`
	CriteriaDeps string
}
```

One field on the REST resource, alongside the existing `criteria`. Existing clients are
unaffected; the field is omitted when empty.

```go
type Archetype struct {
	Resource     `yaml:",inline"`
	// ... existing fields
	Criteria     []TagRef `json:"criteria" yaml:"criteria" binding:"dive"`
	CriteriaDeps string   `json:"criteriaDeps,omitempty" yaml:"criteriaDeps,omitempty"`
}
```

There is no provider field, no list, and no child table. A list would be redundant with
`&&`, which the expression language already has; the provider is already recorded on
every `TechDependency` row and is read from there during matching.

`POST`/`PUT` of an archetype parses and validates the expression and returns `400` with
the reason and offset when parsing fails. Validation covers the boolean syntax, each
reference, and each constraint term, and rejects native-dialect syntax with a message
naming the equivalent — `[1.2,1.3)` → *"use `1.2..1.3`"*, `^1.2` → *"use `1.2+`"*. Users
will try the syntax of the ecosystem they know; the parser should teach rather than
misparse.

### Matching Semantics

Membership is resolved in two steps, and it is worth keeping them distinct because they
answer different questions.

**Step 1 — matching.** Does an archetype match an application? All criteria are ANDed:
the archetype's criteria tags must all be present on the application, **and** the
dependency expression must be satisfied. Disjunction within the dependency criteria is
expressed with `||` inside the expression. There is no way to OR a tag against a
dependency.

An archetype may have a dependency expression and no criteria tags, or tags and no
expression. An archetype with neither matches every application, as today.

**Step 2 — selection.** Of the archetypes that matched, which ones is the application
assigned to? Not necessarily all of them; see **Specificity** below.

**Direct vs. transitive.** A reference matches both direct and transitive dependencies.
The motivating cases — vulnerable and EOL libraries — usually arrive transitively, and
`TechDependency.Indirect` is available if a future `direct:` modifier on a reference
proves necessary. It is not in this proposal; adding a knob before anyone has asked for
it is how this feature grows a configuration surface it does not need.

**Specificity.** `MembershipResolver.Archetypes()`
(`internal/assessment/membership.go:51`) assigns an application only to the *narrowest*
matching archetypes. An archetype is discarded when its criteria tags are a strict
subset of another matching archetype's — more criteria tags wins. Archetypes whose tag
sets are incomparable both survive, so multiple memberships are already normal.

Measuring specificity in tags alone does not survive the addition of dependency
criteria. An archetype with an expression and no criteria tags has the empty tag set,
which is a strict subset of every other, so it loses to *any* tag-bearing archetype that
also matches:

| archetype | criteria tags | expression |
|---|---|---|
| A — "Spring Boot 2" | — | `org.springframework.boot:spring-boot@2` |
| B — "Java" | `Language=Java` | — |

An application tagged `Language=Java` running Boot 2 matches both, and A is discarded —
backwards, since A is plainly the more specific classification. A dependency-only
archetype would produce members only when nothing tagged also matched, which defeats
half of the feature.

Counting references in the expression is not a fix. Disjunction widens and conjunction
narrows, so `a || b || c` has three references and is broader than `a` — reference count
does not correlate with narrowness. Deciding genuine subsumption between two expressions
is decidable but expensive, and not worth it for an ordering heuristic.

**Dominance therefore becomes a product order over both dimensions.** Archetype X
dominates Y when X is at least as constrained as Y in tags *and* in expression, and
strictly more constrained in at least one. For the expression dimension:

- an empty expression is less constrained than any non-empty expression
- two non-empty expressions are comparable only if identical, and otherwise
  incomparable

| X | Y | result |
|---|---|---|
| tags `{Java,Web}`, expr `E` | tags `{Java}`, expr `E` | X dominates |
| tags `{Java}`, expr `E` | tags `{Java}`, no expr | X dominates |
| no tags, expr `E` | tags `{Java}`, no expr | incomparable, both assigned |
| tags `{Java}`, expr `E1` | tags `{Java}`, expr `E2` | incomparable, both assigned |

When no archetype uses an expression the expression dimension is always equal, so this
reduces exactly to the current tag-subset rule and no existing behavior changes. It also
declines to rank two different expressions rather than inventing an order for them,
which is consistent with how the algorithm already treats incomparable tag sets.

### Source of Dependency Data

Criteria are evaluated against `TechDependency` rows belonging to the application's
**most recent** analysis, which is the same scoping the analysis report endpoints already
use. Dependencies from superseded analyses are never considered.

**Applications with no analysis have no dependencies, so every reference evaluates to
false** and the application is not a member. This is the correct default — absence of
evidence is not evidence of a match — but it is a behavior change for an archetype that
previously matched on tags alone and later gains a dependency expression. Extending
discovery to report dependencies
([Dependency Data from Discovery](#dependency-data-from-discovery)) is what keeps this
from being a severe limitation in practice.
It is called out in the UI and in [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy).

### Dependency Data from Discovery

Dependency criteria are only useful if dependency data is widely available. Today it is
produced only by a full analysis, which is expensive and is not routinely run across a
whole portfolio. Discovery is — it is the cheap pass that tags applications during
onboarding — but it currently discards dependency information entirely.

In `konveyor/tackle2-addon-analyzer` today:

- `Analyzer.Run()` skips dependency resolution in discovery mode —
  `if !r.Data.Mode.Discovery { analyzer.GetDependencies(depOutput, false) }` in
  `cmd/analyzer.go`.
- `updateApplication()` returns immediately after tagging when `d.Mode.Discovery` is
  set, in `cmd/main.go`, so no analysis is created and dependencies would not be
  reported even if they had been collected.

A discovery run therefore produces tags and nothing else.

**Proposed change.** Discovery mode calls `GetDependencies()` and reports an analysis
containing the resolved dependencies with an empty insight set. The manifest is already
shaped for it — `builder.Manifest{Analysis, Insights, Deps}` — with only `Deps`
populated.

This makes dependency criteria usable on a portfolio that has been discovered but not
analyzed, which is the common state during onboarding and precisely when archetype
classification is most valuable. It also substantially softens the behavior noted in
[Source of Dependency Data](#source-of-dependency-data), where applications without an
analysis match nothing.

This change spans two repositories: the addon change above, and the hub change below.

#### A discovery analysis replaces the previous report

`AnalysisHandler.analysisIDs()` selects the latest analysis per application with
`MAX(ID) GROUP BY ApplicationID`, so a discovery run supersedes whatever preceded it,
including a full analysis. That is the intended behavior and requires no change.

Discovery is triggered by an edit to the application, and an edit can change what the
application points at. The insights of the previous analysis then describe a codebase
the application no longer refers to, and continuing to report them asserts findings
about the wrong thing. Replacing the report is the correct outcome — "latest analysis
has dependencies and no insights" accurately states *we know what it depends on, we have
not evaluated rules against this version*.

This also closes an existing gap: today an application can be edited and retain a stale
analysis indefinitely, because nothing invalidates it.

No discriminator is added to `Analysis`. Marking the producing mode on the report would
exist only to defeat the superseding behavior described above, and it would make "the
latest analysis" mean different things to different consumers.

**One ambiguity this leaves.** After a discovery run the UI shows zero insights, which
is indistinguishable from a full analysis of a genuinely clean application. One means
"not evaluated", the other "evaluated, nothing found", and reporting an application as
clean when it was never checked is a meaningful misstatement. Content cannot separate
them — a clean application also has dependencies and zero insights. Provenance is
already available outside the analysis record, in the application's task history, which
distinguishes an `analyzer` run from a `tech-discovery` run. That is where the UI should
resolve it rather than denormalizing addon mode onto the report.

**To confirm during implementation.** Whether `GetDependencies()` returns a complete
transitive closure in source-only mode. `Mode.ToOption()` selects `FullAnalysisMode`
only when `withDeps` is set, and `Scope.ToOptions()` notes that "if withDeps is false,
we are only ever doing source analysis." If the dependency list is incomplete without
full mode, discovery must either enable it — spending the time discovery exists to avoid
— or mark the reported dependencies as partial, which would make `Indirect` unreliable
and weaken transitive matching.

### When Associations Are Evaluated

Archetype membership is **not stored**. `MembershipResolver` is constructed per request
and computes membership in memory from the application tag join. Dependency criteria
follow the same model: membership stays derived, so there is no new consistency surface,
no reconciliation, and no stale associations to invalidate on analysis completion or
archetype edit. Results change the moment the underlying data changes.

Preserving that property is the main performance constraint. See
[Resolving Membership](#resolving-membership).

## Design Details

### Implementation

Expression parsing and matching are self-contained, depending on neither the API nor
the assessment packages, so they can be exercised in isolation.

**Version parsing is shared; ordering is not.** A single tokenizer handles every
ecosystem: strip a leading `v`, discard build metadata, split a PEP 440 epoch, split at
the first `-`, then tokenize on `.`/`-`/`_` and digit↔alpha boundaries. This correctly
parses Maven qualifiers, semver prereleases, PEP 440's separator-less `1.2.3a1`, Go
pseudo-versions, and `+incompatible`.

Ordering cannot be shared, and the reason is concrete rather than theoretical. Given
`1.0-1`:

- Maven reads it as release `1.0` with numeric qualifier `1` — sorts **after** `1.0`.
- PEP 440 reads it as an implicit post release — sorts **after** `1.0`.
- Semver reads it as a prerelease — sorts **before** `1.0`.

And Go pseudo-versions *require* the semver reading: `v1.0.1-0.20181226105442-5d4384ee4fb2`
must sort below `v1.0.1`. No single rule satisfies both. `NewComparator(provider)`
therefore returns a comparator that differs in exactly one behavior — whether a numeric
qualifier is a post release — and is otherwise shared.

This is the only place the provider is used, and it is read from the candidate
`TechDependency.Provider`, never from the expression. The user is not asked to supply
information the hub already has, and because the comparator is chosen per candidate
rather than per expression, one expression can span ecosystems and still order each
dependency by its own ecosystem's rules.

**Four constraint forms, one primitive.** `comparePrefix(version, prefix)` returns -1, 0,
or 1, where 0 means "within the line". Every term is then a pair of optional bounds:

```go
type term struct {
	lower *Version
	upper *Version
}
```

| term | lower | upper |
|---|---|---|
| `1.2` | `1.2` | `1.2` |
| `1.2+` | `1.2` | nil |
| `..1.5` | nil | `1.5` |
| `1.2..1.5` | `1.2` | `1.5` |

**Boolean evaluation reuses the selector pattern.** `internal/task/selector.go` scans
references out of the expression with a regular expression, resolves each to a boolean,
substitutes, and evaluates the result with `github.com/PaesslerAG/gval`. Dependency
expressions use the same approach and the same `gval.Language`, so the two should share
one implementation. Two refinements over the existing code: substitute generated
identifiers and pass a parameter map rather than splicing in the literal text
`true`/`false` (which removes the longest-match-first sort that currently guards against
one reference clobbering another), and resolve references lazily so short-circuiting is
real.

The reference scanner never needs to handle parentheses inside a version, because this
syntax has none — a consequence of rejecting Maven's bracket ranges.

**Why `..` and not `-` for ranges.** `-` is load-bearing inside versions in every
ecosystem. `1.2-1.5` is ambiguous, and the case that breaks is the Go pseudo-version
`v0.0.0-20210101120000-abcdef123456`, where both sides of the first `-` are numeric. A
"range only if both sides are numeric" heuristic does not save it; it silently matches
the wrong thing. `..` requires no disambiguation rule at all.

### Resolving Membership

Membership is resolved from an in-memory dataset. The distinct
`(applicationID, name, version, provider)` tuples of the latest analysis are fetched for
the applications in scope, each archetype expression is parsed once, and the expressions
are evaluated against each application's dependency set. `DISTINCT` is required because
the dependency unique index includes `SHA`, so one `(name, version)` can appear more
than once in an analysis.

Scope is the set of applications whose membership the request actually needs:

| request | scope |
|---|---|
| `GET /applications/{id}` and its sub-resources | one application |
| `GET /applications` | all, narrowed by the request's existing filter |
| `GET /archetypes/{id}/applications` | all |
| `GET /archetypes` with member counts | all |

`ApplicationHandler.List` already builds a `qf.Filter` over name, platform, and
repository; the dependency fetch is constrained to the same application set, so a
filtered list costs proportionally less.

**This does not scale past roughly 1000 applications**, and the endpoint under the most
pressure is the primary application list rather than an occasional report, because
`/applications` is not paginated. At ~400 dependencies per enterprise Java application,
a 10k portfolio means fetching on the order of millions of rows per request.

Three levers exist and none is proposed here, because which one is needed should be
settled by measurement rather than by argument:

- **Paginating `/applications`** would bound the fetch at `MaxPage` applications
  automatically. The machinery already exists in `internal/api/base.go`; the endpoint
  simply does not use it. This is the cleanest bound, but it changes the behavior of an
  existing endpoint and belongs in its own proposal.
- **A name pre-filter in SQL**, deriving a prefix from each reference's literal leading
  segments. Two non-obvious constraints apply: SQLite will not use an index for `LIKE`
  under the hub's default `case_sensitive_like` setting, so explicit range predicates
  are needed instead; and because `.`, `:`, and `/` are interchangeable separators, a
  prefix is only selective against a *normalized* name column, not against `Name` as
  stored.
- **Caching the matched set**, invalidated on archetype write and analysis completion.

For context on the baseline: `MembershipResolver` already loads the entire
application-tag join on every request that resolves membership, including single
application reads. The whole-portfolio scan is the status quo; this adds a second scan
roughly 20× larger. It is a multiplier on an existing cost, not a new class of behavior.

Benchmarks against current tag-only resolution time are owed before this proposal moves
off `provisional`.

### Test Plan

**Version comparator.** Table-driven tests seeded from each ecosystem's *published*
orderings — semver.org §11, PEP 440, and Maven's `ComparableVersion` qualifier table —
rather than from the implementation's behavior. These are the authoritative oracle: if
the comparator reproduces the specs, it is correct by construction. Supplemented with
real version strings drawn from dependency corpora (`go.sum`, `package-lock.json`),
which is where the pseudo-version and `+incompatible` cases come from.

**A total-order property test is required, not optional.** A prototype of this
comparator passed 253 example-based cases while containing a transitivity violation
(`1.0.0-alpha < 1.0.0 < 1.0-1`, yet `1.0.0-alpha` ranked above `1.0-1`). It surfaced only
from asserting antisymmetry and transitivity across the whole corpus, because it requires
three versions drawn from two different spec tables to manifest — exactly the combination
a hand-written table does not think to write.

**Constraint matching.** Per-form cases for prefix, `+`, `..`, ranges, and enumerations,
including the discriminating cases: `0.1` must not match `0.10.2`, `2` must not match
`20.1`, and a pseudo-version must not match its own release line.

**Expression parsing.** Valid expressions across all four ecosystems; malformed
expressions; and native-dialect syntax, asserting both rejection and the suggested
equivalent.

**Name matching.** Segment globbing across the separator forms, including
`@jsonjoy.com/**` (a scope containing a dot) and Go major-version suffixes.

**Membership integration.** API-level tests using `test/binding`: archetypes with
dependency criteria only, with tags only, and with both; applications with no analysis;
and applications whose latest analysis supersedes an earlier matching one.

**Specificity.** Each row of the product-order table, plus the property that matters
most for regression safety: with no archetype using an expression, selection produces
exactly the same result as the current tag-subset algorithm. The dependency-only versus
tag-only case — where both must be assigned — is the one the previous tag-only rule got
wrong, so it needs an assertion of its own.

**Discovery analyses.** That a discovery run reports dependencies and an empty insight
set; that a discovery run following a full analysis supersedes it entirely, so insight
reporting returns nothing and dependency matching uses the newly reported set; and that
a full analysis following a discovery run likewise supersedes it. The superseding
behavior is what keeps a stale analysis from outliving an edit to the application, so it
needs an explicit assertion rather than being left to the membership tests.

### Upgrade / Downgrade Strategy

A migration adds the `archetype.CriteriaDeps` column. No existing data is
modified and no existing archetype changes behavior: an archetype with an empty
expression resolves exactly as it does today.

The analyzer addon change requires no hub schema change and no sequencing: a discovery
run reports an analysis through the existing manifest upload, and the hub already
supersedes the previous report.

Downgrade drops the column. Archetypes revert to tag-only criteria, which widens their
membership — the dependency gate is removed. This should be called out in release notes
rather than mitigated, since preserving a filter the schema can no longer express is not
possible.

The one upgrade hazard is behavioral rather than structural: **adding a dependency
expression to an existing archetype will drop every application that has not been
analyzed**, because such applications have no dependencies and so satisfy no reference.
The UI should surface the resulting membership delta before the archetype is saved
(DAC005).

## Open Questions

1. **More memberships per application.** Because two different expressions are
   incomparable under the product order, applications will belong to more archetypes
   once expressions are in use than they do today. Multiple membership is already
   possible — incomparable tag sets produce it — but it becomes common rather than
   occasional. The consumers of membership need review before this is settled:
   assessment inheritance, inherited review and risk, and target profile selection.
   Specifically, what should happen when an application inherits from a tag-based
   archetype and a dependency-based archetype that disagree?

2. **Dashes are not segment separators.** `scikit-learn`, `django-rest-swagger`,
   `brace-expansion`, and `apache-airflow-providers-*` are single segments, so
   `google-cloud-*` cannot be wildcarded — and the dash is the dominant naming convention
   in exactly the Python and npm ecosystems. Adding `-` as a fourth separator would fix
   wildcarding but collides with PEP 503, which declares `-`, `_`, and `.` *equivalent
   within* a name. A per-provider name normalizer could resolve this; it is not in the
   current proposal.

3. **Validation endpoint.** Should expression validation be available independently of
   saving an archetype (`POST /archetypes/criteria/validate`), so the UI can validate as
   the user types without a round-trip through a write?

## Drawbacks

Dependency criteria make archetype membership depend on analysis having run, coupling
two subsystems that are currently independent. An archetype that looks correct can match
nothing simply because the portfolio has not been analyzed, and the failure is silent —
an empty member list, not an error.

It does not scale to large portfolios as proposed. Membership is resolved by fetching
dependencies and matching in memory, and because `/applications` is unpaginated, the
most frequently called endpoint in the product is also the one that resolves membership
for every application. Past roughly 1000 applications this needs one of the levers
described in [Resolving Membership](#resolving-membership). Adopting this feature on a
large estate therefore means accepting a latency regression on the application list, or
sequencing one of those changes first.

It also adds a second expression language to the hub. `internal/task/selector.go` already
defines one for addon and extension selection, with a different reference syntax
(`tag:category=name`). Sharing the boolean layer mitigates the implementation cost but
not the user-facing cost: two syntaxes that look similar and are not interchangeable.

Finally, the version comparator is a correctness liability. It is small, but it encodes
three published ordering specifications, and a subtle error produces wrong archetype
membership rather than a visible failure. The test strategy is weighted accordingly.

## Alternatives

**Per-ecosystem expression grammars.** A factory returning a concrete selector per
language, each accepting its ecosystem's native range syntax — Maven `[1.2,1.3)`, npm
`^1.2`, PEP 440 `>=1.2,<2.0`. This is architecturally clean and removes any tokenizer
ambiguity, since each scanner has full local context. It was rejected on cost/benefit:
each named dialect is a published spec one is then obliged to implement faithfully, and
a partial implementation fails silently rather than loudly. The benefit is also weakest
exactly where it was first requested — Maven version ranges are rare in real `pom.xml`
files, so most Java users would be learning the native syntax too, and Maven's *soft*
version semantics (bare `1.2` means "prefer 1.2, accept anything") would have to be
redefined as a match anyway. The decision is reversible: `NewComparator(provider)` is
already a per-provider factory, so a native scanner can be added later without affecting
callers or existing expressions.

**Maven bracket ranges in the shared syntax.** Rejected because `(1.2,1.3)` collides with
the grouping parentheses of the boolean layer, and because exclusive bounds are rarely
a distinction anyone needs to draw over discrete version numbers.

**Exact version semantics with `3.4.*` for wildcards.** The form suggested in the
originating issue. Prefix semantics supersedes it: it removes a construct, gives `*` a
single meaning across the whole language (one name segment), and fixes range endpoints,
which under exact semantics exclude the last line of the range.

**Materialized archetype membership.** Storing membership in an association table
maintained on analysis completion and archetype edit would make reads trivially cheap.
Rejected for now because membership is currently derived and keeping it derived avoids
an entire class of staleness bug. It remains the escape hatch if the cheaper levers in
[Resolving Membership](#resolving-membership) prove insufficient at scale.

**Matching against all analyses rather than the latest.** Rejected: an application that
once depended on a library but no longer does should not be classified by it.

## Implementation History

- 2026-10-30: Initial proposal.
