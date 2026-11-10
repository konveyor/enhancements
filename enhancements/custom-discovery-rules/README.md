---
title: custom-discovery-rules
authors:
  - "@jortel"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-11-03
last-updated: 2026-11-10
status: provisional
see-also:
  - "/enhancements/custom-migration-targets/README.md"
  - "/enhancements/centralized-configuration-management/README.md"
  - "/enhancements/dependency-archetype-criteria/README.md"
---

# Discovery Configuration

Discovery annotates applications with tags, and those tags are what Archetype
criteria match on. It is the product's most consequential automatic behavior and
its least visible one: a user cannot see what discovery consists of, cannot
influence when it runs, and cannot change what it recognizes. This enhancement
makes discovery something a user can see and configure.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

Discovery becomes configurable and visible. The hub holds a discovery
configuration and the UI presents it, delivering three capabilities.

1. **Discovery is made of named discoverers, and the user can see them.** The
   configuration enumerates the discoverers the product defines — today
   `lang-discovery` and `tech-discovery` — and each has a detail page describing
   what it inspects and what it produces. Discovery stops being an unexplained
   thing that happens to applications.

2. **Each discoverer's triggering events are configurable.** Discovery is
   triggered by the hub rather than launched by a user, and today every discoverer
   runs on every triggering event. An architect chooses which events run which
   discoverer — application created, application updated — and a discoverer with
   no enabled events does not run.

3. **Technology discovery's rules are configurable.** An architect selects which
   *rule bundles* technology discovery runs. A bundle is a named, selectable thing
   carrying labels and optionally rules of its own; the product seeds one for the
   shipped discovery rules, and an architect creates others to contribute custom
   rules — for technologies the shipped rules do not cover, and to extend what they
   do cover.

The third capability is specific to `tech-discovery` and constitutes its detail
page. `lang-discovery` runs no rules, so its detail page is description and
triggering events alone.

Rule bundles follow the pattern the product already establishes for custom
migration targets: a named, selectable entity carrying labels and an optional
ruleset, where the rules are authored as YAML and either uploaded or kept in a
source repository the hub fetches, with credentials where the repository needs
them. This enhancement introduces no new way to author or package rules, and no
new concept for a user to learn beyond the one they already meet when they define
a custom target.

