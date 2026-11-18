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

Archetypes currently classify applications using Criteria Tags alone. An application
is a member of an archetype when the archetype's criteria tags are a subset of the
application's tags. This works well when the portfolio has been curated with tags, but
it cannot express the single most objective fact analysis already discovers about an
application: what it depends on.

This enhancement proposes **dependency criteria** — a boolean expression over the
dependencies reported by analysis and tech-discovery — that may be used alone or in
combination with Criteria Tags to determine archetype membership.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Membership-resolution benchmarks against the tag-only baseline
- [ ] User-facing documentation is created

## Summary

An archetype gains an optional dependency expression, matched against the dependencies
reported by the application's most recent analysis or tech-discovery:

```
[maven](
  org.springframework.boot:spring-boot@>=2|<3
  && (javax.servlet:javax.servlet-api || javax.persistence:javax.persistence-api)
  && !jakarta.servlet:jakarta.servlet-api
)
```

The expression language has three parts. A **boolean layer** composes references with
`&&`, `||`, `!`, and parentheses. An **ecosystem block** — `[maven](…)` — declares which
ecosystem the references inside it name, once rather than per reference. A **dependency
reference** names one package:

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
because every ecosystem's *range* dialect is a separate published spec with its own
operators and its own pitfalls, and nothing is gained by implementing four of them. So a
Java clause and an npm clause differ in their package names and in nothing else.

An application **matches** the archetype when its criteria tags match (as today) **and**
the dependency expression is satisfied; which of the matching archetypes it is assigned
to is decided by specificity.

Blocks compose, so one expression still spans several ecosystems:

```
[maven](org.springframework.boot:spring-boot@>=2|<3) && [npm](react@>=16|<17)
```

