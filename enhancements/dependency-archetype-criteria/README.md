---
title: dependency-archetype-criteria
authors:
  - "@jortel"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-10-30
last-updated: 2026-11-20
status: provisional
see-also:
  - "/enhancements/assessment-module/README.md"
  - "/enhancements/tech-discovery-dependencies/README.md"
---

# Dependency-Based Archetype Criteria

Archetypes classify applications by criteria tags alone, which works when the portfolio
has been curated but cannot express the most objective fact analysis already discovers:
what an application depends on.

This enhancement adds **dependency criteria** — a boolean condition over the
dependencies reported by analysis and tech-discovery — usable alone or alongside
criteria tags.

Two designs for expressing that condition are specified and the choice is open: an
**expression language**, which the body of this document describes, and
[**structured criteria**](#structured-criteria-an-alternative-to-the-expression-language),
an And/Any/Not rule set following the existing `RoleMapper` pattern. Everything below the
boolean layer is identical between them.

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
- **Membership becomes coupled to analysis having run.** Dependency data exists only
  after a full analysis today, which is too rare for this feature to be useful on a
  portfolio that has merely been onboarded.
  [Dependency Reporting from Tech-Discovery](../tech-discovery-dependencies/README.md)
  closes that gap and is a prerequisite; it is a separate enhancement because it changes
  behavior for users who never adopt dependency criteria. Until it lands, an application
  that satisfies an expression but has not been analyzed does not match, and nothing
  distinguishes that from not satisfying it. See
  [Source of Dependency Data](#source-of-dependency-data).
- **Membership resolution gets roughly 20× more expensive.** Unmeasured. My estimate is
  that it becomes a problem near 1000 applications, and `/applications` — which is not
  paginated — reaches that first. See [Resolving Membership](#resolving-membership).

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
copied from a `pom.xml` or a `package.json`. Constraints are uniform
because every ecosystem's *range* dialect is a separate published spec, and implementing
four of them buys little. A Java clause and an npm clause differ in their package names
and in nothing else.

Blocks compose, so one expression can span ecosystems:

```
[maven](org.springframework.boot:spring-boot@>=2|<3) && [npm](react@>=16|<17)
```

Version ordering and boolean evaluation are delegated to existing implementations.
Name splitting and normalization follow published per-ecosystem rules but are implemented
here, because no Go library exposes them. See [Implementation](#implementation).

## Motivation

Tag-based criteria depend on the portfolio being tagged accurately and consistently,
which is the slowest and least reliable part of onboarding a large estate. Tags are
applied by people, drift as applications change, and are frequently absent for
applications that have only just been discovered.

Analysis already produces a precise, machine-generated dependency inventory, refreshed
on every run. Archetypes are how Konveyor groups applications for assessment, review,
and target profiles, so the inability to say "every application on Spring Boot 2" forces
users to maintain by hand a classification the tool could derive.

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
- Define behavior when an application has no dependency data, and identify what has to
  be true elsewhere for criteria to be useful on a portfolio that has been onboarded but
  not fully analyzed.
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
- **SBOM or advisory ingestion.** No SBOM is parsed, no Package URL is accepted as an
  authoring form, and no dependency is rendered *as* a purl. Generating an expression
  from an advisory is a worthwhile feature and is deliberately left out; see
  [Advisory-driven criteria](#advisory-driven-criteria).
- **Changing how dependencies are produced.** Making tech-discovery report them is a
  prerequisite and a separate enhancement; see
  [Dependency Reporting from Tech-Discovery](../tech-discovery-dependencies/README.md).
- **Tag inference from dependencies.** A separate concern.
- **Scaling beyond roughly 1000 applications.** Levers are identified in
  [Resolving Membership](#resolving-membership); none is implemented here.

## Proposal

### User Stories

DAC005 is retired; see [Alternatives](#alternatives). Identifiers are not reused.

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
artifacts, so that I can express "anything importing WebLogic". How far this reaches is
ecosystem-specific; see [Dependency reference](#dependency-reference).*

#### DAC006

*As an Architect, I want to write package names in the form my build files already use,
so that I can paste a coordinate out of a `pom.xml` or a `package.json` instead of
translating it.*

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
expression. Adding an ecosystem is a row in this table plus a [dialect](#dialects).

There is no default ecosystem. An expression that does not declare one cannot be
evaluated correctly.

#### Dependency reference

```
ref         = coordinates [ '@' constraint ]
coordinates = segment ( separator segment )*
segment     = text | '*'
separator   = per ecosystem — see below
```

Coordinates are a sequence of **segments** matched against the reported dependency name.
Both sides are reduced to segments by the same ecosystem-specific rules, so a reference
and a stored name are always compared in the same terms.

**Separators are per-ecosystem, not universal.** Each ecosystem names its packages under
its own conventions, and a character that separates coordinates in one is an ordinary
character in another:

| ecosystem | separators | why |
|---|---|---|
| maven | `.` `:` | the analyzer may emit `group:artifact` or `group.artifact`; both must segment identically |
| golang | `/` `.` | module paths are `/`-delimited and the host is dotted |
| npm | `/` only | a scope may contain a dot — `@jsonjoy.com/util` is two segments, not three |
| pypi | none | PEP 503 makes the whole distribution name one token |

| provider | reported name | segments |
|---|---|---|
| java | `com.fasterxml.jackson.core:jackson-core` | `com fasterxml jackson core jackson-core` |
| java | `com.fasterxml.jackson.core.jackson-core` | *identical* |
| go | `github.com/jortel/go-utils/v2` | `github com jortel go-utils v2` |
| npm | `@babel/core` | `@babel core` |
| python | `zope.interface` | `zope-interface` *(single segment, per PEP 503)* |

**Names match by segment prefix.** A reference matches when its segments are a leading
subsequence of the dependency's segments. `com.sun` matches `com.sun.xml.bind` and
`com.sun.jersey:jersey-core`; it does not match `com.sunflower`, because `sun` and
`sunflower` are different segments. A reference naming every segment matches only that
dependency.

This is how `weblogic` expresses "anything importing WebLogic" with no wildcard syntax,
and how a reference tolerates a dependency name carrying more coordinate components than
the reference does — see [Coordinate depth](#coordinate-depth).

`*` matches exactly one segment, and there is no `**`. It is most useful in a
non-trailing position (`com.*.internal`), since trailing segments are already open. A
trailing `*` is accepted rather than rejected, and it is not redundant: it requires the
candidate to carry at least one further segment, so `com.sun.*` matches
`com.sun.xml.bind` but not `com.sun` itself. That is rarely what an author means, and it
is the one place in the syntax where a reference is *narrower* than it looks.

Because segment count is ecosystem-specific, so is the reach of a prefix. Maven and
golang names are multi-segment and prefix-match freely; npm reaches only the scope
boundary; a pypi name is atomic and matches whole or not at all. Nowhere does a prefix
reach *inside* a segment — `-` is never a separator, in any ecosystem.

**`@` is the only constraint introducer.** `:` is a separator and nothing else, which is
what lets Maven's `.` and `:` be interchangeable. It also removes the arity ambiguity in
Maven's colon form — `G:A:V`, `G:A:P:V`, and `G:A:P:C:V` cannot be told apart without
counting, so `io.netty:netty-transport-native-epoll:jar` would silently parse `jar` as a
version. The parser rejects `group:artifact:1.2` with *"use `group:artifact@1.2`"*.

A **leading** `@` is part of the name — npm scopes retain it (`@babel/core` → segments
`@babel core`). Only a non-leading `@` introduces a constraint, so `@babel/core@7` is
scope `@babel`, name `core`, constraint `7`.

**Normalization is per-ecosystem, and the rules are adopted rather than invented.**
Splitting alone does not cover every convention: PEP 503 declares `-`, `_`, and `.`
equivalent within a Python distribution name, and case is significant in some ecosystems
and not others. Each rule is specified per package type in purl-spec's
[type definitions](https://github.com/package-url/purl-spec/tree/main/types), and the hub
adopts them. Because the block declares the ecosystem, the reference is reduced once at
parse time and the stored name under the same rules at match time.

The rules are adopted, not an implementation of them; see
[Dialects](#dialects).

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
| `@>=1.2\|!=1.4.1\|<2` | a range with a hole |
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
`2.13.0-rc1`, `2.13-SNAPSHOT`, and `2.13rc1`.
This is not a rule defined here — it falls out of each ecosystem's published ordering.
It has two severe consequences, both in Go, one at each end of a range; see
[Go major versions](#go-major-versions).

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
[golang]( github.com/golang-jwt/jwt@>=v4|<v5.0.0-0 )
```

A leading `v` is accepted on either side and is not significant: `@>=v4` and `@>=4` are
the same constraint, and both compare correctly against a stored `v4.5.0`.

Constraining the version also covers the two cases the path cannot express: v0 and v1
share the unversioned path, and `+incompatible` modules carry a major in the version
with no path suffix.

**Bound a major with `<vN.0.0-0`, not `<vN`.** A prerelease of the next major sorts
below it, and the next major's module path prefix-matches the same reference. So
`github.com/golang-jwt/jwt@>=v4|<v5` admits `github.com/golang-jwt/jwt/v5` at
`v5.0.0-rc.1`: the coordinates match, and `v5.0.0-rc.1 < v5.0.0`. `-0` is the lowest
possible prerelease of `5.0.0`, so `<v5.0.0-0` excludes every v5 prerelease and keeps
all of v4. The verbose form is the correct one wherever a major is being excluded.

**The v0 trap.** The same rule bites harder at the lower bound. An untagged Go module
has a pseudo-version — `v0.0.0-20210101120000-abcdef123456` — which is a *prerelease* of
`0.0.0` under semver and therefore sorts **below** `v0`. So `@>=v0|<v1` matches almost no
v0 module. Write `@<v1.0.0-0`, or `@>=v0.0.0-0|<v1.0.0-0` if a lower bound is wanted.

Both traps are the prerelease rule behaving as specified. They are severe in Go and
nowhere else because Go's dominant version form is a prerelease and its major version is
also a path segment, so the version constraint is carrying weight the coordinates cannot.

#### Advisory-driven criteria

Two of the four motivating cases — a vulnerable Jackson, an EOL Struts — begin with an
advisory, so it is worth stating what this proposal does *not* do about that and why.

Advisories state affected ranges as half-open intervals with explicit bounds, which is
the model this syntax already uses:

| source | affected range | equivalent |
|---|---|---|
| OSV | `events: [{introduced: "2.0.0"}, {fixed: "2.15.0"}]` | `@>=2.0.0\|<2.15.0` |
| GitHub / GHSA | `vulnerable_version_range: ">= 2.0.0, < 2.15.0"` | the same, after `,` → `\|` |
| NVD / CVE | `versionStartIncluding` / `versionEndExcluding` | the same |

OSV's `introduced`/`fixed` is semantically half-open, so the translation is exact rather
than approximate. An OSV record also carries the coordinates in two forms — a purl, and
an `ecosystem` plus a `name` that for Maven is already `group:artifact`, the native form
this syntax takes verbatim.

So every field needed to write the expression is present and machine-readable, which is
an argument against an authoring shortcut and for a generator. Nobody hand-copies three
fields out of a JSON record; given a CVE or GHSA identifier the UI could fetch the OSV
entry and emit
`[maven]( com.fasterxml.jackson.core:jackson-databind@>=2.0.0|<2.15.0 )` directly.

That is a separate feature and is out of scope here. It is named because the obvious
alternative — accepting Package URLs as a paste-in reference form — was evaluated and
dropped: a purl carries coordinates but cannot carry a range, so it addresses the half
of the advisory workflow that was never the hard half. See
[Alternatives](#alternatives).

One wrinkle for whoever builds the generator: an advisory often reports several affected
intervals rather than one, because a fix is backported to branches still in support.
Log4Shell is three — `[2.0-beta9, 2.3.1)`, `[2.4, 2.12.2)`, and `[2.13.0, 2.15.0)` — not
the single range its headline suggests. Each interval becomes its own reference, OR'd
together, since a constraint carries one interval. The mapping is mechanical; what is
not mechanical is noticing that there is more than one.

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
            maven dialect: split on . :      -> [com, redhat, fuse, fuse-core]
            maven dialect: normalize         -> unchanged (case-sensitive)
name        com.redhat.fuse.fuse-core
            maven dialect: split on . :      -> [com, redhat, fuse, fuse-core]
            maven dialect: normalize         -> unchanged (case-sensitive)
                                             -> segment match: true

constraint  >=7.9|<7.12
            parse comparators                -> lower 7.9 incl, upper 7.12 excl
            equivalently                     == vers:maven/>=7.9|<7.12
version     7.11.1
            validate as a Maven version      -> ok
            compare under ecosystem "Maven"  -> in range: true
```

Note that the analyzer reported the name dot-joined, with nothing marking where the
group ends and the artifact begins. Both sides go through the same dialect, so the
boundary never has to be recovered — which is also why the stored name cannot be
rendered as a purl.

**Segment matching.** The reference matches when it has no more segments than the
candidate and every one of its segments equals the candidate's in position, with `*`
equal to any single segment. Trailing candidate segments are unconstrained:

| reference | candidate | match |
|---|---|---|
| `com.redhat:fuse-core` | `com.redhat.fuse-core` | yes — maven treats `.` and `:` alike |
| `com.redhat` | `com.redhat.fuse-core` | yes — trailing segments open |
| `com.*.fuse-core` | `com.redhat.fuse-core` | yes — `*` is exactly one segment |
| `com.*` | `com.redhat.fuse-core` | yes — `*` matches `redhat`, rest is trailing |
| `com.redhat:fuse` | `com.redhat.fuse-core` | no — different segments; `-` does not separate |
| `com.redhat:fuse-core:jar` | `com.redhat.fuse-core` | no — reference is deeper than the candidate |
| `com.redhat.*` | `com.redhat` | no — a trailing `*` still requires a segment |

Both sides reduce to `(type, segments, constraint)`. That shape resembles a purl and
deliberately is not one: the reference half may be a prefix or carry `*`, neither of
which a purl can express, and the candidate half has no recoverable namespace boundary.
The constraint half *is* VERS and serializes as such.

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
# Log4Shell (CVE-2021-44228) exposure: three affected branches, one reference each.
# 2.3.1, 2.12.2 and 2.15.0 are the fixes, so each interval stops below its own.
[maven](
  org.apache.logging.log4j:log4j-core@>=2.0-beta9|<2.3.1
  || org.apache.logging.log4j:log4j-core@>=2.4|<2.12.2
  || org.apache.logging.log4j:log4j-core@>=2.13.0|<2.15.0
)
```

```
# Stranded on an old Go major; note the -0 upper bounds and no >=v0 lower bound
[golang](
  github.com/golang-jwt/jwt@>=v4|<v5.0.0-0
  || github.com/go-chi/chi@<v5.0.0-0
  || github.com/jortel/go-utils@<v1.0.0-0
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

One field on the REST resource, alongside the existing `criteria`, omitted when empty.

```go
type Archetype struct {
	Resource     `yaml:",inline"`
	// ... existing fields
	Criteria     []TagRef `json:"criteria" yaml:"criteria" binding:"dive"`
	CriteriaDeps string   `json:"criteriaDeps,omitempty" yaml:"criteriaDeps,omitempty"`
}
```

`CriteriaDeps` is a string because the expression language stores one. Under
[Structured Criteria](#structured-criteria-an-alternative-to-the-expression-language) it
would be a JSON column holding a rule set; the column is the only part of the model that
differs between the two designs, and neither needs a provider field or a child table.

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
| `pkg:maven/g/a` | Package URLs are not accepted; use `[maven]( g:a )` |
| `[java](…)`, `[Maven](…)` | unknown ecosystem; use `[maven]` |

Users will try the syntax of the ecosystem they know. The comma case matters most: PEP
440 reads `1.2,2.0` as conjunction and this syntax has no comma at all, so accepting it
would silently mean the opposite of what a Python user intends. `[java]` is worth a
specific message because `java` is the provider value stored on the dependency and will
be the first guess of anyone who has looked at the data.

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

*Identical* means the parsed forms serialize to the same canonical string. Whitespace,
line breaks, and redundant parentheses are therefore insignificant, and operand order is
not: `a && b` and `b && a` are the same predicate but are treated as incomparable, so an
application matching both archetypes is assigned to both. The test is deliberately
syntactic — recognizing commutativity is the first step toward the subsumption decision
rejected above, and the product order already declines to rank far more common cases.

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

Dependencies are reported by two kinds of run: a full **analysis**, and — once the
[prerequisite](#prerequisite-dependency-data-from-tech-discovery) lands —
**tech-discovery**. Criteria are evaluated against the dependencies of the
**most recent** report, whichever kind produced it, which is the same scoping the
analysis report endpoints already use. Superseded reports are never considered.

A reported dependency carries a provider, a name, a version, and whether it was reached
indirectly, all passthrough from analyzer-lsp. This proposal interprets provider, name,
and version, which is the extent of the coupling it introduces.

An application that has had neither an analysis nor a tech-discovery has no dependency
data, so no expression is satisfied and it does not match — including for negated
expressions. The consequence is a behavior change for an archetype that previously
matched on tags alone and later gains an expression: every member with no dependency data
drops. See [Drawbacks](#drawbacks).

### Prerequisite: Dependency Data from Tech-Discovery

Dependency criteria are only useful if dependency data is widely available, and today it
is produced only by a full analysis, which is not routinely run across a portfolio.
Tech-discovery is run broadly, but it currently discards dependency information.

[Dependency Reporting from Tech-Discovery](../tech-discovery-dependencies/README.md)
proposes the addon change that closes this. It is a prerequisite for this feature being
useful in practice rather than part of it, for two reasons: the change is confined to
`konveyor/tackle2-addon-analyzer` and touches nothing here, and its principal consequence
— a tech-discovery report supersedes a prior analysis, so editing an application discards
its insights — affects every user, including those who never define a dependency
expression. Bundling it would have meant a reviewer objecting to that behavior had to
block this feature as well.

This proposal is implementable without it, and would simply be restricted to applications
that have had a full analysis.

## Design Details

### Implementation

Expression parsing and matching depend on neither the API nor the assessment packages, so
they can be exercised in isolation.

The principle is to delegate wherever a published specification has a maintained Go
implementation, and to implement the published rule ourselves where it does not. Two of
the three published specs this design leans on have no usable library:

| concern | owner | published spec? |
|---|---|---|
| boolean evaluation, block scoping | `github.com/PaesslerAG/gval` — already a hub dependency | — |
| version ordering, per ecosystem | `github.com/google/osv-scalibr/semantic` | yes, and implemented |
| name splitting and normalization | in-house, per-ecosystem dialect | yes — purl-spec, as oracle only |
| the `\|`-separated constraint and its interval | in-house | yes — VERS, as oracle only |
| version syntax validation | in-house | no |
| segment prefix / `*` matching | in-house | no |

The delegation is narrower than it looks, and that is the right trade: the one concern
that would have been both large and silently wrong — ordering Maven qualifiers, PEP 440
epochs, and Go pseudo-versions — is the one with a maintained implementation. What
remains in-house is small and testable, and two parts of it have a citable oracle even
though no library exposes it. See [Dialects](#dialects).

#### Boolean layer and blocks

`internal/task/selector.go` already evaluates boolean expressions over `gval`'s
propositional-logic language, resolving each reference to a boolean. It is refactored so
the boolean layer is one implementation with a pluggable reference resolver, and
dependency criteria supply a second resolver alongside the existing one.

`gval` extends to `[ecosystem](…)` through its prefix-extension hook: `[` carries no
meaning in the propositional-logic language, so claiming it displaces nothing. The
extension reads the ecosystem, parses the parenthesized sub-expression, and compiles each
reference against that ecosystem.

#### Dialects

Everything that differs between ecosystems is reached through a **dialect**, selected at
parse time from the block header. Its shape is an implementation detail; what it must
provide is not:

- **The purl type** — `maven`, `npm`, `pypi`, `golang` — which names the dialect and is
  what an expression writes.
- **The provider** the analyzer emits for this ecosystem, used to partition candidates
  before any matching work. See [Resolving Membership](#resolving-membership).
- **The comparator ecosystem key** — the `osv-scalibr/semantic` name, which is neither
  of the above and is why the mapping is a literal table.
- **Reduction of a name to segments**, covering both splitting and normalization, applied
  identically to a reference at parse time and to a stored name at match time.
- **Validation of a version string** under this ecosystem's rules, which the comparator
  does not do; see [Version comparison](#version-comparison).

Splitting and normalization belong together rather than as separate steps. They are
always applied in sequence to both sides, and separating them invites applying one
without the other — which would silently make the two sides incomparable. Keeping them a
single reduction makes that error unrepresentable.

What stays shared across dialects is the matcher: comparing two segment lists by prefix,
with `*` matching one segment. There is one of those, not four.

**Normalization is implemented, not imported.** purl-spec specifies the rules per package
type, but no Go library exposes them at the component level — `osv-scalibr/purl` exports
only `FromString`, `String`, and `QualifiersFromMap`, and `packageurl-go` normalizes only
as a side effect of parsing a whole `PackageURL`. Reaching the rules through a library
therefore means round-tripping a purl string, which cannot represent a reference carrying
`*` or a bare namespace prefix, and cannot represent a candidate at all, since a stored
name has no recoverable group/artifact boundary. The rules are small — PEP 503 collapse
and case folding for pypi, case preserved for maven, npm, and golang — so each dialect
implements them against purl-spec as the specification, with the test plan citing it as
the oracle.

**Case is preserved outside pypi, and golang needs saying out loud.** Only
`pypi-definition.json` declares `case_sensitive: false` with explicit
`normalization_rules`; maven, npm, and golang all declare `case_sensitive: true`. npm
says why — the 2015 rule forbidding uppercase grandfathered the names that already had
it. `golang-definition.json` is self-contradictory: it sets `case_sensitive: true` and
also carries a note that the namespace and name "shall be lowercased", on a type
annotated as predating Go modules. Go module paths are case-sensitive, so the flag is
followed and the note is not. Folding Go would be friendlier to someone typing
`github.com/masterminds/semver` from memory, and it is rejected anyway: it would make
two distinct module paths one reference, and the cost of that is silent
misclassification rather than a failed lookup.

Adding an ecosystem is a dialect plus a row in the mapping table.

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
  because splitting and normalization are per-ecosystem, a prefix is only selective
  against a *normalized* name column, and the normalization differs by provider.
- **Caching the matched set**, invalidated on archetype write and on a new report.

For baseline: `MembershipResolver` already loads the entire application-tag join on every
request that resolves membership, including single application reads. The whole-portfolio
scan is the status quo; this adds a second scan roughly 20× larger.

Benchmarks against current tag-only resolution are owed before this moves off
`provisional`.

### Test Plan

The budget follows the ownership split above. The comparator is a maintained library
tested against its own specs, so it is verified rather than tested. Name matching,
normalization, and validation carry the weight: none has a library, and only
normalization has a published oracle to cite.

Three cases are called out because a regression in them is silent rather than loud: an
unparseable version must not satisfy a *negated* reference; `@>=v0|<v1` must fail to match
a Go pseudo-version while `@<v1.0.0-0` matches it; and with no archetype using an
expression, selection must produce exactly the current result.

| area | cases |
|---|---|
| comparator conformance | contract test against `osv-scalibr/semantic` for the orderings relied on here, plus `2.13.5 > 2.13`; fails loudly if an upgrade changes ordering |
| version validation | `banana`, `${project.version}`, `LATEST`, `""`, a Maven property placeholder — each rejected by the hub's validator, and a dependency carrying one satisfies no reference, negated included |
| name matching | segment prefix across separator forms; `com.sun` matches `com.sun.xml.bind`, not `com.sunflower`; reference matching greater reported depth; `@jsonjoy.com/util`; `@babel/core@7`; `com.foo:3d-view`; Go path suffixes; non-trailing `*`; a trailing `*` requiring one more segment |
| normalization | PEP 503 equivalence (`apache_airflow`, `apache.airflow` → `apache-airflow`) and pypi case folding; case preserved everywhere else — `github.com/Masterminds/semver` and `github.com/masterminds/semver` are distinct, as are `JSONStream` and `jsonstream` in npm; no leakage across ecosystems — `-` stays load-bearing in Maven and Go |
| expression parsing | all four ecosystems; nested and negated blocks; every row of the rejection table, asserting the suggested equivalent; `1.2,2.0` and `group:artifact:1.2` rejected under any reading; a reference outside a block is a parse error |
| constraint semantics | each comparator and `\|`; `@2` does not match `2.7.18` but `@>=2\|<3` does; a second interval rejected with the `\|\|` equivalent; `>=2.13` excludes `2.13.0-rc1` and `2.13-SNAPSHOT`; per-ecosystem post-release forms that sort *above* their release are included — Maven `2.13-sp`, PEP 440 `2.13.post1` |
| Go idioms | `@>=v4\|<v5.0.0-0` selects `/v4` and excludes both `v5.0.0` and `v5.0.0-rc.1`, where `@>=v4\|<v5` admits the latter; the v0 pseudo-version case above; `v2.0.0+incompatible` behaves as `v2.0.0` |
| per-ecosystem splitting | the separator table: `com.foo:bar` and `com.foo.bar` segment identically in maven; `@jsonjoy.com/util` is two npm segments, not three; `zope.interface` is one pypi segment; `-` never separates anywhere |
| membership (`test/binding`) | dependency criteria only, tags only, both |
| no dependency data | an application with neither an analysis nor a tech-discovery must not match `!foo`, nor a disjunction containing it — the case where evaluating references individually gives the wrong answer |
| specificity | each row of the product-order table; the regression property above; dependency-only vs. tag-only, which the tag-only rule got wrong |
| report currency | an archetype matches on the latest report only; a newer report supersedes an earlier matching one, whichever kind produced it |

### Upgrade / Downgrade Strategy

A migration adds the `archetype.CriteriaDeps` column. No existing data is modified and no
existing archetype changes behavior: an empty expression resolves as it does today.

Downgrade drops the column. Archetypes revert to tag-only criteria, which widens their
membership because the dependency gate is removed. This should be called out in release
notes rather than mitigated; preserving a filter the schema can no longer express is not
possible.

## Open Questions

1. **Expression language or structured criteria?** *This blocks `implementable`.* Two
   candidate designs are specified: the expression language described throughout this
   document, and the And/Any/Not rule set in
   [Structured Criteria](#structured-criteria-an-alternative-to-the-expression-language).
   The structured form expresses every example in this document, follows an existing hub
   pattern, and is simpler for a UI to build; the expression language is terser, keeps
   nesting available without a schema change, and has fewer degenerate states to guard.
   Nothing below the boolean layer differs between them. The whole of what the
   structured form cannot express is tabulated in
   [The gap, in full](#the-gap-in-full) — it is the first thing to read when weighing
   the two. The rest of this document is written against the expression language because
   it is the more detailed of the two, not because the choice is settled.

2. **What does membership resolution actually cost?** *This blocks `implementable`.* The
   20× figure in [Resolving Membership](#resolving-membership) is arithmetic, not
   measurement, and the ~1000-application threshold derived from it is a guess.
   Benchmarks against current tag-only resolution are owed, and which of the three levers
   is needed — if any — should follow from them rather than from argument.

## Drawbacks

Dependency criteria couple archetype membership to analysis, two subsystems that are
currently independent. An archetype that looks correct can match nothing because the
portfolio has not been analyzed, and the failure is silent — an empty member list, not an
error. The same coupling makes a dropped member ambiguous in a way a tag criterion never
is: adding a criteria tag drops applications that genuinely lack it, while adding an
expression also drops applications that satisfy it but have not been analyzed.

**Segment prefix matching leans toward over-matching rather than under-matching.** An
imprecise reference widens, where an exact-match rule would have narrowed. This is not
a new class of error — omitting a criteria tag widens an archetype the same way — but
the mechanism differs: a tag list shows what was written, while `org.apache.struts`
reads as more specific than it is. The mitigation is the same one tag criteria have:
edit the archetype and look at the membership it produces.

**How far a prefix reaches is an ecosystem property, and users will not expect it to be.**
`com.sun` reaches a whole Java namespace, while `lodash` matches one npm package and
nothing else. Each rule is defensible on its own — npm cannot split on `.` without
breaking `@jsonjoy.com/util`, and `lodash.merge` and `lodash-merge` are genuinely
different packages — but the asymmetry is invisible in an expression and has to be
learned from documentation. Case behaves the same way: `pypi` folds it and the other
three do not, so a Go or npm reference typed in the wrong case matches nothing and
reports that as an empty archetype.

The pressure on application listing is the drawback I am least able to bound. Membership
is calculated when applications are fetched, so listing a large estate means resolving
dependency criteria across it, and the effect is hard to predict in advance. Adopting this at scale means accepting a latency regression
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

**Structured criteria instead of an expression language** — not rejected. It is a live
alternative to the expression language and is specified in
[Structured Criteria](#structured-criteria-an-alternative-to-the-expression-language),
below.

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
Rejected as the model because a purl is an identifier, not a pattern: there is no wildcard
layer, so `com.sun` cannot be written as one, and a glob over purl is no longer a purl —
losing the interop that motivated it. The interop also runs one way: matching purl against
purl means rendering stored `(provider, name, version)` as purls, which needs a
group/artifact boundary the stored form does not carry. And percent-encoded
`pkg:maven/...?vers=...%7C...` is a machine format, not hand-authored.

**Package URLs as a paste-in reference form** — a narrower version of the above, where a
purl is accepted alongside the block form for a single fully-qualified reference. Carried
in earlier revisions of this proposal and since dropped. The justification was pasting
from an advisory, and a purl cannot carry a range, so it covers coordinates — the half
that was never hard — and not the affected interval, which is the half that is. Keeping
it would have meant either a second full-fidelity authoring form, with a canonicalization
rule for which one is stored and displayed, or an extended purl-shaped syntax that is
purl-incompatible in exactly the cases where interop was the point. Neither is worth it
for a workflow better served by a generator; see
[Advisory-driven criteria](#advisory-driven-criteria).

Adopted from these specs: VERS comparator syntax, purl types as block names, purl-spec's
per-type normalization rules. Not adopted: multi-interval VERS containment, purl as a
reference form, purl as the internal representation. Worth reopening if SBOM ingestion
lands, since the hub would then hold purl-typed data and the group/artifact objection
would disappear. OSV and CPE were also considered — OSV is an advisory schema rather than
an expression language, CPE identifies vendor and product rather than build coordinates.

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

**A membership preview endpoint** — `POST /archetypes/membership`, accepting unsaved
`{criteria, criteriaDeps}` and returning the matching applications and their count, so
the UI could show a delta before an archetype is saved. Carried in earlier revisions and
dropped. Membership is derived and archetypes are editable, so saving and viewing the
result already shows the same set, one step later; the endpoint bought ordering, not
information. It was also the most expensive call in the proposal — a whole-portfolio
resolution, invoked speculatively while authoring — and it arrived justified by problems
that turned out to be ordinary: over-matching behaves no differently from omitting a
criteria tag, and the drop of members with no dependency data largely disappears once
tech-discovery reports them. Worth adding if the UI asks for it; the resolution logic is
the same either way.

**Materialized archetype membership** — an association table maintained on new reports and
archetype edits. Keeping membership derived avoids a class of staleness bug, but this
remains the escape hatch if the cheaper levers prove insufficient.

**Matching against all reports rather than the latest** — an application that once
depended on a library but no longer does should not be classified by it.

## Structured Criteria: an Alternative to the Expression Language

A live alternative to the expression language, not a rejected one. See
[Open Questions](#open-questions).

Criteria are a list of rules rather than a parsed expression. The schema follows
`RoleMapper` in `internal/auth/ldap.go`, which already maps LDAP groups to roles with
`And`/`Any` pattern lists, extended with `Not`:

```go
type Rule struct {
	Ecosystem string   // maven, npm, pypi, golang
	And       []string // every reference must match
	Any       []string // at least one must match
	Not       []string // none may match
}

// on the archetype
CriteriaDeps []Rule   // every rule must be satisfied
```

**Rules are ANDed.** This is the one divergence from `RoleMapper`, which ORs its rule
set, and it is forced rather than chosen: a rule holds one `Any` list, and the polyglot
case below needs two — one per ecosystem — so they have to conjoin. Someone who knows
`RoleMapper` will assume OR and read every multi-rule archetype backwards, so the
divergence has to be stated wherever the schema is documented.

A reference is written exactly as it is in the expression language —
`<coordinates>@<constraint>`, ecosystem-native coordinates and a VERS constraint. Only
the boolean layer differs.

### Worked examples

The Spring Boot cohort, which needs a disjunction nested inside a conjunction with a
negation alongside it:

```yaml
criteriaDeps:
  - ecosystem: maven
    and:
      - org.springframework.boot:spring-boot@>=2|<3
      - org.springframework.boot:spring-boot-starter-web@>=2.0|<2.8
      - com.fasterxml.jackson.core:jackson-databind@>=2.9|<2.14
    any:
      - javax.servlet:javax.servlet-api
      - javax.persistence:javax.persistence-api
    not:
      - jakarta.servlet:jakarta.servlet-api
```

The polyglot case, which is what forces rules to conjoin:

```yaml
criteriaDeps:
  - ecosystem: maven
    and: [org.springframework.boot:spring-boot@>=2|<3]
    any: [javax.servlet:javax.servlet-api, javax.persistence:javax.persistence-api]
    not: [jakarta.servlet]
  - ecosystem: npm
    and: [react@>=16|<17]
    any: [webpack@>=4|<5, grunt, gulp]
```

A negated block is a rule with only `Not`. `![npm](react)` is
`{ecosystem: npm, not: [react]}`, and the per-block guard gives it the same meaning it
has in the expression language: the application has npm dependencies, none of them React.

### The gap, in full

Every gap is the same thing in different clothing: a disjunction anywhere other than
inside a single `Any` list. **None of them appears in the six worked examples in this
document.**

| not expressible directly | example | CNF rescues it? | realism |
|---|---|---|---|
| disjunction of conjunctions that do not factor | `(a && b) \|\| (c && d)`, sharing nothing | yes, as four `Any` pairs | low — see below |
| negation of a compound | `!(a && b)` — i.e. `!a \|\| !b` | no | very low |
| disjunction mixing positive and negated terms | `a \|\| !b` | no | very low |
| disjunction across ecosystems | `[maven](a) \|\| [npm](b)` | no | low — polyglot cases are conjunctive |
| nesting generally | `a && (b \|\| (c && d))` | yes, as CNF | low |

CNF "rescue" is technical, not practical. `(a && b) || (c && d)` becomes:

```
any: [a, c]    any: [a, d]    any: [b, c]    any: [b, d]
```

Nobody writes that by hand, and nobody reads the intent back out of it. The accurate
claim is that the structured form expresses what the examples need *directly*, and the
rest only after a transformation users will not perform.

**Why the first row is rare.** `(a && b) || (a && c)` factors to `a && (b || c)`, which
is `And` plus `Any`. In practice the conjunctive part is shared — "an old app server
**and** `javax.servlet`" varies only in the server — so it factors. The gap opens only
when the disjuncts share nothing: *(Spring Boot 2 + Hibernate 5) or (Quarkus 2 +
Panache)*. But an archetype asserts that its members take the same assessment and the
same target profile, and two stacks sharing no markers generally do not. Disjunction of
disjoint conjunctions is a signal of two archetypes, and multi-membership from
incomparable criteria is already normal.

### What looks like a gap and is not

- **Several disjunctions in one ecosystem** — `(a || b) && (c || d)` is two rules with
  the same ecosystem, conjoined.
- **Several ranges on one coordinate** — two `Any` entries naming the same package with
  different constraints, which is how a vulnerability spanning two release lines is
  written.
- **A negated block** — `![npm](react)` is `{ecosystem: npm, not: [react]}`, and the
  per-block guard gives it the same meaning.
- **Tags combined with dependencies** — unchanged; criteria tags AND with dependency
  criteria under either design.

### What does not change

Dialects, reference syntax, VERS constraints, segment matching, version validation, the
comparator, the candidate partition by provider, the scaling analysis, and every
[Matching Semantics](#matching-semantics) rule are identical. A reference still has to be
parsed into coordinates and a constraint whether it arrives inside an expression or as a
string in a YAML list.

What disappears is the boolean parser and the `gval` extension — roughly the
[Boolean layer](#boolean-layer) grammar and
[Boolean layer and blocks](#boolean-layer-and-blocks). That is a real saving but a
narrower one than "no grammar, no parser" suggests.

### Semantics already decided

These apply if this design is chosen:

- **Ecosystem sits on the rule**, mirroring the block, which keeps the candidate
  partition available at parse time.
- **An empty rule — no `And`, no `Any`, no `Not` — matches nothing**, and is also
  rejected by validation on `POST`/`PUT`. Both: validation covers the API, and
  matching-nothing covers seeded, imported, or downgraded data that never passed through
  it. Note the discontinuity this creates — an absent or empty `criteriaDeps` means *no
  dependency criteria* and matches on tags alone, while a list containing one empty rule
  matches nothing. In YAML the difference is one nearly-blank element, and a UI that
  appends a rule row when an ecosystem is picked can produce it by accident.
- **A `Not`-only rule is valid and is not empty.** `MappingRule.Empty()` sums
  `len(And) + len(Any)`; porting it with a `Not` field added but not counted would
  classify `{npm, not: [react]}` as empty, so the user's negation would evaluate to the
  opposite of what they wrote, silently. `RoleMapper.and()` and `any()` already return
  true on empty pattern lists, so the rule evaluates correctly once it is past that
  check.

### The trade

| | structured | expression |
|---|---|---|
| precedent | an existing hub pattern | a second reference syntax in the hub |
| UI cost | form controls over three lists; per-field validation and per-reference typeahead come free | a text editor with live parsing and error-offset display, or a builder producing the same controls *plus* a serializer |
| what the author learns | which list a reference belongs in | a grammar — operators, precedence, blocks, `@` |
| clients | constructed field by field | string assembly |
| errors | per-field, located by construction | parse errors carrying an offset the UI must map back to a cursor |
| nesting | CNF only, not writable by hand | direct, and available without a schema change |
| degenerate states | empty rule, `Not`-only rule — each needs a guard | `[maven]()` is a parse error; unrepresentable |
| storage | JSON column | one string column |

The strongest argument for structured is the UI: most archetypes will be authored by
clicking rather than typing, and a form over three lists is both cheaper to build and
easier to use than an editor for a grammar. It is also the argument that compounds —
typeahead against the coordinates the portfolio actually reports, per-field validation,
and a readable diff of what changed all fall out of the structure, where each would be
separate work against a string.

The counter is that the form degrades as rules multiply. A polyglot archetype is two
rule blocks of three lists; the equivalent expression is six lines a reader takes in at
once. For a user who knows the grammar, typing is faster than filling a form — but that
user is the minority of the people who will touch this.

The case for the expression language is terseness, headroom for nesting without a schema
change, and fewer degenerate states to guard. Every example in this document is
expressible either way.

## Implementation History

- 2026-10-30: Initial proposal.
- 2026-11-04: `@` becomes the only constraint introducer; constraint enumeration dropped
  in favor of `||`; expression-level guard added for applications with no dependency
  data.
- 2026-11-10: PURL and VERS evaluated in Alternatives following review.
- 2026-11-17: Ecosystem-agnostic authoring replaced by `[ecosystem](…)` blocks with
  native coordinates. Prefix version semantics replaced by VERS comparators, restricted
  to one interval. Explicit `**` dropped in favor of segment prefix matching. Version
  ordering delegated to `osv-scalibr/semantic`, verified against v0.5.3, with a
  validation wrapper for the malformed-version gap it leaves.
- 2026-11-18: Scope reduced on review. Tech-discovery dependency reporting extracted to
  [its own enhancement](../tech-discovery-dependencies/README.md) and retained here as a
  prerequisite. Purl dropped as a reference form and advisory ingestion named out of
  scope. Membership preview endpoint dropped; DAC005 retired and identifiers are not
  reused. Coordinate splitting moved from a universal rule into the per-ecosystem
  dialect. Open question on conflicting archetypes removed — multiple membership is
  produced by incomparable tag sets today. Rationale for each is recorded in
  [Alternatives](#alternatives) and [Drawbacks](#drawbacks).
- 2026-11-20: Correctness fixes on review. Case is preserved for maven, npm, and golang
  and folded only for pypi, following purl-spec's per-type definitions rather than the
  retired `PURL-TYPES.rst`; golang's self-contradictory entry is called out. An excluded
  Go major is bounded `<vN.0.0-0`, since `<vN` admits that major's prereleases — the
  mirror of the v0 trap. The Log4Shell example becomes the three affected intervals the
  advisory actually reports, replacing an `!=` carve-out that left the 2.3.x and 2.12.x
  fixes in range. Expression equality for specificity defined as canonical-form
  equality. Trailing `*` specified as accepted and depth-requiring.
- 2026-11-18: Structured criteria reframed from a rejected alternative to a live design
  choice and specified in full. The previous rejection was wrong: it claimed the Spring
  Boot cohort could not be expressed as structured criteria, which an And/Any/Not rule
  set does directly. The choice between the two designs is now an open question blocking
  `implementable`.