These are stated as requirements rather than as a design. The resource that
carries the configuration, how it reaches the analyzer add-on, and how the UI
presents it are design details, sketched in [Design Details](#design-details) and
open to change.

Layout, controls, and wording are UXD's and out of scope here.

## Motivation

Discovery tags an application from two independent sources. Language discovery
inspects the repository directly and reports what it recognizes — `Language=Java`,
`Framework=Spring Boot`, `Tooling=Maven` and the like. Technology discovery runs
rules and tags from whatever they match. Both land on the application as ordinary
tags, and Archetype criteria are defined on top of them.

Neither is visible or adjustable. The hub submits both on every event that
triggers discovery — an application created or updated, and a credential it
depends on changing — and which rules technology discovery runs is decided by
configuration installed with the operator. Four consequences, and they are the
whole motivation.

**Users cannot see what discovery is.** There is no page that names the
discoverers, says what each inspects, or says what tags each can produce. An
application acquires tags from a process the product never explains, and the first
time most users think about discovery is when it produces a tag they did not
expect or fails to produce one they did.

**Users cannot adjust when discovery runs.** Every discoverer runs on every
triggering event, so editing an application's name re-runs technology discovery
across its repository. Correcting an owner, attaching a business service, fixing a
typo — each one spends an analysis. Rotating a default credential spends one for
every application in the portfolio that relies on it. On a large portfolio this is
continuous work in service of a result nobody asked for at that moment, and there
is no way to say so.

**Users cannot extend what discovery recognizes.** Organizations have internal
frameworks, internal variants of frameworks the product already detects, and
conventions worth classifying on. None of it is reachable, so classification for
those applications is manual, and the Archetype automation built on top of
discovery tags does not reach them.

**Users cannot narrow what technology discovery does.** The rule selection is
fixed, so an organization cannot restrict discovery to the rules it cares about,
and cannot turn it off.

A secondary observation: rules are already a first-class thing a user can supply.
Custom migration targets accept uploaded rule files or a repository reference, and
analysis profiles do the same. Discovery is the one place rules matter where the
user cannot reach.

### Goals

- A user can see which discoverers exist, what each inspects, and what each
  produces.
- An architect can configure which triggering events run each discoverer,
  including the credential-change events that fan out across the portfolio.
- An architect can stop a discoverer from running.
- An architect can configure which rules participate in technology discovery, by
  selecting rule bundles.
- An architect can contribute custom rules to technology discovery by creating a
  bundle carrying them, authored as YAML and either uploaded or kept in a source
  repository.
- Creating and selecting a rule bundle works the way defining and selecting a
  custom migration target works. A user who has done one recognizes the other.
- Selecting a custom bundle alongside the seeded one extends the shipped rules
  rather than replacing them; replacing them is available by deselecting the
  seeded bundle.
- Shipped behavior continues untouched for anyone who configures nothing.

### Non-Goals

- Detailed UX design. Page composition, controls, and copy are out of scope.
- **User-defined discoverers.** The set of discoverers is defined by the product.
  A user extends `tech-discovery` by supplying rules to it; they do not register a
  discoverer of their own. Each discoverer is a distinct implementation with its
  own inputs and its own output, not an instance of a generic extension point.
- Authoring rules in the UI. Rules are written as YAML and uploaded, or fetched
  from a repository.
- Editing the content of shipped rules.
- Rule-level configuration of the shipped rulesets. Selection operates on rules in
  the aggregate, by label; see [Alternatives](#alternatives).
- Contributing custom rules to an *analysis* target. Analysis profiles already
  accept uploaded files and a repository; nothing here extends that.
- General management of rulesets as user-owned objects, with create, read, update
  and delete of their own; see [Alternatives](#alternatives).
- A CLI surface.

## Proposal

### Background: what is fixed today

Technology discovery is a task the operator installs, and its definition is where
the current configuration lives:

```yaml
kind: Task
metadata:
  name: tech-discovery
  labels:
    konveyor.io/discovery: "technology"
spec:
  data:
    mode:
      discovery: true
    tagger:
      enabled: true
      source: tech-discovery
    rules:
      labels:
        included: ["discovery"]
```

Six facts about the existing implementation, each of which bears on what is being
asked for.

**Discovery is triggered, not launched.** The hub submits discovery tasks when an
application is created or updated, and when an identity an application depends on
changes — directly associated, or the default for its kind where the application
has no identity of that kind of its own. The identity path re-enters the
application-updated path, so every one of these runs the full set of discoverers.
There is no manual launch, in the UI or elsewhere, and no way to say that one of
these events should not trigger a given discoverer. This is why adjusting when
discovery runs has to be configuration, and why turning it off cannot simply be a
matter of not pressing a button.

**Discovery runs for an application only if it has a repository.** The trigger
returns early otherwise, so an application recorded without a source repository is
never discovered and acquires no tags from either discoverer.

**The discoverers run in order, as a pipeline.** The trigger collects the
discovery task kinds, sorts them by task priority and then by label, and submits
them as a single pipelined task group rather than concurrently.

**The discoverers are already distinct, and already labeled.** The installed task
definition distinguishes the technology-discovery kind by label, so the hub's
trigger can already tell one discoverer from another. What it cannot do is treat
them differently.

**Rules are chosen by label.** The add-on asks the hub for the available rulesets
and keeps those matching the task's label selector; the analyzer then applies the
same selector to the rule files themselves. Everything about which rules run
follows from that selector and from the labels rule authors write. This
enhancement makes the selector configurable by putting it on something a user
selects; it does not change the selector language or how labels work.

**Rules can also be supplied directly.** A custom migration target carries a
ruleset of its own — uploaded files or a repository with credentials — and rules
supplied that way run without being selected, because the add-on marks them
always-included. Analysis profiles carry rules the same way. This is why
contributing custom rules to discovery is a matter of giving discovery something
to carry them rather than inventing anything.

**These two mechanisms are already combined in one entity.** A migration target is
a named, selectable thing with an image, a list of labels, and an optional
ruleset; selecting it contributes both the rules its labels select and the rules
it carries. That combination is precisely what configuring technology discovery
requires, and it is the basis of the [rule bundle](#technology-discovery-rule-bundles)
below.

**Tags come from the rules that run.** They take the form `Category=Value`; the
tagger creates the category and tag if absent and replaces the application's tags
for the `tech-discovery` source — its own, separate from the one language
discovery writes under. From there Archetype matching proceeds normally.

> **Where this behavior lives.** For reviewers wanting to confirm the above: the
> task definition is in `tackle2-operator`
> (`roles/tackle/templates/customresource-task.yml.j2`); rule selection, the
> always-include rewrite, and the tagger are in `tackle2-addon-analyzer`
> (`cmd/rules.go`, `cmd/tagger.go`); the discovery trigger is in `tackle2-hub`
> (`internal/trigger/application.go`), with the credential-change triggers and
> their affected-set resolution in `internal/trigger/identity.go`; targets and
> their seeding — the closest existing analogue for a selectable bundle of labels
> and rules — are in `internal/api/target.go` and `internal/seed/target.go`, and
> analysis profiles, which carry rules the same way, are in
> `internal/api/profile.go`.

### Discoverers

The discovery configuration enumerates discoverers. There are two, both defined by
the product:

| Discoverer | Inspects | Produces |
| --- | --- | --- |
| `lang-discovery` | The application's repository, directly | Language, framework, and tooling tags |
| `tech-discovery` | The application's repository, by running rules | Tags matched by the rules that run |

The set is opinionated: the product decides what a discoverer is, and a user
configures the ones that exist rather than adding to them. The two differ in what
they can be configured *with* — `tech-discovery` is rule-driven and so has rules to
configure, while `lang-discovery` has none — but both are configurable in the ways
that apply to every discoverer: whether they run, on what, and what the user can
see about them.

Naming them is itself part of the requirement. Today "discovery" is one
undifferentiated thing from the user's side, which is why the gap between "my
Java tags are wrong" and "the rule that produces that tag is in this ruleset" is
unbridgeable from inside the product.

### Triggering events

Each discoverer carries the set of events that cause it to run. The events are
the ones the hub already triggers on:

| Event | Scope |
| --- | --- |
| **Application created** | The one application |
| **Application updated** | The one application |
| **Associated identity changed** | Applications the identity is directly associated with |
| **Default identity changed** | Every application with no identity of that kind of its own |

The last two matter because a credential change can alter what discovery is able
to see. A repository that could not be cloned before a credential was fixed
contains, from discovery's point of view, nothing; rediscovering on that change is
the hub being correct rather than the hub being eager.

They also differ sharply in cost. The first three affect a bounded set of
applications. A change to a default identity is a portfolio-wide fan-out — by
design, since a default applies to every application that has not overridden it —
and it is the single most expensive thing that can trigger discovery. An
organization that rotates a default credential on a schedule is re-analyzing its
entire portfolio on that schedule, today, with no way to decline.

An architect enables or disables each event per discoverer. A discoverer with no
enabled events does not run — that is how discovery, or one half of it, is turned
off. Stating it as the absence of triggers rather than as a separate switch keeps
one mechanism where there would otherwise be two, and makes "off" mean something
precise: the hub creates no task for that discoverer.

The granularity is what makes this useful rather than blunt. An organization can
keep discovery on creation, drop it on every subsequent field edit, keep it when a
credential changes because that genuinely changes what is visible, and drop it on
default-identity rotation because the cost is not worth it. Each of those is a
separate judgment and today they are one.

Defaults preserve present behavior: every event enabled for every discoverer, which
is what an installation does today.

### Discoverer detail

Each discoverer has a detail page, and it serves two purposes.

The first is visibility, and it applies to both discoverers equally. The page says
what the discoverer inspects, what kind of tags it produces, which events trigger
it, and when it last ran. For `tech-discovery` it also says which rules are
currently selected to run. A user who wonders why an application carries a tag, or
does not, should be able to start here rather than in the product documentation.

The second is configuration specific to that discoverer. For `tech-discovery` that
is the rules — the two sections that follow. For `lang-discovery` there is none,
because it runs no rules; its detail page is description and events.

### Technology discovery: rule bundles

Both halves of the rules requirement — which shipped rules participate, and how an
architect contributes their own — are served by one entity and one act. The
entity is a **rule bundle**: a named, selectable thing that contributes rules to a
discoverer. The act is selecting it.

> **On the name.** "Rule bundle" names the entity by its traits rather than by the
> resource implementing it, and is not a proposed user-facing term. The product
> already has an entity with these traits; see
> [Candidate shape](#candidate-shape).

A rule bundle has four traits, and the requirement is the traits rather than any
particular model:

- **A name.** It is referred to, selected, and listed by it.
- **An image.** Bundles are chosen from a visual list, and the image is what makes
  that list scannable rather than a column of strings. This is a presentation
  concern that nonetheless belongs in the entity, because selection is the primary
  interaction and the image is what carries it.
- **A list of labels.** These select rules that already exist in the installation
  — which is how the shipped discovery rules are chosen today.
- **An optional reference to a ruleset.** Rules supplied this way are carried by
  the bundle itself, as uploaded YAML files or as a source repository with optional
  credentials. Rules supplied this way always run; they are not subject to label
  selection.

The two traits that carry rules are what let one entity serve both halves. A
bundle with labels and no ruleset selects rules the installation already has. A
bundle with a ruleset and no labels contributes rules the installation did not
have. A bundle can do both. These correspond exactly to the two mechanisms
described in [Background](#background-what-is-fixed-today), which is why no new
rule-delivery mechanism is needed.

#### Discovery rules are ordinary rules

Rule manifests are identical for analysis and discovery: same syntax, same
schema, same validation. A discovery rule is not a distinct kind of artifact. What
makes a rule useful to discovery is what it *does* — it produces a tag, which the
tagger then applies to the application — and not anything about its form.

This is mostly good news, and it is why an architect contributing a discovery rule
has nothing new to learn and why the product needs no second validation path. Two
consequences follow that the design has to accept rather than fix.

The first is that a bundle's purpose cannot be inferred from its contents. Nothing
in a ruleset marks it as a discovery ruleset, so whatever distinguishes a bundle
intended for discovery from one intended for analysis is a property of the bundle,
declared by whoever creates it. It cannot be validated against the rules inside.

The second is that a rule which produces no tag is accepted, runs, and does
nothing. An architect who writes a rule with an analysis action rather than a tag
action gets no error at upload and no error at run; they get an application that
is not tagged. This is a quiet failure, it is reachable by an easy mistake, and
the discoverer's detail page is the natural place to surface it — a bundle that
contributes no tag-producing rules is worth saying out loud.

#### The seeded bundle

The product seeds a bundle carrying the `discovery` label and no ruleset,
selected by default. It reproduces what the installed task definition does today,
so an installation where nobody configures anything runs exactly the rules it runs
now. Being seeded, it is selectable and deselectable but not editable or
deletable — the same treatment shipped migration targets already receive.

#### Selection, and what follows from it

An architect selects the bundles `tech-discovery` runs, on its detail page. Every
configuration the requirement calls for is a selection:

| Configuration | Expressed as |
| --- | --- |
| Shipped rules, as today | The seeded bundle alone — the default |
| Shipped rules plus custom rules | The seeded bundle plus the architect's own |
| Only custom rules | The architect's own, seeded bundle deselected |
| Narrowed shipped rules | A bundle whose labels select the wanted subset |

This is the point of the entity. Extending rather than replacing is not a rule the
system enforces, it is what selecting two bundles means; "run only my rules" needs
no special case, only a deselection; and contributing a rule and having it
participate are one act rather than two, so there is no state in which an
architect has supplied a rule that is not running.

Selection operates on rules in the aggregate. An architect narrows what
participates by selecting a bundle whose labels select less, not by toggling
individual rules; the product has no mechanism for enumerating the rules inside a
ruleset, and introducing one is a larger piece of work than this enhancement. An
organization wanting a specific subset of the shipped rules can carry them in a
bundle of its own.

The configuration is global. Discovery is a property of the installation, not of
an application or an archetype, and nothing in the current trigger or task
definition is parameterized per application.

#### Selecting nothing is not the same as off

Selecting no bundles and disabling a discoverer both result in no tags, and they
are different states:

- **No bundles selected.** `tech-discovery` runs and produces no tags. This is a
  valid configuration rather than an error, and it is not the way to turn discovery
  off.
- **No triggering events enabled.** The hub creates no task. This is the only
  control that means off.

### What this does not change

Rules are authored as YAML by hand, outside the product, as they are today. The
label selector language, the order rules run in, and how a rule produces a tag are
untouched. Shipped rules keep running and are not editable. Language discovery's
behavior is unchanged — what becomes configurable about it is when it runs, not
what it does. Archetype matching proceeds exactly as it does now, from tags it
cannot distinguish by origin.

### User Stories

#### Seeing what discovery does

An application has acquired a tag its owner does not recognize. They open
discovery configuration, see that there are two discoverers, and open
`tech-discovery`. The page says it runs rules against the repository, lists which
rules are currently selected, and shows when it last ran for that application. The
question is answerable inside the product.

#### Stopping discovery from running on every edit

An organization maintains a large portfolio and its architects edit application
records constantly — owners, business services, repositories. Each edit re-runs
technology discovery across the repository. The architect disables the update
event for `tech-discovery`, keeping it on creation, and the churn stops.

#### Surviving a credential rotation

The organization rotates the default source credential quarterly, as policy
requires. Today that re-analyzes every application relying on it, which is most of
the portfolio, and the run takes the better part of a week for a result that is
almost always identical to the one before it. The architect disables the
default-identity event for `tech-discovery` and leaves it enabled for
`lang-discovery`, which is cheap. Rotation stops triggering a portfolio-wide
analysis, and when the architect does want one — after a rotation that genuinely
changed which repositories are reachable — they run the discoverer across the
portfolio deliberately.

#### Registering a rule for an internal framework

An architect has written `acme-platform.yaml`, a rule that detects their in-house
service framework and emits `Internal Framework=Acme Platform`. They create a rule
bundle named "Acme Platform", give it an image so it is recognizable in the list,
upload the file to it, and select it for `tech-discovery` alongside the seeded
bundle.

The next time an application is created, discovery runs, the rule matches, and the
application is tagged. An Archetype with that criteria tag then associates the
matching applications automatically, and they inherit its assessment and target
profiles.

#### Rules maintained in a repository

The organization keeps its discovery rules in a git repository, reviewed and
released like any other code. The architect creates a bundle pointing at the
repository, with credentials, rather than uploading files. Discovery fetches it on
each run, so the rules stay current without anyone re-uploading anything.

#### Replacing the shipped classification

An organization has curated its own discovery rules and finds the shipped
classification noise. The architect selects their own bundle and deselects the
seeded one. Discovery produces their tags and none of the shipped ones. Nothing
about this is a special mode — it is the same selection control used to add a
bundle, used to remove one.

#### Turning technology discovery off

An organization does not use technology discovery and does not want it running
across a large portfolio. The architect disables every triggering event for
`tech-discovery`. Language discovery continues, and analyses are unaffected.

#### Retiring a rule

A framework has been decommissioned and the architect deselects the bundle
carrying its rule. Discovery stops applying the tag. Applications keep the tags
they have until each is rediscovered, because the tagger replaces the full set for
its source rather than removing tags when a rule goes away.

### Reclassifying existing applications

Discovery runs on triggering events and nothing else starts it, so changing the
configuration does not reclassify applications already discovered. This is
existing behavior and applies equally to the shipped rules — an upgrade that
corrects a built-in discovery rule reclassifies nothing.

Two things make that gap untenable once discovery is configurable.

The first is expectation. Where nobody can alter discovery, nobody expects an
alteration to take effect; once an architect can save a rule, they will expect
their portfolio to reflect it, and watching nothing happen is a poor answer.

The second is that configurable triggering events remove the automatic path. An
architect who disables the update event has switched off the mechanism that would
eventually have reclassified their applications — and the organizations most
likely to switch it off are the large portfolios where reclassification matters
most.

The proposal is therefore an explicit action rather than an automatic fan-out: a
discoverer's detail page offers running it across the portfolio, and the architect
takes that action when they have finished configuring.

The precedent cuts against this and should be acknowledged. The hub already
fans out across the portfolio automatically when a default identity changes, so a
configuration change doing the same would be consistent rather than novel, and the
machinery exists. The argument for the explicit action is not that the fan-out is
unaffordable but that its frequency differs: a default credential rotates
occasionally, whereas discovery configuration is edited repeatedly during setup —
selecting a bundle, uploading a rule, correcting it, uploading again. Firing a
portfolio-wide analysis on each save turns an afternoon of configuration into days
of compute. An explicit action separates the editing from the applying, which is
the distinction that matters here and does not arise for credentials.

What remains open is the scope of the action: whole portfolio, or a selection. See
[Open Questions](#open-questions).

## Design Details

This section is deliberately thin. The capabilities above are the requirement; what
follows is the shape the design is expected to take, recorded so reviewers can
judge feasibility, and expected to be replaced by a concrete design before this
enhancement is `implementable`.

### Candidate shape

Two pieces: a singular hub resource holding the discovery configuration — a
collection of discoverers, each with an identifier, its enabled triggering events,
and its selected rule bundles — and whatever resource implements the bundle
itself.

The hub's trigger already distinguishes the discoverers by label, so gating on
events is a filter in a place that already knows which discoverer it is about. The
add-on already fetches rule files, clones a rule repository, resolves credentials,
and marks directly-supplied rules always-included, so a bundle's rules reach the
analyzer by a path that exists. The hub-side work is a resource, a migration, and
a change to the trigger; the UI is a list, a detail page, and a selection control.

**The trigger has to learn why it fired.** The identity triggers do not submit
tasks themselves; they resolve the affected applications and call the
application-updated path for each. From inside that path, a credential rotation
and a user renaming an application are indistinguishable. Gating the four events
separately therefore means carrying the reason into the trigger rather than
filtering on what is already there — a small change, but a change to a signature
on a hot path rather than a new branch in one place, and the one piece of this
that is not already sitting in the code waiting to be configured.

**Target is the existing entity with the bundle's traits.** It is named, carries
an image, carries a list of labels, and carries an optional reference to a
ruleset, which is the whole trait list; its ruleset in turn carries either rule
files or a repository with an identity for credentials. It already distinguishes
seeded from user-created — a target is builtin when it has a UUID, and the API
exposes that to the UI — which is what makes the seeded bundle deselectable but
not deletable without new machinery. The seeder is keyed on UUID and already
handles renames and name collisions, so re-seeding across upgrades is solved. The
custom-target UI is the selection and authoring surface this requirement
describes, built already.

Reusing it is the leading option and not a foregone one. What it would need:

- **A discriminator.** Targets are migration targets today, and discovery bundles
  must not appear in analysis flows nor migration targets in discovery. A field on
  the entity, defaulting to the analysis meaning so existing rows are untouched.
  It has to be declared rather than derived: per
  [Discovery rules are ordinary rules](#discovery-rules-are-ordinary-rules),
  the manifests are identical, so nothing in a bundle's contents reveals what it
  is for. Deriving it from the labels — "it is a discovery bundle because it
  carries the discovery label" — should also be rejected: it makes the entity's
  purpose depend on a field the user edits for another reason, so a label change
  silently removes a bundle from discovery.
- **A decision on eligibility versus selection.** A migration target is chosen per
  analysis by a user; discovery has no such moment, because the hub triggers it.
  So either the discriminator alone activates a bundle, or it marks the bundle
  eligible and the discovery configuration records which eligible bundles are
  selected. The latter costs a reference but is what makes the seeded bundle
  deselectable, and it is what the [selection table](#selection-and-what-follows-from-it)
  above assumes.
- **A sweep of the consumers.** Everything that lists targets today — the analysis
  wizard, target profiles, the UI ordering setting, the bindings — has to filter on
  the discriminator. Missing one surfaces as a discovery bundle appearing as a
  migration target, which is quiet and wrong.
- **A reconciled permission model.** Targets are administered today, while
  discovery configuration belongs with architects per [RBAC](#rbac). One of the
  two has to give.

What it does *not* need is divergent handling of the rules themselves. Rule
manifests are identical for analysis and discovery — same syntax, same schema —
so a bundle's ruleset is the same object a migration target's is, validated on
upload by one path and carried to the analyzer by one path. MTA-8042 lists format
parity as a dependency; it is satisfied. This is the largest single reason the
reuse is cheap.

The argument against is the noun. A target answers "what am I migrating *to*,"
and discovery is not migrating anywhere — it classifies. Reusing it generalizes
Target to "a named, selectable bundle of labels and optional rules, scoped to a
purpose," which is a coherent abstraction but a wider one than the name suggests,
and OCPSTRAT-3880 is already asking whether the target-family naming holds up. If
the generalization is taken, it should be taken deliberately, and the UI need not
call these targets on the discovery page even if the resource is shared.

Other points to settle, none of which change the requirement:

- **How "last ran" is sourced for the detail page.** Discovery runs per
  application, so a discoverer-level view of it is an aggregate the hub does not
  currently keep. The visibility requirement is cheap except for this part.
- **Whether the event set is open.** Four events exist because the hub triggers on
  four things, and they are already heterogeneous — two about an application's own
  lifecycle, two about credentials it depends on. That heterogeneity suggests the
  set will grow; if platform-sourced applications or scheduled rediscovery add
  more, the configuration should carry them without a schema change.
- **Whether a bundle's labels and ruleset are both editable after creation.** The
  custom-target form fixes the rules source at creation; whether that constraint
  should carry over is worth revisiting rather than inheriting.
- **Whether a singular configuration resource stays singular.** The configuration
  is stated as global, but per-archetype discovery is a plausible extension and
  allowing for it now is cheaper than retrofitting it; see
  [Open Questions](#open-questions).

### Relationship to OCPSTRAT-3914

Three of the parent Feature's acceptance criteria do not match what is proposed
here. They are recorded so the divergence is deliberate rather than silent, and
each needs the Feature text amended or a second Feature filed.

The Feature asks that users "create, update, and delete custom discovery rules
through the Hub API and UI." Rules are not user-owned objects here; a user
supplies rule files or a repository to a bundle, as they already do for custom
targets. The reasoning is in [Alternatives](#alternatives).

The Feature asks that built-in rules "can be individually disabled or overridden."
Per-rule control needs a mechanism for enumerating the rules within a ruleset that
the product does not have and that warrants its own epic; the near-term
requirement is scoped to ruleset granularity, which is what this document
specifies.

The Feature describes neither the discoverer model nor configurable triggering
events, both of which are proposed here.

### RBAC

Configuring discovery is an architect's act, and discovery affects every
application in the portfolio. The configuration should be readable by every role
that reads the rest of the portfolio and writable by architect and admin, matching
how analysis profiles are already granted.

Disabling a discoverer is the one piece treated differently. It is
installation-wide and it silently changes what every subsequent application gets
classified as, which puts it closer to an administrative act than to the rest of
the configuration. The design should gate it separately and more narrowly.

The UI needs a write gate equivalent to the one it applies to custom targets:
configuration hidden or read-only without it. The visibility parts of the detail
page are readable by anyone who can read applications, since their purpose is
explaining tags that are already visible.

### Test Plan

**Hub.** The configuration round-trips through the API and survives upgrade.
Defaults reproduce present behavior for an installation that configures nothing.
Permissions are enforced per role, including the separate gate on disabling a
discoverer.

**Triggering events.** Each of the four is gated independently, which means
testing each in isolation rather than in aggregate: with only the update event
disabled for `tech-discovery`, creating an application runs it and editing one
does not; with only the default-identity event disabled, rotating a default
credential runs nothing while associating a credential directly with one
application still runs it. `lang-discovery` is unaffected by any of
`tech-discovery`'s settings, and the reverse. With every event disabled, the hub
creates no task for that discoverer.

**Identity events resolve the right applications.** A directly associated
identity triggers only its own applications. A default identity triggers every
application with no identity of that kind of its own, and skips those that have
one. These are the existing affected-set semantics and the gating must not
perturb them.

**Applications without a repository.** Never discovered, on any event, and not
counted as a failure by the portfolio-wide action.

**Rule bundles.** Each row of the
[selection table](#selection-and-what-follows-from-it) is a test: the seeded
bundle alone reproduces today's tags; seeded plus custom produces both sets;
custom alone produces exactly the custom tags and none of the shipped ones; a
bundle whose labels select a subset produces only that subset's tags. A bundle
carrying uploaded rules and one carrying a repository each produce tags. A bundle
carrying both labels and a ruleset contributes from both.

**The seeded bundle.** It exists after a fresh install and after an upgrade from a
version predating this work, is selected by default, and cannot be edited or
deleted. Deselecting and reselecting it restores the shipped classification.

**Rules that produce no tags.** A bundle carrying a rule with an analysis action
rather than a tag action is accepted on upload and runs without error, and the
application acquires no tag from it. The test asserts the benign outcome — no
failure, no partial tagging — and asserts whatever the detail page is specified to
say about a bundle contributing nothing.

**Selecting nothing is not off.** Asserted directly, because the two are easy to
conflate: with no bundles selected the discoverer runs and produces no tags; with
no triggering events enabled the hub creates no task. Different observable states.

**Visibility.** The detail page reports the discoverer's events, its selected
bundles, and its last run, and reflects a configuration change without one.

**UI.** The configuration can be read, edited, and saved; uploads and repository
references behave as they do on the custom-target form; write affordances are
absent without permission.

**Reclassification.** The explicit action re-runs a discoverer across the portfolio
and applications acquire tags from the new configuration, including when the update
event is disabled.

**End-to-end.** The test proving the premise: supply a single rule file emitting a
novel `Category=Value` tag, create an application whose repository trips the rule,
and assert that discovery tags it, that the category and tag were created, and that
an Archetype with that criteria tag picks the application up. Repeat with the rule
in a source repository to cover the other shape.

### Upgrade / Downgrade Strategy

Nothing changes for an installation that configures nothing: the defaults are the
behavior shipped today — every event enabled for every discoverer, and the rule
selection the operator installs.

The hub, add-on, and UI land together. An older add-on against a newer hub ignores
the rule configuration and runs discovery as it always did — a quiet failure mode
in a separately released component, and one the design should try to make noisy.
Event gating is enforced in the hub's trigger and so is not subject to this.

Downgrade loses the configuration and returns discovery to its installed default,
including re-enabling discoverers an architect had turned off.

## Open Questions

> 1. **The scope of the reclassification action.** Per
>    [Reclassifying existing applications](#reclassifying-existing-applications),
>    the proposal is an explicit action rather than an automatic fan-out. What
>    remains is whether it applies to the whole portfolio or to a selection, and
>    how it behaves against a portfolio large enough that the run takes hours.
>
> 2. **Whether discovery configuration should be scoped per archetype.** Stated as
>    global, which matches the trigger and the task definition. Per-archetype
>    discovery is coherent — an archetype is a classification of applications, so
>    applying different discovery to it is not obviously circular, though it is
>    close enough to deserve thought — and deciding now is cheaper than
>    retrofitting.
>
> 3. **Whether `lang-discovery` has enough of a detail page to justify one.** It
>    has events and a description and no configuration. If it does not, the
>    discoverer model is less symmetric than presented here.
>
> 4. **UX design.** These capabilities are specified functionally and have no
>    approved design. UXD work on custom discovery rules — including import and
>    platform association — is in progress separately (MTA-8042, 2026-10-27), and
>    the two need reconciling before scheduling. The same comment scopes the 8.4
>    commitment to capability and plumbing with UI deferred, which the phasing
>    should respect.

## Drawbacks

**A tag's absence stops being evidence.** In an installation where discovery is
fixed, an application without a `Spring Boot` tag is an application that is not
Spring Boot. Once discovery is configurable, absence has three possible causes —
the technology is not there, the rule that detects it was not selected, or the
discoverer was not running when the application was last discovered — and nothing
on the application distinguishes them. Archetype criteria match on these tags
automatically, so the ambiguity propagates into association without anyone looking
at it.

**Narrowing is quiet, and leaves the portfolio in two populations.** An architect
who deselects the seeded bundle or disables a discoverer gets no error, and the
effect lands only on applications discovered after the change; everything
discovered before it keeps the tags it had. The portfolio then holds applications
classified under two different sets of rules, distinguishable only by when each was
last discovered. Expressing the configuration as selection keeps this to one
control rather than three, but a deselection remains a destructive act that looks
like a checkbox.

**An application can be discovered once, at the worst possible moment, and never
again.** Discovery skips applications with no repository, and a repository is
often unreadable until a credential is attached after the record is created. An
organization that enables only the created event — a reasonable-looking way to cut
cost — discovers each application at the point where its repository is least
likely to be readable, and nothing ever re-runs it. The configuration permits this
and says nothing. This is also what makes the explicit reclassification action
load-bearing rather than a convenience: it is the only remaining way back.

**A rule of the wrong kind fails silently.** Analysis and discovery manifests are
identical, so a rule written with an analysis action rather than a tag action is
accepted on upload, runs without error, and produces nothing. The product cannot
tell an architect they have written the wrong kind of rule, only run it. Rules
remain hand-authored, so the documentation carries weight that a richer authoring
surface would not ask of it.

## Alternatives

**Ruleset management as the delivery vehicle.** Rulesets could be made first-class
user-owned objects with create, read, update and delete of their own, and marked
for discovery by adding a label to the ruleset. It works, and it generalizes past
discovery, but it answers the question at the wrong level. The requirement is to
add rules to discovery; that design asks a user to create a ruleset, understand
labels, take an opt-in governing only whether the ruleset is *considered*, and
separately ensure their rule files carry the label governing whether the rules
actually *run*. A user can reasonably do all of it and get nothing. General
ruleset management may still be worth doing; it is not how this capability should
be reached.

**Per-rule enable and disable.** An architect could toggle individual shipped
rules rather than selecting bundles of them. It requires a mechanism for
enumerating the rules inside a ruleset, which the product does not have and which
is its own body of work; and the same outcome is reachable by carrying the wanted
rules in a bundle of one's own. Deferred rather than rejected: if rule enumeration
is built for another reason, selection could be refined on top of it without
changing anything here.

**A separate switch for "discovery enabled."** Turning a discoverer off could be a
boolean distinct from its triggering events. Rejected because it creates two
mechanisms that can disagree — a discoverer enabled with no events, or disabled
with all of them — and the resulting precedence question is the kind of ambiguity
[Selecting nothing is not the same as off](#selecting-nothing-is-not-the-same-as-off)
exists to remove.

**Rule configuration as plain fields on the discoverer.** Rather than an entity,
`tech-discovery` could carry a label selector, a file list, and a repository
reference directly — the shape an analysis profile has. Rejected because it makes
every capability a separate field with its own semantics, and those semantics then
have to be reconciled: an empty selector with files supplied, an empty selector
without them, and a disabled discoverer become three states a user must hold in
mind rather than one selection. It also gives a user nothing to name, nothing to
recognize in a list, and no way to keep two sets of rules from two repositories.
The entity is what turns four behaviors into one act.

**Shipping the configuration in the operator.** The task definition is already
configuration, and exposing it through the operator's CR would cost very little.
Rejected because it puts an architect's routine act behind a cluster-admin edit,
offers no way to supply rule files, delivers nothing for visibility, and is
invisible from the product.

**Making discovery part of analysis profiles.** Discovery is rule-driven, and
analysis profiles already carry rules. Rejected because an analysis profile is
selected per analysis by a user, while discovery is triggered by the hub for every
application; the two have no common selection point, and overloading the profile
would mean an application's discovery behavior depended on which profile someone
last analyzed with.

## Implementation History

- 2026-11-03: Initial draft, proposing user-managed rulesets with a discovery
  opt-in.
- 2026-11-05: Reframed around the two capabilities being asked for — configuring
  which rules participate in technology discovery, and contributing custom rules to
  it — with the mechanism reduced to a candidate shape.
- 2026-11-10: Widened from technology-discovery rules to discovery configuration,
  following team discussion: discovery is modeled as a set of product-defined
  discoverers, each with configurable triggering events and a detail page, with the
  rule configuration becoming `tech-discovery`'s share of it. Resolved
  reclassification to an explicit action and recorded the divergence from
  OCPSTRAT-3914.
- 2026-11-10: Recast the rule configuration around a selectable *rule bundle* —
  named, with an image, a list of labels, and an optional ruleset reference —
  seeded for the shipped rules and created by an architect for custom ones.
  Selecting bundles replaces the separate selector, custom-rules, and precedence
  mechanisms with one act. Target is recorded as the existing entity with these
  traits, in [Candidate shape](#candidate-shape), without the requirement
  depending on reusing it.
- 2026-11-10: Recorded that analysis and discovery rule manifests are identical,
  confirming the parity MTA-8042 lists as a dependency, and drew out the two
  consequences: a bundle's purpose must be declared rather than inferred, and a
  rule producing no tag fails quietly.
- 2026-11-10: Completed the triggering event set with the two credential-change
  events the hub already fires on — a directly associated identity, and a default
  identity — and recorded their differing blast radius, the repository
  precondition, and the pipelined ordering of discoverers.