Where a published specification already exists, the design uses an implementation of it
rather than its own — version ordering, boolean evaluation, and name normalization are
all delegated. See [Implementation](#implementation).

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

Dependency data also stays current on its own. Tech-discovery is triggered by an edit to
an application, so once tech-discovery reports dependencies
([Dependency Data from Tech-Discovery](#dependency-data-from-tech-discovery)) an
application's dependency set is refreshed whenever the application changes. Criteria
built on it keep classifying correctly across edits, where tag criteria depend on someone
noticing the change and re-tagging.

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

- Define a dependency expression syntax whose references are `<coordinates>@<constraint>`:
  coordinates written the way the ecosystem's own build files write them, and a single
  constraint grammar — VERS — shared by every ecosystem.
- Define version constraint semantics that behave correctly against the version strings
  each ecosystem actually produces, by delegating ordering to a maintained implementation
  of each ecosystem's rules rather than reimplementing them.
- Extend the Archetype model and API to store dependency criteria alongside Criteria
  Tags, without breaking existing clients.
- Make dependency data available from tech-discovery, so criteria work on a portfolio
  that has been onboarded but not fully analyzed.
- Define **matching and selection** semantics when dependency criteria are combined with
  Criteria Tags, and how they interact with the existing archetype specificity rules.
- Bound the cost of membership resolution: identify the scaling limit, the endpoint that
  reaches it first, and the levers that lift it.

### Non-Goals

- **A general-purpose query language.** This matches dependencies, not incidents, facts,
  or arbitrary application attributes.
- **Ecosystem-native version syntax.** A reference is `<coordinates>@<constraint>`. The
  coordinates are ecosystem-native; the constraint is VERS and is identical in every
  ecosystem. Maven `[1.2,1.3)`, npm `^1.2`/`~1.2`, PEP 440 `>=1.2,<2.0`, and Ruby `~> 2.1`
  are therefore rejected — with a message naming the VERS equivalent, not silently. This
  is a deliberate asymmetry, not an oversight: see
  [Alternatives](#alternatives) for why the native half stops at the `@`.
- **Full VERS constraint lists.** A constraint is one interval; see
  [Version constraint](#version-constraint).
- **Dependency resolution in the hub.** The hub matches what the addon reported; it does
  not resolve or walk a dependency graph. The addon continues to do the resolving —
  including, under this proposal, during tech-discovery.
- **SBOM ingestion** as an alternative source of dependency data. Package URLs are
  accepted as a paste-in authoring form, but no SBOM is parsed and no dependency is ever
  rendered *as* a purl; see [Package URLs](#package-urls-as-an-input-form).
- **Tag inference from dependencies.** Automatically tagging applications based on
  dependencies is a separate concern and is out of scope.
- **Scaling beyond roughly 1000 applications.** The levers are identified in
  [Resolving Membership](#resolving-membership) but none is implemented here; the
  resulting limit is stated in [Drawbacks](#drawbacks).

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

There are two levels. The outer one composes ecosystem blocks; the inner one composes
references within a block.

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

**Blocks do not nest, and a reference appears only inside one.** The separation is what
makes the ecosystem of every reference unambiguous from its enclosing block alone, with
no scoping rule to resolve.

The boolean layer is shared with the addon/extension selector, which defines its own
reference syntax (`tag:category=name`) over the same operators. Both use one
implementation; see [Implementation](#implementation).

#### Ecosystem blocks

A block declares the ecosystem of every reference inside it:

```
[maven]( org.springframework:spring-core@>=6|<7 && !com.sun.xml.bind )
```

The ecosystem name is a Package URL type — `maven`, `npm`, `pypi`, `golang`. Blocks
compose with the boolean layer like any other term, so an expression may contain several,
and a block may appear under a negation.

Declaring the ecosystem rather than inferring it is what lets references be written in
native coordinate form: `org.springframework:spring-core` and `@babel/core` are parsed by
rules that already know which ecosystem they belong to. It also determines, at parse
time, which version comparator applies and which name normalization rules apply — neither
of which has to be rediscovered per candidate.

It has a second effect that matters for cost. A block can only ever match dependencies
whose `Provider` corresponds to its type, so every block is a partition of the candidate
set before any name or version work happens. See
[Resolving Membership](#resolving-membership).

**An unrecognized ecosystem is a parse error**, not an expression that matches nothing.
The mapping is a literal table, because none of the three names agree:

| block | provider | comparator ecosystem |
|---|---|---|
| `maven` | `java` | `Maven` |
| `npm` | `nodejs` | `npm` |
| `pypi` | `python` | `PyPI` |
| `golang` | `go` | `Go` |

These four are the ecosystems for which both halves exist: a provider the analyzer emits
and an ordering the comparator implements. Dependencies reported under any other provider
cannot be matched and are simply never candidates — a limitation worth stating plainly,
since it is invisible in an expression. Adding an ecosystem is a row in this table
plus a coordinate plugin.

There is no default ecosystem. An expression that does not say which ecosystem it means
cannot be evaluated correctly, and guessing is worse than refusing.

#### Dependency reference

```
ref         = coordinates [ '@' constraint ]
coordinates = segment (('.' | ':' | '/') segment)*
segment     = text | '*'
```

Coordinates are a sequence of **segments**, matched against the reported dependency name.
`.`, `:`, and `/` are all segment separators, so the common Java form `group:artifact`
works whether the analyzer emitted a colon or a dot:

| provider | reported name | segments |
|---|---|---|
| java | `com.fasterxml.jackson.core:jackson-core` | `com fasterxml jackson core jackson-core` |
| java | `com.fasterxml.jackson.core.jackson-core` | *identical* |
| go | `github.com/jortel/go-utils/v2` | `github com jortel go-utils v2` |
| python | `zope.interface` | `zope interface` |
| npm | `@babel/core` | `@babel core` |

**Names match by segment prefix.** A reference matches a dependency when the reference's
segments are a leading subsequence of the dependency's segments. `com.sun` matches
`com.sun.xml.bind` and `com.sun.jersey:jersey-core`; it does not match `com.sunflower`,
because `sun` and `sunflower` are different segments. A reference that names every
segment matches only that dependency.

This is what makes `weblogic` express "anything importing WebLogic" with no wildcard
syntax at all, and it is also what makes a reference tolerate a dependency name that
carries more coordinate components than the reference does — see
[Coordinate depth](#coordinate-depth).

`*` matches exactly one segment. It is only meaningful in a non-trailing position
(`com.*.internal`), since trailing segments are already open. There is no `**`: it would
mean the same thing as omitting it.

**`@` is the only constraint introducer.** `:` is a separator and nothing else, which is
what lets the three separators be interchangeable. It also removes the arity ambiguity in
Maven's own colon form — `G:A:V`, `G:A:P:V`, and `G:A:P:C:V` cannot be told apart
without counting, so `io.netty:netty-transport-native-epoll:jar` would silently parse
`jar` as a version. The parser rejects `group:artifact:1.2` with *"use
`group:artifact@1.2`"* rather than misreading it.

A **leading** `@` is part of the name — npm scopes retain it (`@babel/core` → segments
`@babel core`). Only a non-leading `@` introduces a constraint, so `@babel/core@7` is
scope `@babel`, name `core`, constraint `7`.

**Normalization is per-ecosystem, and its rules are not invented here.** Separator
interchange does not reach every ecosystem's naming conventions: PEP 503 declares `-`,
`_`, and `.` equivalent within a Python distribution name, and npm and golang names are
case-folded. Each is a published rule, specified per package type in
[purl-spec](https://github.com/package-url/purl-spec/blob/main/PURL-TYPES.rst), and the
hub adopts those rules rather than defining its own. Because the block declares the
ecosystem, the reference is normalized once at parse time, and the stored name is
normalized under the same rules at match time.

Note that the rules are adopted, not the parser. A reference containing `*` is not a
purl and cannot be round-tripped through one; see
[Implementation](#implementation).

#### Version constraint

The block selects which comparator orders the versions; it does not change how the
constraint is written. A constraint is a list of comparisons separated by `|`, in the
comparator syntax of [VERS](https://github.com/package-url/vers-spec):

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

**A bare version is exact, not a prefix.** `@2` matches a dependency whose version string
is literally `2`, which almost nothing is. The 2 line is `@>=2|<3`. This is more verbose
than a prefix form, and deliberately so: a prefix form has to invent a rule for what the
line beneath `2.0-beta9` is, and it reads `@2.13` as matching `2.13.7`, which is correct
but has to be learned. Comparisons say what they do.

**A constraint is one interval.** At most one lower bound, at most one upper bound, and
any number of `!=`. VERS itself permits several intervals in one list —
`>=1|<2|>=3|<4` is `[1,2) ∪ [3,4)` — but the boolean layer already expresses that, more
legibly and without a second place for disjunction to live:

```
foo@>=1|<2 || foo@>=3|<4
```

Restricting to one interval also makes `|` read one way (*and also*) instead of reading
as conjunction at two constraints and disjunction at four, and it collapses the VERS
validity rules — sorted by version, unique versions, no redundant contiguous
comparators — to a trivial check.

What is accepted remains a strict subset of valid VERS, so a constraint can be emitted as
a canonical `vers:` string without changing the language. The restriction is relaxable
later; it would not be tightenable.

**Prereleases sort below their release, in every ecosystem.** `>=2.13` excludes
`2.13.0-rc1`, `2.13-SNAPSHOT`, and `2.13rc1`; it includes `2.13-sp` and `2.13.post1`.
This is not a rule this proposal defines — it falls out of each ecosystem's published
ordering. It has one severe consequence, in Go; see
[Go major versions](#go-major-versions).

#### Coordinate depth

The analyzer reports a dependency name at whatever coordinate depth it has. Segment
prefix matching means a reference does not have to match that depth: if a Java dependency
is reported as `io.netty:netty-transport-native-epoll:jar:linux-x86_64`, the reference
`io.netty:netty-transport-native-epoll` still matches it, and so does
`io.netty:netty-transport-native-epoll:jar`.

This matters beyond convenience. If the analyzer later begins reporting more
fully-qualified names, old and new reports coexist in the same portfolio for as long as
the old ones survive, and references written against either depth keep working. No
migration of stored expressions is required.

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

Constraining the version is strictly better than matching the path, because it also
covers the two cases the path cannot express: v0 and v1 share the unversioned path, and
`+incompatible` modules carry a major in the version with no path suffix at all.

**The v0 trap.** An untagged Go module has a pseudo-version — `v0.0.0-20210101120000-abcdef123456`
— which is a *prerelease* of `0.0.0` under semver and therefore sorts **below** `v0`. So
`@>=v0|<v1` matches almost no v0 module. Write `@<v1`, or `@>=v0.0.0-0|<v1` if a lower
bound is wanted. This is the prerelease rule behaving exactly as specified; it simply
happens to be severe in the one ecosystem whose dominant version form is a prerelease.

#### Package URLs as an input form

A reference may be pasted as a
[Package URL](https://github.com/package-url/purl-spec), which carries its own ecosystem
and therefore needs no enclosing block:

```
pkg:maven/com.fasterxml.jackson.core/jackson-databind@2.9.10
  ≡  [maven]( com.fasterxml.jackson.core:jackson-databind@2.9.10 )
```

This exists for one workflow. The motivating cases — a vulnerable Jackson, an EOL
Struts — are read off an advisory, and advisories speak purl. A purl's `@version` is a
single concrete version, which is exactly what a bare version means here, so the two
forms agree without a special rule.

It is an input form and not the canonical one. A purl identifies one package and has no
wildcard layer, so `com.sun` cannot be written as one, and the two forms do not compose —
a reference is written either way, not half each. Qualifiers and subpaths are rejected
rather than ignored: they narrow what a purl identifies, and dropping them silently would
make the reference match more than the identifier the user pasted.

Nothing is ever rendered *as* a purl. Doing so would require deciding where the group
ends and the artifact begins in `com.fasterxml.jackson.core.jackson-core`, which is the
ambiguity interchangeable separators exist to sidestep; see
[Alternatives](#alternatives).

#### Evaluating a reference

A reference and a candidate dependency reduce to the same shape before anything is
compared. The block supplies the ecosystem for both halves; the coordinates and the
constraint are then matched independently.

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

Note what the analyzer reported: the name arrived **dot-joined**, with nothing marking
where the group ends and the artifact begins. Separator interchange makes that
irrelevant to matching — and makes rendering the same name *as* a purl impossible, which
is the same fact seen from the other side.

**Segment matching.** The reference matches when it has no more segments than the
candidate and every one of its segments equals the candidate's in position, with `*`
equal to any single segment. Trailing candidate segments are unconstrained:

| reference | candidate | match |
|---|---|---|
| `com.redhat:fuse-core` | `com.redhat.fuse-core` | yes — equal, separators interchangeable |
| `com.redhat` | `com.redhat.fuse-core` | yes — trailing segments open |
| `com.*.fuse-core` | `com.redhat.fuse-core` | yes — `*` is exactly one segment |
| `com.*` | `com.redhat.fuse-core` | yes — `*` matches `redhat`, rest is trailing |
| `com.redhat:fuse` | `com.redhat.fuse-core` | no — `fuse` and `fuse-core` are different segments; `-` does not separate |
| `com.redhat:fuse-core:jar` | `com.redhat.fuse-core` | no — reference is deeper than the candidate |

**Both sides reduce to `(type, segments, version)`.** That shape is purl-shaped and
deliberately not a purl: the reference half may be a prefix or carry `*`, neither of
which a purl can express, and the candidate half has no recoverable namespace boundary.
The constraint half, by contrast, *is* VERS, and can be serialized as such without loss.

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

The two blocks are evaluated independently against their own ecosystem's dependencies
and combined by the outer `&&`, so an application matches only if both halves hold. A
reference is confined to its block: the npm `react` can never match a Maven artifact that
happens to be named `react`, and `!jakarta.servlet` excludes Java packages only.

Blocks are terms like any other, so a block may be negated:

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

There is no provider field, no list, and no child table. A list would be redundant with
`&&`, which the expression language already has; the ecosystem is declared by the block.

`POST`/`PUT` of an archetype parses and validates the expression and returns `400` with
the reason and offset when parsing fails. Validation covers the boolean syntax, the block
ecosystem, each reference, and each constraint, and rejects foreign syntax with a message
naming the equivalent:

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

Users will try the syntax of the ecosystem they know; the parser should teach rather than
misparse. The comma case matters most: PEP 440 reads `1.2,2.0` as conjunction and this
syntax has no comma at all, so accepting it would silently mean the opposite of what a
Python user intends.

The ecosystem name is a purl type and is case-sensitive, which means `[Maven]` and
`[java]` are both errors. This is worth a specific message rather than a generic one,
because `java` is the provider value stored on the dependency and will be the first
guess of anyone who has looked at the data.

#### Membership preview

`POST /archetypes/membership` accepts `{criteria, criteriaDeps}` and returns the matching
application refs and their count, without persisting an archetype. For an existing
archetype the UI diffs the result against current membership to show the delta before
save (DAC005), which is what surfaces the upgrade hazard in
[Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy).

The count matters as much as the list. Because names match by segment prefix, the
characteristic authoring error is an expression that matches *more* than intended — a
reference truncated by one segment widens silently, where under an exact-match rule it
would have failed visibly.

**Preview resolves membership across the whole portfolio and costs accordingly** — the
same as `GET /archetypes/{id}/applications`; see
[Resolving Membership](#resolving-membership). It is therefore an explicit action, not a
keystroke-driven one.

Syntax validation is separable and is not: a malformed expression returns `400` with the
reason and offset without any membership resolution, because parsing fails before the
dependency set is fetched. The same endpoint can be called on every keystroke for
validation alone and on demand for membership, which is why no separate `validate` route
is needed.

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

**No dependency data.** When an application has no dependency data, a non-empty
dependency expression is unsatisfied regardless of its structure. The guard is applied to
the expression as a whole, not to its references: a reference resolving to false would
make `!jakarta.servlet:jakarta.servlet-api` resolve to *true*, so an expression leading
with a negation would match every application with no dependency data — the opposite of
the intent. See [Source of Dependency Data](#source-of-dependency-data).

**The same guard applies per block.** A block whose ecosystem contributed no
dependencies at all to the application's latest report is unsatisfied regardless of
its structure, so a negated block is false rather than true. Without this, `![npm](react)`
would be satisfied by every application whose analysis or tech-discovery never ran an npm
provider — asserting the absence of a React dependency on evidence that was never
collected. The reasoning is identical to the expression-level case one level down, and
the hub cannot distinguish "has no npm dependencies" from "npm was never inspected" from
the reported data alone.

The consequence is worth stating: **a negated block only ever means something for an
ecosystem the latest report actually covered.** `![npm](react)` selects applications
known to have npm dependencies, none of them React — not applications with no front end.

**Unparseable versions.** A candidate dependency whose version string does not parse
under its ecosystem's rules does not satisfy any constraint, including a negated one. The
comparator does not reject such strings on its own; the hub must, and the reason is in
[Implementation](#implementation).

**Step 2 — selection.** Of the archetypes that matched, which ones is the application
assigned to? Not necessarily all of them; see **Specificity** below.

**Direct vs. transitive.** A reference matches both direct and transitive dependencies.
The motivating cases — vulnerable and EOL libraries — usually arrive transitively, and
the analyzer already reports whether a dependency was reached indirectly, so a future
`direct:` modifier is available if one proves necessary. It is not in this proposal;
adding a knob before anyone has asked for it is how this feature grows a configuration
surface it does not need.

**Specificity.** `MembershipResolver.Archetypes()`
(`internal/assessment/membership.go`) assigns an application only to the *narrowest*
matching archetypes. An archetype is discarded when its criteria tags are a strict
subset of another matching archetype's — more criteria tags wins. Archetypes whose tag
sets are incomparable both survive, so multiple memberships are already normal.

Measuring specificity in tags alone does not survive the addition of dependency
criteria. An archetype with an expression and no criteria tags has the empty tag set,
which is a strict subset of every other, so it loses to *any* tag-bearing archetype that
also matches:

| archetype | criteria tags | expression |
|---|---|---|
| A — "Spring Boot 2" | — | `[maven](org.springframework.boot:spring-boot@>=2\|<3)` |
| B — "Java" | `Language=Java` | — |

An application tagged `Language=Java` running Boot 2 matches both, and A is discarded —
wrong, since A's expression is a real constraint that the tag-subset rule cannot see. A
dependency-only archetype would produce members only when nothing tagged also matched,
which defeats half of the feature.

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

For the A/B case above the product order does not rank A above B — it declines to rank
them, so both are assigned. Ranking a tag against an expression would need a common scale
of specificity that does not exist, and declining to invent one is consistent with how
the algorithm already treats incomparable tag sets. Not discarding A is the whole of the
fix.

When no archetype uses an expression the expression dimension is always equal, so this
reduces exactly to the current tag-subset rule and no existing behavior changes.

### Source of Dependency Data

Dependencies are reported by two kinds of run: a full **analysis**, and — under this
proposal — **tech-discovery**. Both produce a report, and the two are referred to
separately throughout this document, because which one ran determines what else the
report contains.

Criteria are evaluated against the dependencies of the application's **most recent**
report, whichever kind produced it, which is the same scoping the analysis report
endpoints already use. Dependencies from superseded reports are never considered.

A reported dependency carries a provider, a name, a version, and whether it was reached
indirectly. All of it is passthrough from analyzer-lsp, and the hub does not interpret
any of it today. This proposal interprets provider, name, and version, which is the
extent of the coupling it introduces.

**An application that has had neither an analysis nor a tech-discovery has no dependency
data, so no dependency expression is satisfied** and the application does not match —
including for negated expressions, per the guard in
[Matching Semantics](#matching-semantics).

The consequence is a behavior change for an archetype that previously matched on tags
alone and later gains a dependency expression: every member with no dependency data
drops. It is surfaced before save (DAC005) and in
[Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy). Extending tech-discovery to
report dependencies is what keeps it from being a severe limitation in practice; see
[Dependency Data from Tech-Discovery](#dependency-data-from-tech-discovery).

### Dependency Data from Tech-Discovery

Dependency criteria are only useful if dependency data is widely available. Today it is
produced only by a full analysis, which is expensive and is not routinely run across a
whole portfolio. Tech-discovery is — it is the cheap pass that tags applications during
onboarding — but it currently discards dependency information entirely.

Two things in `konveyor/tackle2-addon-analyzer` produce that today: `Analyzer.Run()`
skips dependency resolution during tech-discovery, and `updateApplication()` returns
after tagging without creating a report — so dependencies would not be reported even
if they had been collected.

Note that tech-discovery is not a profile setting; it arrives through the task data.
Varying dependency collection *by profile* would require extending the analysis profile
mode first, and is not proposed.

**Proposed change.** Tech-discovery resolves dependencies and reports them with an empty
insight set. The upload manifest already carries the report, insights, and dependencies
as separate parts, so this populates only the last.

This makes dependency criteria usable on a portfolio that has had tech-discovery run but
has not been analyzed — the common state during onboarding, and precisely when archetype
classification is most valuable. It is what keeps "no dependency data, no match" from
being a severe practical limitation.

The change is confined to the addon; it relies on existing hub behavior, described below.

#### A tech-discovery report replaces the previous one

The hub already treats the newest report per application as the current one, so a
tech-discovery supersedes whatever preceded it, including a full analysis. That is the
intended behavior and requires no change.

As noted in [Motivation](#motivation), tech-discovery is triggered by an edit to the
application, and an edit can change what the application points at. The insights of the
previous analysis then describe a codebase the application no longer refers to, and
continuing to report them asserts findings about the wrong thing. Replacing the report is
the correct outcome — "latest report has dependencies and no insights" accurately
states *we know what it depends on, we have not evaluated rules against this version*.

This also closes an existing gap: today an application can be edited and retain a stale
analysis indefinitely, because nothing invalidates it.

No **addon-mode** discriminator is added to `Analysis`. Marking the producing run on the
report would exist only to defeat the superseding behavior described above, and it would
make "the latest report" mean different things to different consumers.

**One ambiguity this leaves.** After a tech-discovery the UI shows zero insights, which
is indistinguishable from a full analysis of a genuinely clean application. One means
"not evaluated", the other "evaluated, nothing found", and reporting an application as
clean when it was never checked is a meaningful misstatement. Content cannot separate
them — a clean application also has dependencies and zero insights. Provenance is
already available outside the report, in the application's task history, which
distinguishes an `analyzer` run from a `tech-discovery` run. That is where the UI should
resolve it rather than denormalizing the producing run onto the report.

**To confirm during implementation.** Whether dependency resolution returns a complete
transitive closure in source-only mode. Full analysis mode is selected only when
dependencies are requested, and without it the addon is doing source analysis alone. If
the dependency list is incomplete in that mode, tech-discovery must either enable full
mode — spending the time tech-discovery exists to avoid — or mark the reported
dependencies partial. Marking the data is a narrower discriminator than marking the
producing run, so it does not conflict with the position above, but it would make the
direct/transitive distinction unreliable and weaken transitive matching.

### When Associations Are Evaluated

Archetype membership is **not stored**. `MembershipResolver` is constructed per request
and computes membership in memory from the application tag join. Dependency criteria
follow the same model: membership stays derived, so there is no new consistency surface,
no reconciliation, and no stale associations to invalidate when a new report lands or an
archetype is edited. Results change the moment the underlying data changes.

Preserving that property is the main performance constraint. See
[Resolving Membership](#resolving-membership).

## Design Details

### Implementation

Expression parsing and matching are self-contained, depending on neither the API nor
the assessment packages, so they can be exercised in isolation.

The design principle is to own nothing that is already a published specification with a
maintained implementation. Three dependencies and three in-house components:

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
header. It owns the ecosystem's coordinate structure and its normalization rule; segment
splitting and prefix/`*` matching are shared across all of them, so there is one matcher
rather than one per ecosystem.

The plugin is the counterpart to the `semantic` ecosystem key: both are chosen by the
block, and both exist because the rule they encode is published per ecosystem and differs
between them. Its interface is an implementation detail and is not specified here.

**A pattern is not a purl.** `packageurl-go` and `osv-scalibr/purl` have no wildcard
layer, and normalizing a struct whose name field contains `*` is undefined. Normalization
is therefore a plugin method applying purl-spec's per-type rules to segments, not a
round-trip through a purl parser. This also avoids constructing a purl from stored data,
which would require splitting `com.fasterxml.jackson.core.jackson-core` into namespace and
name — a boundary the stored form does not carry.

#### Version comparison

Ordering is delegated to `osv-scalibr/semantic`, which implements each ecosystem's
published rules. Its ecosystem keys are OSV names and are case-sensitive, which is why
the mapping in [Ecosystem blocks](#ecosystem-blocks) is a literal table rather than a
transformation.

This retires what would otherwise be the largest correctness liability in the proposal.
Behavior was verified against v0.5.3 rather than assumed — Maven qualifier ordering
including the `ga`/`final`/`release` aliases, PEP 440 pre/post/epoch handling, and Go
pseudo-versions and `+incompatible`. Three results drive decisions elsewhere in this
document:

- `2.13.5` is **greater** than `2.13`, which is why a line is written `@>=2.13|<2.14`
  and not `@<=2.13`.
- A leading `v` is stripped, so `@>=v4` and `@>=4` are one constraint.
- Go is ordered as plain semver with no Go-specific handling, which gives the correct
  pseudo-version and `+incompatible` results and also produces the v0 trap in
  [Go major versions](#go-major-versions).

**The library does not validate, and the hub must.** It returns no error for a malformed
version, despite documenting that it does. Verified: `banana`, `${project.version}`,
`LATEST`, and the empty string all parse in Maven, npm, Go, PyPI, Debian, and Alpine, and
sort *below* real versions in most ecosystems but *above* them in Debian. An unresolved
Maven property in analyzer output would therefore satisfy any `<X` constraint silently.

The hub validates candidate version strings per ecosystem before comparison and treats a
failure as "this dependency does not satisfy this reference", per
[Matching Semantics](#matching-semantics). This is a thin wrapper, but it cannot be
skipped and no library supplies it.

**Constraint evaluation.** A parsed constraint is an optional lower bound, an optional
upper bound, and a set of exclusions. A version satisfies it when it falls within
whichever bounds are present and equals none of the exclusions, with every comparison
going through the ecosystem's comparator.

### Resolving Membership

Membership is resolved from an in-memory dataset. The distinct
`(application, provider, name, version)` tuples of the latest report are fetched for
the applications in scope, each archetype expression is parsed once, and the expressions
are evaluated against each application's dependency set. Distinctness has to be requested
explicitly: dependencies are keyed by artifact hash as well, so one name and version can
appear more than once in a single report.

Scope is the set of applications whose membership the request actually needs:

| request | scope |
|---|---|
| `GET /applications/{id}` and its sub-resources | one application |
| `GET /applications` | all, narrowed by the request's existing filter |
| `GET /archetypes/{id}/applications` | all |
| `GET /archetypes` with member counts | all |
| `POST /archetypes/membership` (preview) | all |

The application list endpoint already filters on name, platform, and repository; the
dependency fetch is constrained to the same application set, so a filtered list costs
proportionally less.

**Ecosystem blocks partition the candidate set.** A `[maven]` block can only match
dependencies whose provider is `java`, which is known at parse time. Provider is the
leading column of the dependency table's composite index, so the filter is index-supported
and is unaffected by the `case_sensitive_like` problem that limits a name pre-filter. The
existing dependency report already groups by provider and name, so this is a dimension
the schema and its queries treat as primary.

On a polyglot portfolio this is a large constant-factor reduction: a Java-only expression
never parses an npm version. It does not change the asymptotics.

**This does not scale past roughly 1000 applications**, and the endpoint under the most
pressure is the primary application list rather than an occasional report, because
`/applications` is not paginated.

The arithmetic behind that threshold, as estimates rather than measurements. An
enterprise Java application reports on the order of 400 distinct dependencies — roughly
120–250 for a modest WAR, 250–500 for a typical multi-module EAR, and up to 900 for an
old EAR that bundles everything in `/lib`. So the latest-report dependency set is
about 400k rows at 1000 applications and 4M at 10,000.

Distinct `(name, version)` pairs behave very differently: an enterprise draws on a
library universe of perhaps 2,000–5,000 artifact names with a handful of versions each
in play, so distinct pairs are roughly 15k–30k at 1000 applications and only 40k–80k at
10,000. The dedup ratio rises from about 25× to 50–100× as the portfolio grows, because
the library universe saturates while row count does not. Any future optimization should
exploit that: the costly work — version parsing and comparison — scales with distinct
versions in use, not with applications × dependencies.

Three levers exist and none is proposed here, because which one is needed should be
settled by measurement rather than by argument:

- **Paginating `/applications`** would bound the fetch at one page of applications
  automatically. The pagination machinery already exists; this endpoint simply does not
  use it. The cleanest bound, but it changes the behavior of an existing endpoint and
  belongs in its own proposal.
- **A name pre-filter in SQL**, deriving a prefix from each reference's literal leading
  segments. Two non-obvious constraints apply: SQLite will not use an index for `LIKE`
  under the hub's default `case_sensitive_like` setting, so explicit range predicates
  are needed instead; and because `.`, `:`, and `/` are interchangeable separators, a
  prefix is only selective against a *normalized* name column, not the name as stored.
  This composes with the provider partition above, on the same index.
- **Caching the matched set**, invalidated on archetype write and on a new report.

For context on the baseline: `MembershipResolver` already loads the entire
application-tag join on every request that resolves membership, including single
application reads. The whole-portfolio scan is the status quo; this adds a second scan
roughly 20× larger. It is a multiplier on an existing cost, not a new class of behavior.

Benchmarks against current tag-only resolution time are owed before this proposal moves
off `provisional`.

### Test Plan

The test budget follows the ownership split in [Implementation](#implementation). The
version comparator is a maintained library tested against its own specs, so it is
*verified* rather than *tested*; the components with no library and no second
implementation get the weight.

**Comparator conformance.** A table asserting the ordering behavior this proposal relies
on, run against `osv-scalibr/semantic` as a contract test rather than as a test of our
own code. The cases are those in [Implementation](#implementation), plus `2.13.5 > 2.13`,
which the documented `@>=2.9|<2.14` idiom depends on. Its purpose is to fail loudly if a
library upgrade changes ordering, not to validate the specs.

**Version validation.** The gap the library leaves, so this is ours and is asserted
directly: `banana`, `${project.version}`, `LATEST`, `""`, and a Maven property
placeholder must each be rejected by the hub's validator, and a dependency carrying one
must not satisfy any reference, **including a negated one**. Debian's inverted sort order
for garbage is the case that shows why relying on "unparseable sorts low" is not a
substitute for validating.

**Name matching.** The component with no library and no oracle, so it carries the
heaviest table. Segment prefix matching across the separator forms; `com.sun` matching
`com.sun.xml.bind` but not `com.sunflower`; a reference matching a dependency reported at
greater coordinate depth; `@jsonjoy.com/util` (a scope containing a dot, which must not
split into two name segments); `@babel/core@7` (leading `@` is name, inner `@` is
constraint); `com.foo:3d-view`
(digit-leading final segment is a name, not a version); Go major-version path suffixes;
and `*` in non-trailing position.

**Per-ecosystem normalization.** Cases drawn from purl-spec's published per-type rules
rather than from the implementation: PEP 503 equivalence (`apache_airflow` and
`apache.airflow` match `apache-airflow`) and npm and golang case folding. Each asserts
that pattern and stored name are normalized together, and that a rule belonging to one
ecosystem does not leak into another — `-` must remain load-bearing in a Maven artifact
id and a Go path element, and Maven coordinates must stay case-sensitive.

**Expression parsing.** Valid expressions across all four ecosystems; nested and
negated blocks; malformed expressions; and every row of the rejection table, asserting
both rejection and the suggested equivalent. `1.2,2.0` and `group:artifact:1.2` must not
be accepted under any reading, and a reference outside a block must be a parse error
rather than a default-ecosystem guess.

**Constraint semantics.** Per-form cases for each comparator and for `|`; that a bare
version is exact and `@2` does not match `2.7.18`; that `@>=2|<3` does; that a second
interval (`@>=1|<2|>=3|<4`) is rejected with the `||` equivalent; and the prerelease
boundary — `>=2.13` excludes `2.13.0-rc1` and `2.13-SNAPSHOT` but includes `2.13-sp`.

**Go version idioms.** That `@>=v4|<v5` selects a module on `/v4` and not `/v5`; that
`@>=v0|<v1` fails to match a pseudo-version while `@<v1` matches it — asserted
explicitly, because it is the documented trap and a regression here would be silent; and
that `v2.0.0+incompatible` behaves as `v2.0.0`.

**Package URL input.** That a pasted purl resolves to the same candidate set as the
equivalent block-and-reference form; that its type constrains the ecosystem, so
`pkg:npm/foo` does not match a PyPI `foo`; and that qualifiers and subpaths are rejected
with a message rather than ignored.

**Membership integration.** API-level tests using `test/binding`: archetypes with
dependency criteria only, with tags only, and with both; applications with no dependency
data; and applications whose latest report supersedes an earlier matching one.

**No dependency data.** An application that has had neither an analysis nor a
tech-discovery must not match a negated reference (`!foo`) or a disjunction containing
one. This is the case where evaluating
references individually gives the wrong answer, so it is asserted directly rather than
inferred from the tag-only cases.

**Membership preview.** That `POST /archetypes/membership` returns the same set the
archetype would produce if saved with those criteria, and that a malformed expression
returns `400` with an offset.

**Specificity.** Each row of the product-order table, plus the property that matters
most for regression safety: with no archetype using an expression, selection produces
exactly the same result as the current tag-subset algorithm. The dependency-only versus
tag-only case — where both must be assigned — is the one the previous tag-only rule got
wrong, so it needs an assertion of its own.

**Tech-discovery.** That a tech-discovery reports dependencies and an empty insight
set; that a tech-discovery following a full analysis supersedes it entirely, so insight
reporting returns nothing and dependency matching uses the newly reported set; and that
a full analysis following a tech-discovery likewise supersedes it.

### Upgrade / Downgrade Strategy

A migration adds the `archetype.CriteriaDeps` column. No existing data is
modified and no existing archetype changes behavior: an archetype with an empty
expression resolves exactly as it does today.

The analyzer addon change requires no hub schema change and no sequencing: a
tech-discovery reports through the existing manifest upload, and the hub already
supersedes the previous report.

Downgrade drops the column. Archetypes revert to tag-only criteria, which widens their
membership — the dependency gate is removed. This should be called out in release notes
rather than mitigated, since preserving a filter the schema can no longer express is not
possible.

The one upgrade hazard is behavioral rather than structural: **adding a dependency
expression to an existing archetype will drop every application with no dependency
data** (see [Source of Dependency Data](#source-of-dependency-data)). The UI
surfaces the resulting membership delta before the archetype is saved, via
`POST /archetypes/membership` (DAC005).

## Open Questions

1. **How do the consumers of membership handle conflicting archetypes?** *This blocks
   `implementable`.* Because two different expressions are incomparable under the product
   order, applications will belong to more archetypes once expressions are in use.
   Multiple membership is not new — incomparable tag sets already produce it — so this is
   a pre-existing gap in assessment inheritance, inherited review and risk, and target
   profile selection, which this feature makes routine rather than rare. It is listed as
   blocking because shipping dependency criteria without it converts an edge case into
   the normal path. Resolving it is work in the assessment module, not here.

2. **Dash segmentation in npm.** `-` is not a segment separator, so there is no
   wildcarding within an npm package name. Per-ecosystem normalization settles Python:
   PEP 503 declares `-`, `_`, and `.` equivalent within a distribution name, so folding
   them is a published rule rather than a product decision. It does not settle npm, where
   `.` and `-` are distinct characters in distinct package names — `lodash.merge` and
   `lodash-merge` could both exist, and segmenting on `-` conflates them with each other
   and with the separator forms. The choice is between leaving npm names single-segment,
   which costs wildcarding within a name, and accepting the conflation. Java and Go are
   unaffected: `-` is load-bearing in a Maven artifact id and a Go path element, and
   neither ecosystem declares it equivalent to anything.

## Drawbacks

Dependency criteria make archetype membership depend on an analysis or a tech-discovery
having run, coupling two subsystems that are currently independent. An archetype that
looks correct can match nothing simply because the portfolio has neither been analyzed
nor had tech-discovery run against it, and the failure is silent — an empty member list,
not an error.

**Segment prefix matching makes the characteristic failure over-matching rather than
under-matching.** A reference truncated by one segment silently widens, and an archetype
that captures too many applications looks plausible where an empty one does not. The
match count in the preview exposes it after the fact; completing references against the
coordinates the portfolio actually reports would prevent most of it, since the hub
already holds every name every report has produced.

The dependency matching adds potentially significant pressure on application listing scalability.
Archetype membership is calculated when applications are fetched. The implementation will need
to build in-memory dependency maps for pages of applications. The impact on listing large numbers
of applications will be difficult to predict.
Adopting this on a large estate means accepting a latency regression there, or sequencing one of
the levers in[Resolving Membership](#resolving-membership) first.

It adds a second *reference* syntax to the hub, alongside the `tag:category=name` form
used for addon and extension selection. Sharing the boolean layer removes the
implementation cost but not the user-facing one: two syntaxes that look similar and are
not interchangeable.

Delegating version ordering removes the largest correctness liability but creates a
dependency on `osv-scalibr`'s per-ecosystem implementations, including for ecosystems
Konveyor cares about more than `osv-scalibr`'s primary users do. The conformance table in
the test plan is what makes a behavior change on upgrade visible rather than silent.

Finally, the library's silent handling of malformed versions is a liability the hub
inherits by using it. The validation wrapper is small, but the failure it guards against
— a dependency with an unresolved version placeholder satisfying every `<X` constraint —
produces wrong membership with no error anywhere, which is the same class of failure a
hand-rolled comparator would have produced.

## Alternatives

**Structured criteria instead of an expression language.** A list of
`{ecosystem, coordinates, constraint}` objects on the archetype resource, ANDed together,
rather than a parsed DSL. This is the conventional shape for an API of this kind: no
grammar, no parser, no error offsets, no new syntax in the hub, and criteria that clients
generating YAML can construct field by field.

Rejected because the criteria that motivate the feature are not conjunctions. The Spring
Boot cohort needs a disjunction nested inside a conjunction, with a negation alongside it:

```
(javax.servlet:javax.servlet-api || javax.persistence:javax.persistence-api)
&& !jakarta.servlet:jakarta.servlet-api
```

Representing that structurally means encoding an expression tree as data — `allOf`,
`anyOf`, `not` nodes — which is the same language with more punctuation and less
readability. Flattening it to a conjunctive list instead would drop exactly the cases the
feature exists for; negation is the sharpest, since `!jakarta.servlet` has no structured
form that is not a tree node.

The simple case remains simple: an expression of `&&`-joined references *is* a
conjunctive list, and a UI may present it as one without the stored form changing.

**Native version ranges.** Accepting each ecosystem's own range dialect inside its block —
Maven `[1.2,1.3)`, npm `^1.2`/`~1.2`, PEP 440 `>=1.2,<2.0`, Ruby `~> 2.1` — so that the
whole reference, and not only the coordinates, reads as the build file writes it.

Rejected on four grounds.

Each dialect is a published specification one is then obliged to implement faithfully,
and a partial implementation fails silently rather than loudly. For Maven ranges no Go
library of adequate maturity exists.

Maven's *soft* version semantics make its most common form unusable as a predicate: bare
`1.2` means "prefer 1.2, accept anything", so a faithful reading matches every version
and the only usable reading is a redefinition — leaving users with Maven syntax and
non-Maven meaning on the form they write most often.

Two dialects collide with the boolean layer. Maven's `(1.2,1.3)` is ambiguous against
grouping parentheses, and npm ranges contain their own `||`, so `foo@^1.2 || ^2.0` cannot
be resolved into one term or two without a disambiguation rule.

And the benefit is weakest for Java, where it would be most expected: version ranges are
rare in real `pom.xml` files, so most Java users would be learning the native syntax too.

The native half therefore stops at the `@`. Coordinates are what users have in hand and
what they get wrong when forced to translate; ranges are four specifications of uneven
familiarity that collide with the grammar and, in Maven's case, do not mean as a predicate
what they mean in a build file.

**PURL and VERS as the authoring syntax and internal model.** References written as
Package URLs and constraints as `vers:` URIs, stored and matched in that form:

```
pkg:maven/org.springframework/spring-core?vers=vers:maven%2F%3E%3D6.0.0%7C%3C6.1.0
```

This is what an SBOM-oriented tool would do. Both standards are widely emitted — SPDX,
CycloneDX, OSV, GitHub advisories — both carry the ecosystem explicitly, and adopting
them wholesale would make advisory data directly usable as criteria.

Rejected as the model, and adopted in part.

*A purl is an identifier, not a pattern.* purl-spec defines no wildcard layer, so
`com.sun` and `weblogic` cannot be written as purls and `packageurl-go`'s `PackageURL`
cannot carry one. Adopting purl for references means inventing a glob over it — and a
purl pattern is not a purl, so the interop that motivates the choice does not survive the
construct required to make it usable.

*The interop runs one way only.* Dependency data reaches the hub as
`(provider, name, version)`, so matching purl against purl means rendering stored
dependencies as purls, which means deciding where the group ends and the artifact begins
in `com.fasterxml.jackson.core.jackson-core`. That boundary is not recoverable from the
stored form. Reading a pasted purl is lossless; writing one is not.

*Raw purl+vers is not writable by hand.* Percent-encoded query parameters are a machine
interchange format, and the authoring surface is the half of this feature users touch.

What is adopted: VERS comparator syntax for constraints, purl types as block names,
purl-spec's per-type normalization rules, and purl as a paste-in form for a single
reference. What is not: multi-interval VERS containment
([Version constraint](#version-constraint)), and purl as the internal representation.

Two weaker standards in the same space: OSV is an advisory schema rather than an
expression language, and CPE identifies vendor and product rather than build coordinates,
with well-documented matching problems for this purpose.

This is worth reopening if SBOM or advisory ingestion lands, since the hub would then
hold purl-typed data with the type already resolved and the group/artifact objection
would disappear.

**Ecosystem-agnostic references.** No blocks: one neutral coordinate grammar, with
normalization and version ordering selected per candidate from the dependency's recorded
provider at match time. Terser, and it never asks the user for something the hub already
knows.

Rejected because the ecosystem is needed before any candidate is in hand. Native
coordinate forms cannot be parsed without it — `@babel/core` and `group:artifact` are
different shapes — so an agnostic syntax must also be a neutral one, and users translate
rather than paste. Declaring it also buys the candidate partition in
[Resolving Membership](#resolving-membership), which reading the provider per candidate
cannot: the filter is only known once the candidate has been loaded.

**Prefix version semantics.** `@2` meaning the whole 2 line, with range endpoints as line
infima and suprema, so that `2.9..2.13` includes `2.13.5`. Terser for the most common
query, and it carries the prerelease rule with no special case.

Rejected because comparators say what they do. Prefix semantics needs an invented rule
for the line beneath a qualified version such as `2.0-beta9`, and it reads `@2.13` as
matching `2.13.7` by a convention that has to be learned, where `>=` and `<` are
understood on sight. The cost is accepted: "all Spring Boot 2" is `@>=2|<3`, and a user
who writes `@<=2.13` meaning the 2.13 line gets something narrower.

**Explicit wildcards rather than implicit prefix matching.** Names match exactly unless a
wildcard is written — `com.sun.**` to reach a namespace, `com.sun` to match only a
dependency of that exact name.

Rejected because the exact reading is rarely what is wanted: a coordinate reported at
greater depth would not match its own shorter form, and every namespace query would carry
a `**`. The trade is real and is recorded in [Drawbacks](#drawbacks) — implicit prefix
matching makes over-matching the default failure, where explicit wildcards would make
under-matching the default.

**Materialized archetype membership.** Storing membership in an association table
maintained on a new report and on archetype edit would make reads trivially cheap.
Rejected because membership is currently derived, and keeping it derived avoids an entire
class of staleness bug. It remains the escape hatch if the cheaper levers in
[Resolving Membership](#resolving-membership) prove insufficient at scale.

**Matching against all reports rather than the latest.** Rejected: an application that
once depended on a library but no longer does should not be classified by it.

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
  Where either kind of run is meant, the text says "report".
