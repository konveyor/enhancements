---
title: dependency-archetype-criteria
authors:
  - "@jortel"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-10-30
last-updated: 2026-11-18
status: provisional
see-also:
  - "/enhancements/assessment-module/README.md"
---

# Dependency-Based Archetype Criteria

Archetypes classify applications by criteria tags alone, which works when the portfolio
has been curated but cannot express the most objective fact analysis already discovers:
what an application depends on.

This enhancement adds **dependency criteria** — a boolean expression over the
dependencies reported by analysis and tech-discovery — usable alone or alongside
criteria tags.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Membership-resolution benchmarks against the tag-only baseline
- [ ] User-facing documentation is created

## Summary

An archetype gains an optional dependency expression, matched against the dependencies
of the application's most recent analysis or tech-discovery:

```
[maven](
  org.springframework.boot:spring-boot@>=2|<3
  && (javax.servlet:javax.servlet-api || javax.persistence:javax.persistence-api)
  && !jakarta.servlet:jakarta.servlet-api
)
```

An application matches when its criteria tags match — as today — and the expression is
satisfied. Which of the matching archetypes it is assigned to is decided by specificity.

### What changes in existing behavior

Three things, all in systems that already work:

- **Archetype selection.** `MembershipResolver.Archetypes()` ranks by tag subset, so an
  archetype with an expression and no tags has the empty tag set and loses to every
  tag-bearing archetype that also matches. It would produce members only when nothing
  tagged matched, which defeats half the feature. Selection becomes a product order over
  both dimensions. See [Specificity](#specificity).
- **Tech-discovery starts reporting.** Today it tags and returns without creating a
  report, so dependency data exists only after a full analysis — too rare for this
  feature to be useful. Under this proposal tech-discovery reports dependencies with an
  empty insight set, which means it supersedes a prior analysis and that application's
  insights go with it. See
  [Dependency Data from Tech-Discovery](#dependency-data-from-tech-discovery).
- **Membership resolution gets roughly 20× more expensive.** Unmeasured. My estimate is
  that it becomes a problem near 1000 applications, and `/applications` — which is not
  paginated — reaches that first. See [Resolving Membership](#resolving-membership).

There is also an adoption hazard: adding an expression to an existing archetype drops
every member with no dependency data. A preview endpoint surfaces the delta before save.

### The expression language

Three parts. A **boolean layer** composes references with `&&`, `||`, `!`, and
parentheses. An **ecosystem block** — `[maven](…)` — declares which ecosystem the
references inside it name, once rather than per reference. A **dependency reference**
names one package:

```
<coordinates> [ '@' <constraint> ]
```

The two halves come from different places:

| half | form | varies by ecosystem? |
|---|---|---|
| `<coordinates>` | that ecosystem's own coordinate form — `group:artifact`, `@scope/name`, a module path | **yes** |
| `<constraint>` | [VERS](https://github.com/package-url/vers-spec) comparators — `>=`, `<=`, `>`, `<`, `=`, `!=`, separated by `\|` | **no** |

Coordinates are ecosystem-specific because that is what users have in hand — a line
copied from a `pom.xml`, a `package.json`, or an advisory. Constraints are uniform
because every ecosystem's *range* dialect is a separate published spec, and implementing
four of them buys little. A Java clause and an npm clause differ in their package names
and in nothing else.

Blocks compose, so one expression can span ecosystems:

```
[maven](org.springframework.boot:spring-boot@>=2|<3) && [npm](react@>=16|<17)
```

Version ordering, boolean evaluation, and name normalization are delegated to existing
implementations rather than written here. See [Implementation](#implementation).

## Motivation

Tag-based criteria depend on the portfolio being tagged accurately and consistently,
which is the slowest and least reliable part of onboarding a large estate. Tags are
applied by people, drift as applications change, and are frequently absent for
applications that have only just been discovered.

Analysis already produces a precise, machine-generated dependency inventory, refreshed
on every run. Archetypes are how Konveyor groups applications for assessment, review,
and target profiles, so the inability to say "every application on Spring Boot 2" forces
users to maintain by hand a classification the tool could derive.

Dependency data also stays current on its own. Tech-discovery is triggered by an edit to
an application, so once it reports dependencies an application's dependency set is
refreshed whenever the application changes. Tag criteria depend on someone noticing the
change and re-tagging.

The cases users describe most often:

- **Framework and version cohorts.** "All Spring Boot 2 applications", to assess the
  Boot 3 migration as a group.
- **Platform coupling.** "Anything importing WebLogic or WebSphere", which is rarely
  tagged but always visible in the dependency list.
- **Vulnerability and EOL sweeps.** "Everything on a Jackson older than 2.13", scoped to
  a wave rather than chased application by application.
- **Mixed criteria.** "Business-critical applications (tag) that are also on Struts 2
  (dependency)" — neither half expresses it alone.

### Goals

- Define a dependency expression syntax whose references are `<coordinates>@<constraint>`:
  coordinates written the way the ecosystem's own build files write them, and one
  constraint grammar — VERS — shared by every ecosystem.
- Delegate version ordering to a maintained implementation of each ecosystem's rules
  rather than reimplementing them.
- Extend the Archetype model and API to store dependency criteria alongside criteria
  tags, without breaking existing clients.
- Make dependency data available from tech-discovery, so criteria work on a portfolio
  that has been onboarded but not fully analyzed.
- Define matching and selection semantics when dependency criteria combine with criteria
  tags, including the interaction with existing specificity rules.
- Bound the cost of membership resolution: identify the scaling limit, the endpoint that
  reaches it first, and the levers that lift it.

### Non-Goals

- **A general-purpose query language.** This matches dependencies, not incidents, facts,
  or arbitrary application attributes.
- **Ecosystem-native version syntax.** Coordinates are native; the constraint is VERS
  everywhere. Maven `[1.2,1.3)`, npm `^1.2`, PEP 440 `>=1.2,<2.0`, and Ruby `~> 2.1` are
  rejected with a message naming the VERS equivalent. See
  [Alternatives](#alternatives) for why the native half stops at the `@`.
- **Full VERS constraint lists.** A constraint is one interval; see
  [Version constraint](#version-constraint).
- **Dependency resolution in the hub.** The hub matches what the addon reported. The
  addon keeps doing the resolving, including during tech-discovery.
- **SBOM ingestion.** Package URLs are accepted as a paste-in authoring form, but no
  SBOM is parsed and no dependency is rendered *as* a purl.
- **Tag inference from dependencies.** A separate concern.
- **Scaling beyond roughly 1000 applications.** Levers are identified in
  [Resolving Membership](#resolving-membership); none is implemented here.

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

*As an Architect, I want to match a whole package namespace without enumerating
artifacts, so that I can express "anything importing WebLogic".*

#### DAC005

*As an Architect, I want to see which applications an archetype's criteria currently
match before I save it, so that I can tell whether the expression says what I meant.*

#### DAC006

*As an Architect, I want to write package names in the form my build files already use,
so that I can paste a coordinate out of a `pom.xml`, a `package.json`, or a security
advisory instead of translating it.*

#### DAC007

*As an Architect working on a polyglot portfolio, I want one expression language and one
version grammar across ecosystems, so that only the package names differ between a Java
clause and an npm clause.*

### Expression Syntax

#### Boolean layer

Two levels. The outer composes ecosystem blocks; the inner composes references within a
block.

```
expr     = or
or       = and ( '||' and )*
and      = unary ( '&&' unary )*
unary    = '!' unary | block | '(' or ')'
block    = '[' ecosystem ']' '(' refExpr ')'

refExpr  = refOr
refOr    = refAnd ( '||' refAnd )*
refAnd   = refUnary ( '&&' refUnary )*
refUnary = '!' refUnary | ref | '(' refOr ')'
```

Blocks do not nest, and a reference appears only inside one, so the ecosystem of every
reference follows from its enclosing block with no scoping rule to resolve.

The boolean layer is shared with the addon/extension selector, which defines its own
reference syntax (`tag:category=name`) over the same operators. One implementation
serves both; see [Implementation](#implementation).

#### Ecosystem blocks

A block declares the ecosystem of every reference inside it:

```
[maven]( org.springframework:spring-core@>=6|<7 && !com.sun.xml.bind )
```

The ecosystem name is a Package URL type. Blocks compose with the boolean layer like any
other term, so an expression may contain several, and a block may be negated.

Declaring the ecosystem is what lets references be written natively:
`org.springframework:spring-core` and `@babel/core` are parsed by rules that already
know which ecosystem they belong to. It also fixes, at parse time, which version
comparator and which normalization rules apply.

It has a second effect that matters for cost: a block can only match dependencies whose
`Provider` corresponds to its type, so every block partitions the candidate set before
any name or version work happens. See [Resolving Membership](#resolving-membership).

An unrecognized ecosystem is a parse error, not an expression that matches nothing. The
mapping is a literal table, because none of the three names agree:

| block | provider | comparator ecosystem |
|---|---|---|
| `maven` | `java` | `Maven` |
| `npm` | `nodejs` | `npm` |
| `pypi` | `python` | `PyPI` |
| `golang` | `go` | `Go` |

These four are the ecosystems where both halves exist: a provider the analyzer emits and
an ordering the comparator implements. Dependencies reported under any other provider
can never be candidates — a limitation worth stating because it is invisible in an
expression. Adding an ecosystem is a row in this table plus a coordinate plugin.

There is no default ecosystem. An expression that does not declare one cannot be
evaluated correctly.

#### Dependency reference

```
ref         = coordinates [ '@' constraint ]
coordinates = segment (('.' | ':' | '/') segment)*
segment     = text | '*'
```

Coordinates are a sequence of **segments** matched against the reported dependency name.
`.`, `:`, and `/` are all segment separators, so the common Java form `group:artifact`
works whether the analyzer emitted a colon or a dot:

| provider | reported name | segments |
|---|---|---|
| java | `com.fasterxml.jackson.core:jackson-core` | `com fasterxml jackson core jackson-core` |
| java | `com.fasterxml.jackson.core.jackson-core` | *identical* |
| go | `github.com/jortel/go-utils/v2` | `github com jortel go-utils v2` |
| python | `zope.interface` | `zope interface` |
| npm | `@babel/core` | `@babel core` |

**Names match by segment prefix.** A reference matches when its segments are a leading
subsequence of the dependency's segments. `com.sun` matches `com.sun.xml.bind` and
`com.sun.jersey:jersey-core`; it does not match `com.sunflower`, because `sun` and
`sunflower` are different segments. A reference naming every segment matches only that
dependency.

This is how `weblogic` expresses "anything importing WebLogic" with no wildcard syntax,
and how a reference tolerates a dependency name carrying more coordinate components than
the reference does — see [Coordinate depth](#coordinate-depth).

`*` matches exactly one segment. It is only meaningful in a non-trailing position
(`com.*.internal`), since trailing segments are already open. There is no `**`.

**`@` is the only constraint introducer.** `:` is a separator and nothing else, which is
what lets the three separators be interchangeable. It also removes the arity ambiguity in
Maven's colon form — `G:A:V`, `G:A:P:V`, and `G:A:P:C:V` cannot be told apart without
counting, so `io.netty:netty-transport-native-epoll:jar` would silently parse `jar` as a
version. The parser rejects `group:artifact:1.2` with *"use `group:artifact@1.2`"*.

A **leading** `@` is part of the name — npm scopes retain it (`@babel/core` → segments
`@babel core`). Only a non-leading `@` introduces a constraint, so `@babel/core@7` is
scope `@babel`, name `core`, constraint `7`.

**Normalization is per-ecosystem, and the rules are adopted rather than invented.**
Separator interchange does not cover every naming convention: PEP 503 declares `-`, `_`,
and `.` equivalent within a Python distribution name, and npm and golang names are
case-folded. Each is specified per package type in
[purl-spec](https://github.com/package-url/purl-spec/blob/main/PURL-TYPES.rst), and the
hub adopts those rules. Because the block declares the ecosystem, the reference is
normalized once at parse time and the stored name under the same rules at match time.

The rules are adopted, not the parser. A reference containing `*` is not a purl and
cannot round-trip through one; see [Implementation](#implementation).

#### Version constraint

The block selects which comparator orders the versions; it does not change how the
constraint is written. A constraint is a list of comparisons separated by `|`, in
[VERS](https://github.com/package-url/vers-spec) comparator syntax:

```
constraint = comparison ( '|' comparison )*
comparison = [ '>=' | '<=' | '>' | '<' | '=' | '!=' ] version
```

| constraint | meaning |
|---|---|
| `@6.0.4` | exactly 6.0.4 — a bare version is `=` |
| `@>=2\|<3` | the 2 line |
| `@>=2.9\|<2.14` | the 2.9 through 2.13 lines |
| `@<5.4` | anything below 5.4 |
| `@>=2.0-beta9\|!=2.12.2\|<2.15.0` | a range with a hole |
| *omitted* | any version |

**A bare version is exact, not a prefix.** `@2` matches a version string that is
literally `2`, which almost nothing is; the 2 line is `@>=2|<3`. This is more verbose
than a prefix form on purpose — a prefix form has to invent a rule for what the line
beneath `2.0-beta9` is, and `@2.13` matching `2.13.7` is a convention that has to be
learned.

**A constraint is one interval:** at most one lower bound, at most one upper bound, and
any number of `!=`. VERS permits several intervals in one list — `>=1|<2|>=3|<4` is
`[1,2) ∪ [3,4)` — but the boolean layer already expresses that:

```
foo@>=1|<2 || foo@>=3|<4
```

Restricting to one interval keeps `|` reading one way (*and also*) rather than as
conjunction at two comparisons and disjunction at four, and it collapses the VERS
validity rules to a trivial check. What is accepted stays a strict subset of valid VERS,
so a constraint can be emitted as a canonical `vers:` string. The restriction is
relaxable later; it would not be tightenable.

**Prereleases sort below their release, in every ecosystem.** `>=2.13` excludes
`2.13.0-rc1`, `2.13-SNAPSHOT`, and `2.13rc1`; it includes `2.13-sp` and `2.13.post1`.
This is not a rule defined here — it falls out of each ecosystem's published ordering.
It has one severe consequence, in Go; see [Go major versions](#go-major-versions).

#### Coordinate depth

The analyzer reports a dependency name at whatever coordinate depth it has, and segment
prefix matching means a reference does not have to match that depth: if a Java dependency
is reported as `io.netty:netty-transport-native-epoll:jar:linux-x86_64`, both
`io.netty:netty-transport-native-epoll` and
`io.netty:netty-transport-native-epoll:jar` match it.

This matters beyond convenience. If the analyzer later reports more fully-qualified
names, old and new reports coexist in the same portfolio for as long as the old ones
survive, and references written against either depth keep working. No migration of
stored expressions is required.

#### Go major versions

In Go the major version appears in the module path (`github.com/golang-jwt/jwt/v4`), so
segment prefix matching alone cannot separate majors — `github.com/golang-jwt/jwt`
matches `/v4` and `/v5` both. Constrain the version instead, which is equivalent because
a module's version major always agrees with its path:

```
[golang]( github.com/golang-jwt/jwt@>=v4|<v5 )
```

A leading `v` is accepted on either side and is not significant: `@>=v4` and `@>=4` are
the same constraint, and both compare correctly against a stored `v4.5.0`.

Constraining the version also covers the two cases the path cannot express: v0 and v1
share the unversioned path, and `+incompatible` modules carry a major in the version
with no path suffix.

**The v0 trap.** An untagged Go module has a pseudo-version —
`v0.0.0-20210101120000-abcdef123456` — which is a *prerelease* of `0.0.0` under semver
and therefore sorts **below** `v0`. So `@>=v0|<v1` matches almost no v0 module. Write
`@<v1`, or `@>=v0.0.0-0|<v1` if a lower bound is wanted. This is the prerelease rule
behaving as specified; it is severe here because Go's dominant version form is a
prerelease.

#### Package URLs as an input form

A reference may be pasted as a
[Package URL](https://github.com/package-url/purl-spec), which carries its own ecosystem
and needs no enclosing block:

```
pkg:maven/com.fasterxml.jackson.core/jackson-databind@2.9.10
  ≡  [maven]( com.fasterxml.jackson.core:jackson-databind@2.9.10 )
```

This exists for one workflow: the motivating cases — a vulnerable Jackson, an EOL Struts
— are read off an advisory, and advisories speak purl. A purl's `@version` is a single
concrete version, which is what a bare version means here, so the forms agree without a
special rule.

It is an input form, not the canonical one. A purl identifies one package and has no
wildcard layer, so `com.sun` cannot be written as one, and the two forms do not compose
within a single reference. Qualifiers and subpaths are rejected rather than ignored:
they narrow what a purl identifies, so dropping them silently would make the reference
match more than the identifier the user pasted.

Nothing is ever rendered *as* a purl, because that requires deciding where the group ends
and the artifact begins in `com.fasterxml.jackson.core.jackson-core`.

#### Evaluating a reference

A reference and a candidate dependency reduce to the same shape before anything is
compared. The block supplies the ecosystem for both halves; coordinates and constraint
are then matched independently.

```
REFERENCE   [maven]( com.redhat.fuse:fuse-core@>=7.9|<7.12 )

CANDIDATE   provider  java
            name      com.redhat.fuse.fuse-core
            version   7.11.1

ecosystem   maven  (from the block)
            java   (from the candidate)      -> agree, via the mapping table

coordinates com.redhat.fuse:fuse-core
            split on . : /                   -> [com, redhat, fuse, fuse-core]
            normalize per purl-spec, maven   -> unchanged
name        com.redhat.fuse.fuse-core
            split on . : /                   -> [com, redhat, fuse, fuse-core]
            normalize per purl-spec, maven   -> unchanged
                                             -> segment match: true

constraint  >=7.9|<7.12
            parse comparators                -> lower 7.9 incl, upper 7.12 excl
            equivalently                     == vers:maven/>=7.9|<7.12
version     7.11.1
            validate as a Maven version      -> ok
            compare under ecosystem "Maven"  -> in range: true
```

Note that the analyzer reported the name dot-joined, with nothing marking where the
group ends and the artifact begins. Separator interchange makes that irrelevant to
matching — and makes rendering the same name as a purl impossible.

**Segment matching.** The reference matches when it has no more segments than the
candidate and every one of its segments equals the candidate's in position, with `*`
equal to any single segment. Trailing candidate segments are unconstrained:

| reference | candidate | match |
|---|---|---|
| `com.redhat:fuse-core` | `com.redhat.fuse-core` | yes — equal, separators interchangeable |
| `com.redhat` | `com.redhat.fuse-core` | yes — trailing segments open |
| `com.*.fuse-core` | `com.redhat.fuse-core` | yes — `*` is exactly one segment |
| `com.*` | `com.redhat.fuse-core` | yes — `*` matches `redhat`, rest is trailing |
| `com.redhat:fuse` | `com.redhat.fuse-core` | no — different segments; `-` does not separate |
| `com.redhat:fuse-core:jar` | `com.redhat.fuse-core` | no — reference is deeper than the candidate |

Both sides reduce to `(type, segments, version)`, which is purl-shaped and deliberately
not a purl: the reference half may be a prefix or carry `*`, and the candidate half has
no recoverable namespace boundary. The constraint half *is* VERS and serializes as such.

#### Examples

```
# Spring Boot 2 → 3 migration cohort
[maven](
  org.springframework.boot:spring-boot@>=2|<3
  && org.springframework.boot:spring-boot-starter-web@>=2.0|<2.8
  && com.fasterxml.jackson.core:jackson-databind@>=2.9|<2.14
  && (javax.servlet:javax.servlet-api || javax.persistence:javax.persistence-api)
  && !jakarta.servlet:jakarta.servlet-api
)
```

```
# Legacy application server coupling — segment prefix, no wildcards needed
[maven]( weblogic || com.ibm.websphere || org.jboss.as || com.sun )
```

```
# Log4Shell exposure: affected range with the fixed 2.12.2 carved out
[maven]( org.apache.logging.log4j:log4j-core@>=2.0-beta9|!=2.12.2|<2.15.0 )
```

```
# Stranded on an old Go major; note <v1 rather than >=v0|<v1
[golang](
  github.com/golang-jwt/jwt@>=v4|<v5
  || github.com/go-chi/chi@<v5
  || github.com/jortel/go-utils@<v1
)
```

```
# Vulnerable pins
[pypi](
  pyyaml@<5.4 || jinja2@<2.11.3 || urllib3@<1.26.5 || cryptography@<3.3.2
)
```

```
# Polyglot: a Spring Boot 2 service with a legacy front end, where both halves
# have to move together. Each block carries its own boolean structure.
[maven](
  org.springframework.boot:spring-boot@>=2|<3
  && (javax.servlet:javax.servlet-api || javax.persistence:javax.persistence-api)
  && !jakarta.servlet
)
&& [npm](
  react@>=16|<17
  && (webpack@>=4|<5 || grunt || gulp)
)
```

Each block is evaluated against its own ecosystem's dependencies and combined by the
outer `&&`. A reference is confined to its block: the npm `react` can never match a
Maven artifact named `react`, and `!jakarta.servlet` excludes Java packages only.

Blocks are terms, so a block may be negated:

```
[maven]( org.springframework.boot:spring-boot@>=2|<3 ) && ![npm]( react )
```

This reads *Spring Boot 2, with a front end that is not React* — not *with no front
end*. An application whose latest report never covered npm is excluded rather than
matched, per the per-block guard in [Matching Semantics](#matching-semantics).

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

No provider field, no list, no child table. A list would be redundant with `&&`, and the
ecosystem is declared by the block.

`POST`/`PUT` parses and validates the expression, returning `400` with the reason and
offset on failure. Validation covers boolean syntax, block ecosystem, each reference, and
each constraint, and rejects foreign syntax with a message naming the equivalent:

| rejected | message |
| --- | --- |
| `[1.2,1.3)` | use `@>=1.2\|<1.3` |
| `^1.2` | use `@>=1.2\|<2` |
| `~1.2` | use `@>=1.2\|<1.3` |
| `1.2,2.0` | use `@>=1.2\|<2.0` — `,` is not a separator |
| `group:artifact:1.2` | use `group:artifact@1.2` |
| `3.4.*` | use `@>=3.4\|<3.5` |
| `com.sun:**` | trailing segments are already open; use `com.sun` |
| `@>=1\|<2\|>=3\|<4` | one interval per constraint; use `foo@>=1\|<2 \|\| foo@>=3\|<4` |
| `vers:maven/>=1.2\|<2.0` | use `@>=1.2\|<2.0` |
| reference outside a block | wrap in an ecosystem block, e.g. `[maven](…)` |
| `[java](…)`, `[Maven](…)` | unknown ecosystem; use `[maven]` |

Users will try the syntax of the ecosystem they know. The comma case matters most: PEP
440 reads `1.2,2.0` as conjunction and this syntax has no comma at all, so accepting it
would silently mean the opposite of what a Python user intends. `[java]` is worth a
specific message because `java` is the provider value stored on the dependency and will
be the first guess of anyone who has looked at the data.

#### Membership preview

`POST /archetypes/membership` accepts `{criteria, criteriaDeps}` and returns the matching
application refs and their count without persisting an archetype. For an existing
archetype the UI diffs the result against current membership to show the delta before
save (DAC005), which is what surfaces the hazard in
[Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy).

The count matters as much as the list: because names match by segment prefix, the
characteristic authoring error is an expression that matches *more* than intended — a
reference truncated by one segment widens silently, where under an exact-match rule it
would have failed visibly.

Preview resolves membership across the whole portfolio and costs the same as
`GET /archetypes/{id}/applications`, so it is an explicit action rather than a
keystroke-driven one. Syntax validation is separable and cheap — a malformed expression
returns `400` before any dependency set is fetched — so the same endpoint serves both and
no separate `validate` route is needed.

### Matching Semantics

Membership resolves in two steps: *matching* (does this archetype match this
application?) and *selection* (which of the matching archetypes is it assigned to?).
Both change.

#### Specificity

This is the part of the existing algorithm that breaks.

`MembershipResolver.Archetypes()` (`internal/assessment/membership.go`) assigns an
application only to the narrowest matching archetypes, discarding an archetype whose
criteria tags are a strict subset of another matching archetype's. Measuring specificity
in tags alone does not survive dependency criteria:

| archetype | criteria tags | expression |
|---|---|---|
| A — "Spring Boot 2" | — | `[maven](org.springframework.boot:spring-boot@>=2\|<3)` |
| B — "Java" | `Language=Java` | — |

An application tagged `Language=Java` running Boot 2 matches both. A's tag set is empty,
so it is a strict subset of B's and gets discarded — even though its expression is a real
constraint the tag-subset rule cannot see. A dependency-only archetype would produce
members only when nothing tagged also matched.

Counting references in the expression is not a fix: disjunction widens and conjunction
narrows, so `a || b || c` has three references and is broader than `a`. Deciding genuine
subsumption between two expressions is decidable but expensive, and not worth it for an
ordering heuristic.

**Dominance becomes a product order over both dimensions.** X dominates Y when X is at
least as constrained as Y in tags *and* in expression, and strictly more constrained in
at least one. For the expression dimension:

- an empty expression is less constrained than any non-empty expression
- two non-empty expressions are comparable only if identical, otherwise incomparable

| X | Y | result |
|---|---|---|
| tags `{Java,Web}`, expr `E` | tags `{Java}`, expr `E` | X dominates |
| tags `{Java}`, expr `E` | tags `{Java}`, no expr | X dominates |
| no tags, expr `E` | tags `{Java}`, no expr | incomparable, both assigned |
| tags `{Java}`, expr `E1` | tags `{Java}`, expr `E2` | incomparable, both assigned |

For A/B the product order declines to rank them, so both are assigned. Ranking a tag
against an expression would need a common scale of specificity that does not exist, and
declining to invent one matches how the algorithm already treats incomparable tag sets.

When no archetype uses an expression the expression dimension is always equal, so this
reduces exactly to the current tag-subset rule and nothing existing changes.

#### Matching

All criteria are ANDed: the archetype's criteria tags must all be present **and** the
dependency expression must be satisfied. Disjunction within dependency criteria is
expressed with `||` inside the expression; there is no way to OR a tag against a
dependency. An archetype may have an expression and no tags, or tags and no expression;
one with neither matches everything, as today.

**No dependency data.** When an application has none, a non-empty expression is
unsatisfied regardless of its structure. The guard applies to the expression as a whole,
not to its references: a reference resolving to false would make
`!jakarta.servlet:jakarta.servlet-api` resolve to *true*, so an expression leading with a
negation would match every application with no dependency data.

**The same guard applies per block.** A block whose ecosystem contributed no dependencies
to the latest report is unsatisfied regardless of structure. Without this, `![npm](react)`
would be satisfied by every application whose report never ran an npm provider —
asserting the absence of React on evidence never collected. The hub cannot distinguish
"has no npm dependencies" from "npm was never inspected".

So a negated block only means something for an ecosystem the latest report actually
covered: `![npm](react)` selects applications known to have npm dependencies, none of
them React.

**Unparseable versions.** A candidate whose version string does not parse under its
ecosystem's rules satisfies no constraint, including a negated one. The comparator does
not reject such strings; the hub must. See [Implementation](#implementation).

**Direct vs. transitive.** A reference matches both. The motivating cases — vulnerable
and EOL libraries — usually arrive transitively, and the analyzer already reports
indirectness, so a `direct:` modifier is available later if it proves necessary.

Membership remains derived rather than stored: `MembershipResolver` is constructed per
request and computes in memory, so there is no new consistency surface and nothing to
invalidate when a report lands. Preserving that is the main performance constraint; see
[Resolving Membership](#resolving-membership).

### Source of Dependency Data

Dependencies are reported by two kinds of run: a full **analysis**, and — under this
proposal — **tech-discovery**. Criteria are evaluated against the dependencies of the
**most recent** report, whichever kind produced it, which is the same scoping the
analysis report endpoints already use. Superseded reports are never considered.

A reported dependency carries a provider, a name, a version, and whether it was reached
indirectly, all passthrough from analyzer-lsp. This proposal interprets provider, name,
and version, which is the extent of the coupling it introduces.

An application that has had neither an analysis nor a tech-discovery has no dependency
data, so no expression is satisfied and it does not match — including for negated
expressions. The consequence is a behavior change for an archetype that previously
matched on tags alone and later gains an expression: every member with no dependency data
drops. It is surfaced before save (DAC005) and in
[Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy).

### Dependency Data from Tech-Discovery

Dependency criteria are only useful if dependency data is widely available, and today it
is produced only by a full analysis, which is not routinely run across a portfolio.
Tech-discovery is, but it currently discards dependency information: `Analyzer.Run()`
skips dependency resolution during tech-discovery, and `updateApplication()` returns
after tagging without creating a report.

**Proposed change.** Tech-discovery resolves dependencies and reports them with an empty
insight set. The upload manifest already carries report, insights, and dependencies as
separate parts, so this populates only the last. The change is confined to
`konveyor/tackle2-addon-analyzer` and relies on existing hub behavior.

This makes dependency criteria usable on a portfolio that has had tech-discovery but no
analysis — the common state during onboarding, and when archetype classification is most
valuable.

Note that tech-discovery is not a profile setting; it arrives through the task data.
Varying dependency collection *by profile* would require extending the analysis profile
mode first, and is not proposed.

#### A tech-discovery report replaces the previous one

The hub already treats the newest report per application as current, so a tech-discovery
supersedes whatever preceded it, including a full analysis. No change required.

This is the right outcome rather than an accident. Tech-discovery is triggered by an edit,
and an edit can change what the application points at, so the insights of the previous
analysis may describe a codebase the application no longer refers to. "Latest report has
dependencies and no insights" accurately states *we know what it depends on, we have not
evaluated rules against this version*. It also closes an existing gap: today an
application can be edited and retain a stale analysis indefinitely.

No **addon-mode** discriminator is added to `Analysis`. Marking the producing run on the
report would exist only to defeat the superseding behavior, and would make "the latest
report" mean different things to different consumers.

**One ambiguity this leaves.** After a tech-discovery the UI shows zero insights, which
is indistinguishable from a full analysis of a genuinely clean application — one means
"not evaluated", the other "evaluated, nothing found", and reporting an application as
clean when it was never checked is a meaningful misstatement. Content cannot separate
them. Provenance is available outside the report, in the task history, which distinguishes
an `analyzer` run from a `tech-discovery` run; that is where the UI should resolve it.

**To confirm during implementation.** Whether dependency resolution returns a complete
transitive closure in source-only mode. Full analysis mode is selected only when
dependencies are requested, and without it the addon is doing source analysis alone. If
the dependency list is incomplete in that mode, tech-discovery must either enable full
mode — spending the time tech-discovery exists to avoid — or mark the reported
dependencies partial, which would weaken transitive matching.

## Design Details

### Implementation

Expression parsing and matching depend on neither the API nor the assessment packages, so
they can be exercised in isolation. The principle is to own nothing that is already a
published specification with a maintained implementation:

| concern | owner |
|---|---|
| boolean evaluation, block scoping | `github.com/PaesslerAG/gval` (already a hub dependency) |
| version ordering, per ecosystem | `github.com/google/osv-scalibr/semantic` |
| name normalization rules | purl-spec, adopted as rules |
| coordinate parsing, name matching | in-house, per-ecosystem plugin |
| version syntax validation | in-house |
| the `\|`-separated constraint and its interval | in-house |

#### Boolean layer and blocks

`internal/task/selector.go` already evaluates boolean expressions over `gval`'s
propositional-logic language, resolving each reference to a boolean. It is refactored so
the boolean layer is one implementation with a pluggable reference resolver, and
dependency criteria supply a second resolver alongside the existing one.

`gval` extends to `[ecosystem](…)` through its prefix-extension hook: `[` carries no
meaning in the propositional-logic language, so claiming it displaces nothing. The
extension reads the ecosystem, parses the parenthesized sub-expression, and compiles each
reference against that ecosystem.

#### Coordinate plugin

Per-ecosystem behavior is reached through a plugin selected at parse time from the block
header. It owns the ecosystem's coordinate structure and normalization rule; segment
splitting and prefix/`*` matching are shared, so there is one matcher rather than one per
ecosystem. Its interface is an implementation detail and is not specified here.

A pattern is not a purl: `packageurl-go` and `osv-scalibr/purl` have no wildcard layer,
and normalizing a struct whose name field contains `*` is undefined. Normalization is
therefore a plugin method applying purl-spec's per-type rules to segments, not a
round-trip through a purl parser.

#### Version comparison

Ordering is delegated to `osv-scalibr/semantic`, which implements each ecosystem's
published rules. Its ecosystem keys are OSV names and are case-sensitive, which is why
the mapping in [Ecosystem blocks](#ecosystem-blocks) is a literal table.

Behavior was verified against v0.5.3 rather than assumed — Maven qualifier ordering
including the `ga`/`final`/`release` aliases, PEP 440 pre/post/epoch handling, and Go
pseudo-versions and `+incompatible`. Three results drive decisions elsewhere:

- `2.13.5` is **greater** than `2.13`, which is why a line is written `@>=2.13|<2.14` and
  not `@<=2.13`.
- A leading `v` is stripped, so `@>=v4` and `@>=4` are one constraint.
- Go is ordered as plain semver with no Go-specific handling, which gives correct
  pseudo-version and `+incompatible` results and also produces the v0 trap.

**The library does not validate, and the hub must.** It returns no error for a malformed
version, despite documenting that it does. Verified: `banana`, `${project.version}`,
`LATEST`, and the empty string all parse in Maven, npm, Go, PyPI, Debian, and Alpine, and
sort *below* real versions in most ecosystems but *above* them in Debian. An unresolved
Maven property in analyzer output would therefore satisfy any `<X` constraint silently.

The hub validates candidate version strings per ecosystem before comparison and treats
failure as "this dependency does not satisfy this reference". The wrapper is thin, but it
cannot be skipped and no library supplies it.

A parsed constraint is an optional lower bound, an optional upper bound, and a set of
exclusions. A version satisfies it when it falls within whichever bounds are present and
equals none of the exclusions, with every comparison going through the comparator.

### Resolving Membership

Membership is resolved from an in-memory dataset: the distinct
`(application, provider, name, version)` tuples of the latest report are fetched for the
applications in scope, each expression is parsed once, and the expressions are evaluated
per application. Distinctness has to be requested explicitly — dependencies are keyed by
artifact hash as well, so one name and version can appear more than once in a report.

Scope is the set of applications the request actually needs:

| request | scope |
|---|---|
| `GET /applications/{id}` and its sub-resources | one application |
| `GET /applications` | all, narrowed by the request's existing filter |
| `GET /archetypes/{id}/applications` | all |
| `GET /archetypes` with member counts | all |
| `POST /archetypes/membership` (preview) | all |

**Ecosystem blocks partition the candidate set.** A `[maven]` block can only match
dependencies whose provider is `java`, which is known at parse time. Provider is the
leading column of the dependency table's composite index, so the filter is
index-supported. On a polyglot portfolio this is a large constant-factor reduction — a
Java-only expression never parses an npm version — but it does not change the asymptotics.

**This does not scale past roughly 1000 applications.** That number is an estimate, not a
measurement: at a few hundred dependencies per enterprise Java application, the
latest-report dependency set is on the order of 400k rows there. The endpoint under the
most pressure is the primary application list rather than an occasional report, because
`/applications` is not paginated.

One property is worth keeping in mind for whichever lever gets picked: the costly work is
version parsing and comparison, which scales with *distinct* versions in use rather than
with applications × dependencies.

Three levers exist; none is proposed here, because which one is needed should be settled
by measurement:

- **Paginating `/applications`** would bound the fetch at one page automatically. The
  machinery already exists; this endpoint does not use it. The cleanest bound, but it
  changes an existing endpoint's behavior and belongs in its own proposal.
- **A name pre-filter in SQL**, deriving a prefix from each reference's literal leading
  segments. Two constraints apply: SQLite will not use an index for `LIKE` under the hub's
  default `case_sensitive_like` setting, so explicit range predicates are needed; and
  because separators are interchangeable, a prefix is only selective against a
  *normalized* name column.
- **Caching the matched set**, invalidated on archetype write and on a new report.

For baseline: `MembershipResolver` already loads the entire application-tag join on every
request that resolves membership, including single application reads. The whole-portfolio
scan is the status quo; this adds a second scan roughly 20× larger.

Benchmarks against current tag-only resolution are owed before this moves off
`provisional`.

### Test Plan

The budget follows the ownership split above. The comparator is a maintained library
tested against its own specs, so it is verified rather than tested; name matching and
validation have no library and no oracle, so they carry the weight.

Three cases are called out because a regression in them is silent rather than loud: an
unparseable version must not satisfy a *negated* reference; `@>=v0|<v1` must fail to match
a Go pseudo-version while `@<v1` matches it; and with no archetype using an expression,
selection must produce exactly the current result.

| area | cases |
|---|---|
| comparator conformance | contract test against `osv-scalibr/semantic` for the orderings relied on here, plus `2.13.5 > 2.13`; fails loudly if an upgrade changes ordering |
| version validation | `banana`, `${project.version}`, `LATEST`, `""`, a Maven property placeholder — each rejected by the hub's validator, and a dependency carrying one satisfies no reference, negated included |
| name matching | segment prefix across separator forms; `com.sun` matches `com.sun.xml.bind`, not `com.sunflower`; reference matching greater reported depth; `@jsonjoy.com/util`; `@babel/core@7`; `com.foo:3d-view`; Go path suffixes; non-trailing `*` |
| normalization | PEP 503 equivalence (`apache_airflow`, `apache.airflow` → `apache-airflow`); npm and golang case folding; no leakage across ecosystems — `-` stays load-bearing in Maven and Go, Maven stays case-sensitive |
| expression parsing | all four ecosystems; nested and negated blocks; every row of the rejection table, asserting the suggested equivalent; `1.2,2.0` and `group:artifact:1.2` rejected under any reading; a reference outside a block is a parse error |
| constraint semantics | each comparator and `\|`; `@2` does not match `2.7.18` but `@>=2\|<3` does; a second interval rejected with the `\|\|` equivalent; `>=2.13` excludes `2.13.0-rc1` and `2.13-SNAPSHOT`, includes `2.13-sp` |
| Go idioms | `@>=v4\|<v5` selects `/v4` not `/v5`; the v0 pseudo-version case above; `v2.0.0+incompatible` behaves as `v2.0.0` |
| purl input | a pasted purl resolves to the same candidate set as the block form; `pkg:npm/foo` does not match a PyPI `foo`; qualifiers and subpaths rejected, not ignored |
| membership (`test/binding`) | dependency criteria only, tags only, both; no dependency data; a latest report superseding an earlier matching one |
| no dependency data | neither analysis nor tech-discovery must not match `!foo` or a disjunction containing it — the case where evaluating references individually gives the wrong answer |
| preview | returns the set the archetype would produce if saved; malformed expression returns `400` with an offset |
| specificity | each row of the product-order table; the regression property above; dependency-only vs. tag-only, which the tag-only rule got wrong |
| tech-discovery | reports dependencies with an empty insight set; supersedes a prior full analysis entirely, and is superseded by a later one |

### Upgrade / Downgrade Strategy

A migration adds the `archetype.CriteriaDeps` column. No existing data is modified and no
existing archetype changes behavior: an empty expression resolves as it does today.

The analyzer addon change requires no hub schema change and no sequencing — a
tech-discovery reports through the existing manifest upload, and the hub already
supersedes the previous report.

Downgrade drops the column. Archetypes revert to tag-only criteria, which widens their
membership because the dependency gate is removed. This should be called out in release
notes rather than mitigated; preserving a filter the schema can no longer express is not
possible.

The one upgrade hazard is behavioral: **adding a dependency expression to an existing
archetype drops every application with no dependency data.** The UI surfaces the
membership delta before save via `POST /archetypes/membership` (DAC005).

## Open Questions

1. **How do the consumers of membership handle conflicting archetypes?** *This blocks
   `implementable`.* Because two different expressions are incomparable under the product
   order, applications will belong to more archetypes once expressions are in use.
   Multiple membership is not new — incomparable tag sets already produce it — so this is
   a pre-existing gap in assessment inheritance, inherited review and risk, and target
   profile selection. It blocks because shipping dependency criteria turns an edge case
   into the normal path. Resolving it is work in the assessment module, not here.

2. **Dash segmentation in npm.** `-` is not a segment separator, so there is no
   wildcarding within an npm package name. Python is settled by PEP 503, which declares
   `-`, `_`, and `.` equivalent within a distribution name. npm is not: `.` and `-` are
   distinct characters in distinct package names — `lodash.merge` and `lodash-merge` could
   both exist — so segmenting on `-` conflates them with each other and with the separator
   forms. The choice is between leaving npm names single-segment, costing wildcarding
   within a name, and accepting the conflation. Java and Go are unaffected.

## Drawbacks

Dependency criteria couple archetype membership to analysis, two subsystems that are
currently independent. An archetype that looks correct can match nothing because the
portfolio has not been analyzed, and the failure is silent — an empty member list, not an
error.

**Segment prefix matching makes the characteristic failure over-matching.** A reference
truncated by one segment silently widens, and an archetype that captures too many
applications looks plausible where an empty one does not. The preview count exposes it
after the fact; completing references against the coordinates the portfolio actually
reports would prevent most of it, since the hub already holds every name every report has
produced.

The pressure on application listing is the drawback I am least able to bound. Membership
is calculated when applications are fetched, so the implementation has to build in-memory
dependency maps for whole pages of applications, and the effect on listing a large estate
is hard to predict in advance. Adopting this at scale means accepting a latency regression
there, or sequencing one of the levers in
[Resolving Membership](#resolving-membership) first.

It adds a second *reference* syntax alongside the `tag:category=name` form used for addon
and extension selection. Sharing the boolean layer removes the implementation cost but not
the user-facing one: two syntaxes that look similar and are not interchangeable.

Delegating version ordering removes the largest correctness liability but creates a
dependency on `osv-scalibr`'s per-ecosystem implementations, including for ecosystems
Konveyor cares about more than its primary users do. The conformance table makes a
behavior change on upgrade visible rather than silent. The library's silent handling of
malformed versions is a liability the hub inherits by using it — the validation wrapper is
small, but what it guards against is wrong membership with no error anywhere.

## Alternatives

**Structured criteria instead of an expression language** — a list of
`{ecosystem, coordinates, constraint}` objects, ANDed. The conventional API shape: no
grammar, no parser, no error offsets. It fails because the motivating criteria are not
conjunctions — the Spring Boot cohort needs a disjunction nested inside a conjunction with
a negation alongside it, so representing it structurally means encoding an expression tree
as `allOf`/`anyOf`/`not` nodes, which is the same language with more punctuation.
Flattening to a conjunctive list drops exactly the cases the feature exists for. The
simple case stays simple either way: an expression of `&&`-joined references *is* a
conjunctive list, and a UI may present it as one.

**Native version ranges** — each ecosystem's own dialect inside its block, so the whole
reference reads as the build file writes it. Each dialect is a published spec one is then
obliged to implement faithfully, and for Maven ranges no Go library of adequate maturity
exists. Maven's *soft* semantics also make its most common form unusable as a predicate:
bare `1.2` means "prefer 1.2, accept anything". Two dialects collide with the boolean
layer — Maven's `(1.2,1.3)` against grouping parens, and npm's own `||`. The benefit is
weakest for Java, where ranges are rare in real `pom.xml` files.

**PURL and VERS as the authoring syntax and internal model** — references as purls,
constraints as `vers:` URIs, stored and matched in that form. Attractive because both are
widely emitted (SPDX, CycloneDX, OSV, advisories) and carry the ecosystem explicitly.
Adopted in part. Rejected as the model because a purl is an identifier, not a pattern:
there is no wildcard layer, so `com.sun` cannot be written as one, and a glob over purl is
no longer a purl — losing the interop that motivated it. The interop also runs one way:
matching purl against purl means rendering stored `(provider, name, version)` as purls,
which needs a group/artifact boundary the stored form does not carry. And percent-encoded
`pkg:maven/...?vers=...%7C...` is a machine format, not hand-authored.

Adopted: VERS comparator syntax, purl types as block names, purl-spec normalization rules,
purl as a paste-in form. Not adopted: multi-interval VERS containment, purl as the
internal representation. Worth reopening if SBOM or advisory ingestion lands, since the
hub would then hold purl-typed data and the group/artifact objection disappears. OSV and
CPE were also considered — OSV is an advisory schema rather than an expression language,
CPE identifies vendor and product rather than build coordinates.

**Ecosystem-agnostic references** — no blocks, one neutral coordinate grammar, ordering
selected per candidate from the recorded provider. The ecosystem is needed before any
candidate is in hand: `@babel/core` and `group:artifact` are different shapes, so an
agnostic syntax must also be a neutral one and users translate rather than paste.
Declaring it also buys the candidate partition, which reading the provider per candidate
cannot.

**Prefix version semantics** — `@2` meaning the whole 2 line. Terser, but it needs an
invented rule for the line beneath a qualified version such as `2.0-beta9`, and `@2.13`
matching `2.13.7` is a convention that has to be learned. Cost accepted: a user who writes
`@<=2.13` meaning the 2.13 line gets something narrower.

**Explicit wildcards rather than implicit prefix matching** — `com.sun.**` to reach a
namespace. The exact reading is rarely what is wanted: a coordinate reported at greater
depth would not match its own shorter form, and every namespace query would carry a `**`.
The trade is real and recorded in [Drawbacks](#drawbacks).

**Materialized archetype membership** — an association table maintained on new reports and
archetype edits. Keeping membership derived avoids a class of staleness bug, but this
remains the escape hatch if the cheaper levers prove insufficient.

**Matching against all reports rather than the latest** — an application that once
depended on a library but no longer does should not be classified by it.

## Implementation History

- 2026-10-30: Initial proposal.
- 2026-11-04: `@` becomes the only constraint introducer; constraint enumeration dropped
  in favor of `||`; expression-level guard added for applications with no dependency
  data; membership preview endpoint added (absorbing the validation-endpoint question);
  version bounds restated as line endpoints; scaling limit moved from Goals to
  Non-Goals.
- 2026-11-10: PURL and VERS evaluated in Alternatives following review. Purl accepted as
  an input form for references and purl-spec's per-type normalization adopted; vers
  rejected as the constraint grammar. Open question on dashes narrowed to npm.
- 2026-11-17: Ecosystem-agnostic authoring replaced by `[ecosystem](…)` blocks with
  native coordinates. Prefix version semantics replaced by VERS comparators, restricted
  to one interval. Explicit `**` dropped in favor of segment prefix matching. Version
  ordering delegated to `osv-scalibr/semantic`, with behavior verified against v0.5.3 and
  a validation wrapper added for the malformed-version gap it leaves. Candidate partition
  by provider added to the scaling analysis. Test plan reweighted toward name matching
  and validation.
- 2026-11-17: Editorial pass. Implementation-level citation reduced throughout; blocks
  made non-nestable in the grammar; ecosystem coverage and the unmatchable-provider
  consequence stated; preview separated into validation (cheap, per keystroke) and
  membership (whole-portfolio, on demand); open question on the single-interval
  restriction folded into [Version constraint](#version-constraint).
- 2026-11-18: Terminology pass. The cheap onboarding pass is called **tech-discovery**
  throughout rather than "discovery", and the report it produces is called a
  tech-discovery rather than an analysis — "analysis" now means a full analyzer run only.
- 2026-11-18: Rewritten for a reader who already knows the architecture. Summary leads
  with the three changes to existing behavior rather than with syntax; specificity moved
  to the front of Matching Semantics as the algorithm this breaks; explanation of current
  Konveyor behavior cut throughout; speculative dependency-count arithmetic replaced by a
  single stated estimate; Alternatives and Test Plan compressed. No design decisions
  changed.
