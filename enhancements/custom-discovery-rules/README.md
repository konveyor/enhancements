---
title: custom-discovery-rules
authors:
  - "@jortel"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-11-03
last-updated: 2026-11-03
status: provisional
see-also:
  - "/enhancements/custom-migration-targets/README.md"
  - "/enhancements/dependency-archetype-criteria/README.md"
---

# Custom Discovery Rules

Technology discovery annotates applications with tags, and those tags are what
Archetype criteria match on. The rules that produce those tags ship with the product
and cannot be extended. An organization whose portfolio is built on an internal
framework, a proprietary platform, or any technology Konveyor does not know about
has no way to make discovery recognize it, and therefore no way to make Archetype
association work for that part of the estate.

The mechanism to fix this nearly exists. Rulesets are a first-class hub resource with
full CRUD, and the discovery task selects what it runs by label across *every*
ruleset in the hub, including ones a user created. What is missing is a way to create
one, and a way to say what it is for. This enhancement adds ruleset management to the
UI, and closes the gaps that currently make a ruleset useful only when it is attached
to a migration target.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

A ruleset becomes something a user can manage on its own. The UI gains general
create, read, update, and delete for the rulesets a user owns, and the form carries
an opt-in marking a ruleset for technology discovery. Checking it is the whole act of
registering custom discovery rules: the next discovery run picks the ruleset up,
emits its tags, and those tags flow into Archetype matching exactly as the built-in
discovery tags do.

Making that opt-in mean something requires the hub to model a ruleset's labels, which
today it does not. Rules carry labels; the ruleset that contains them does not,
even though the rule format it represents has labels at both levels and lets rules
inherit from the ruleset. Closing that gap is the substance of this enhancement.
It makes the opt-in declarative rather than a property inferred from file contents,
and it makes rulesets that draw their rules from a source repository work at all —
today they can never be selected, because a ruleset with no rules recorded in the hub
has no labels to match.

Discovery is the motivating use case but not the only one. A ruleset participates in
whatever selects its labels, so one labeled for a migration target augments that
target. That is not a quirk of this feature; it is what a target is. The UI is
therefore general ruleset management with a discovery opt-in, rather than a page that
pretends discovery rulesets are their own kind of thing.

This enhancement describes the use cases and the capability each page must provide.
It stops short of specifying layout, controls, and wording — that is UXD's to design,
and is tracked separately on the parent epic.

## Motivation

The tags produced by technology discovery are the raw material for the Archetype
governance model. `Language=Java`, `Framework=Spring Boot`, `Tooling=Maven` and the
rest are discovered automatically on every application, and an Enterprise Architect
defines Archetypes on top of them. That machinery works well right up to the point
where the portfolio contains something Konveyor has never heard of — and in a large
enterprise estate, a great deal of it is exactly that: internal frameworks, forked
platforms, in-house service meshes, corporate build tooling.

For those applications the architect has three bad options: tag them by hand and
re-tag them as the estate drifts, define Archetypes on criteria that do not really
describe the application, or leave them unclassified. Each one undermines the
premise that Archetypes are derived rather than maintained.

There is a second motivation, smaller but related. A ruleset is a real resource with
real capabilities — rules from uploaded files or from a source repository, a
dependency graph between rulesets — and none of it is reachable. The only way to
create one is through the REST API, or indirectly by creating a custom migration
target, which produces a ruleset as a side effect and hides it. Rules are invisible
unless they belong to a target, and the repository option does not work at all
outside one. Giving rulesets a home in the UI is worth doing for its own sake, and it
is what makes the discovery opt-in a one-line addition rather than a bespoke feature.

### Goals

- A user can create, view, update, and delete rulesets they own, and manage the rule
  files in them.
- Marking a ruleset for technology discovery is an explicit, declarative act, and is
  sufficient on its own — no particular labeling inside the rule files is required.
- Rulesets that draw their rules from a source repository work, including for
  discovery.
- A user can see what each ruleset participates in, derived from its labels, without
  having to reason about the selector.
- Rulesets shipped with the product, and rulesets owned by a migration target, are
  not managed here and cannot be damaged from here.
- The hub's model of a ruleset matches the rule format it represents, so that labels
  mean the same thing in both.

### Non-Goals

- Detailed UX design. Page composition, controls, and copy are out of scope; this
  enhancement states what each page must let the user do and what data it has to
  work with.
- Authoring rules in the UI. Rules are written as YAML and uploaded, or fetched from
  a repository, as they are for custom migration targets.
- Managing built-in rulesets or the rulesets owned by migration targets. Neither
  appears in these pages.
- Enabling or disabling built-in discovery rulesets.
- Reclassifying applications that already exist. This is a pre-existing gap that
  custom rules make more visible; see
  [Reclassifying existing applications](#reclassifying-existing-applications).
- Changing the label selector language, the order in which rules run, or how results
  are reported. The add-on changes are confined to how a ruleset's labels are
  resolved and when its rules are marked always-included.
- A general label editor on the ruleset. The discovery opt-in sets one label; see
  [Open Questions](#open-questions).
- A CLI surface. The hub's client bindings already expose ruleset CRUD for any
  client that wants it.

## Proposal

### How discovery rules are selected today

This section records the existing behavior the design depends on, because the design
is mostly a consequence of it.

The operator defines a `tech-discovery` task, and that definition is where the term
`discovery` is given its meaning:

```yaml
kind: Task
metadata:
  name: tech-discovery
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

The task runs the analyzer add-on in discovery mode. To resolve the included labels,
the add-on asks the hub for *every* ruleset and keeps those that match. A ruleset's
labels, for the purpose of that match, are the union of the labels on its rules — so
today a ruleset is a candidate precisely when one of its rules is labeled
`discovery`.

The matched rulesets' rule files are then downloaded, and the same label is applied a
second time, as a rule-level selector evaluated by the analyzer against the content
of those files. Rules that survive both passes emit tags of the form
`Category=Value`. The add-on's tagger splits each on the `=`, creates the tag
category and tag if they do not exist, and replaces the application's tags for the
`tech-discovery` source. From there the tags are indistinguishable from built-in
discovery tags, and Archetype matching proceeds normally.

The consequence: **a ruleset whose rules are labeled `discovery` is already run by
the discovery task, however it was created.** No registration, no profile, no
association with an Archetype or an Application.

Two things stop that from being a usable feature. There is no UI to create a ruleset.
And the only way to say "this ruleset is for discovery" is to put the label inside
every rule file, which couples a property of the ruleset to the contents of files
the user authors elsewhere — and which cannot be done at all when the rules live in a
repository the hub never reads.

> **Where this behavior lives.** For reviewers wanting to confirm the claims in this
> document: the task definition is in `tackle2-operator`
> (`roles/tackle/templates/customresource-task.yml.j2`); ruleset selection, the
> rule-level selector, the always-included rewrite, and the tagger are in
> `tackle2-addon-analyzer` (`cmd/rules.go`, `cmd/tagger.go`); the ruleset resource,
> the label filter, the built-in guards, the target cascade, and the discovery
> trigger are in `tackle2-hub` (`internal/api/ruleset.go`, `internal/api/target.go`,
> `internal/trigger/application.go`); and the existing custom-rule upload is in
> `tackle2-ui` (`client/src/app/pages/migration-targets/`).

### Targets are label sets

This is existing behavior, stated here because the design depends on it and because
it is not obvious from the UI.

A migration target is a named set of labels, plus the presentation that makes it
selectable — an image, a description, a provider. When a profile selects a target,
the target contributes nothing but its labels to the analysis. Its own ruleset gets
no special treatment: it is pulled in afterwards by the same label match that pulls
in every other ruleset carrying those labels.

So there is no distinction in the system between "the target's rules" and "anyone
else's rules carrying the target's label." A ruleset labeled for a target augments
that target, for built-in targets as much as custom ones, and has done since targets
existed.

That has a consequence for this enhancement, pulled out in
[What a ruleset participates in](#what-a-ruleset-participates-in): a user's ruleset
may take part in analyses as well as discovery, depending on how its rules are
labeled. It is a capability rather than a leak, but it is invisible today, and these
pages are the first place it becomes reachable without the API.

### User Stories

#### Registering a rule for an internal framework

An architect has written `acme-platform.yaml`, a rule that detects their in-house
service framework and emits `Framework=Acme Platform`. They create a ruleset named
"Acme internal platform", upload the file, mark the ruleset for technology discovery,
and save.

The next time an application is created or edited, discovery runs, the rule matches,
and the application is tagged `Framework=Acme Platform`. Nothing about the rule file
had to be written in a particular way for this to work.

#### Driving an Archetype from a custom tag

Having registered the rule above, the architect creates an Archetype whose criteria
tag is `Framework=Acme Platform`. Applications discovered to be on the internal
platform are associated with it automatically, and inherit its assessment and target
profiles.

#### Rules maintained in a repository

The organization keeps its discovery rules in a git repository, reviewed and released
like any other code. The architect creates a ruleset pointing at that repository
rather than uploading files, marks it for discovery, and saves. Discovery clones the
repository on each run, so the rules stay current without anyone re-uploading
anything.

Because the repository is shared and may hold rules for other purposes, only the
rules in it labeled `discovery` take part. The architect labels them accordingly in
the repository, where the rest of their rule authoring already happens.

#### Extending a built-in target without forking it

An architect wants their own checks to run whenever anyone analyzes for EAP 8. They
create a ruleset whose rules carry the EAP 8 target label. They do not mark it for
discovery and they do not create a custom target. From then on it participates in any
analysis selecting that target, alongside the shipped rules, and the ruleset's detail
tells them so.

#### Correcting a rule

The rule produced false positives. The architect opens the ruleset, replaces the rule
file with a corrected version, and saves. The old rule and its file are replaced.
Subsequent discovery runs use the new rule.

#### Retiring a rule

A framework has been decommissioned. The architect deletes the ruleset. Its rules go
with it and discovery no longer applies the tag. Tags already applied to applications
are not removed by the delete; they are replaced on the next discovery run of each
application, because the tagger replaces the full set for the `tech-discovery`
source.

### Labels on the ruleset

The rule format has labels at two levels. A ruleset manifest carries labels, each
rule carries labels, and a rule's effective label set is the union of its own and its
ruleset's. The hub models only the rule half. Its ruleset is therefore strictly less
expressive than the artifact it represents, and that single omission is the cause of
most of what does not work today.

This enhancement adds labels to the ruleset, and defines a ruleset's labels — for
every purpose, including the add-on's selection — as the union of its own and those
of its rules.

Three things follow.

**The discovery opt-in becomes declarative.** Marking a ruleset for discovery sets a
label on the ruleset. It no longer depends on what is written inside the rule files,
so it cannot be accidentally undone by editing one, and it can be read back from the
ruleset without fetching anything.

**Repository-backed rulesets become possible.** A ruleset whose rules come from a
repository has no rules recorded in the hub and therefore, today, no labels — so the
add-on's selection can never match it. It is created successfully, looks correct, and
never runs. With labels of its own, it is selectable like any other.

**Nothing existing changes.** Rulesets that exist today have no ruleset labels, so
the union is exactly what it was. No seeded ruleset, no target's ruleset, and no
profile changes what it matches.

### What runs, and where it came from

Selecting a ruleset is only half of it. Once its rules are on disk, the analyzer
applies the label selector a second time, per rule. The two passes have to agree, or
a user gets a ruleset that is selected, downloaded, and then matches nothing — a
silent failure with nothing to diagnose it by.

The add-on already has a mechanism for making the second pass defer to the first: for
rule directories it obtains from a repository, it rewrites the ruleset manifest to
mark the rules as always-included, so they run whatever the selector says. Its
purpose is to honor rules a user supplied explicitly. But it is applied by origin —
every repository directory gets it — rather than by how the ruleset was chosen, and
that is the wrong distinction. Two changes make the two passes agree:

**A ruleset's labels are written into the manifest the add-on generates.** For a
ruleset whose rules are uploaded files, the add-on currently synthesizes a manifest
containing only a name. Writing the ruleset's labels into it means the rules inherit
them, exactly as the rule format specifies. A ruleset marked for discovery therefore
runs its rules during discovery, with no labeling required inside the files.

**The always-included rewrite applies only to rulesets with no labels of their own.**
A ruleset selected *by label* should have that same label selection keep applying to
its contents. A ruleset chosen as a unit — referenced explicitly by a profile, or
pulled in as a dependency of another — was not chosen by a predicate, so all of it
runs, as today. Labels on the ruleset are precisely the signal that distinguishes the
two, which the add-on has not had until now.

This leaves a deliberate asymmetry, which needs documenting because the same opt-in
produces different breadth depending on where the rules come from:

- **Uploaded files.** The user curated this set in this form, so all of it
  participates. The ruleset's labels are inherited by every rule in it.
- **A repository.** The repository is shared and may hold rules for other purposes,
  so its own labeling is respected and only the rules labeled `discovery` take part.
  Its manifest is no longer overwritten.

The rule holds for analysis runs too, which is some evidence it is the right rule
rather than a special case: a labeled ruleset selected because one of its rules
matches a target label is not always-included, so rule-level selection still applies
inside it and only the matching rules run.

### What a ruleset participates in

A ruleset takes part in whatever selects its labels. Marked for discovery, it runs
during discovery. Carrying a target's label, it runs in analyses selecting that
target — see [Targets are label sets](#targets-are-label-sets). Both at once is
possible and legitimate.

Deliberately, this is a capability: augmenting a shipped target today means creating
a custom target, which then appears as a second tile the user has to remember to
select alongside the real one. Labeling a ruleset is better in every respect.

Accidentally, it is a trap. Rule authors start from examples. Someone copies a rule
out of the shipped rulesets as a skeleton, does not strip its target label, marks the
ruleset for discovery, and their half-finished rule is now running in every analysis
for that target in the organization — with nothing in the analysis wizard mentioning
it and nothing in the results pointing back at it.

The mitigation is to make it visible, and the data is already in hand. Every page
that shows a ruleset shows what it participates in, derived from the union of its
labels: technology discovery for the discovery label, and the named target for a
target label. That turns action at a distance into a column. A warning when a ruleset
marked for discovery also carries target labels is a reasonable second layer and a UX
question.

Note that this runs one way only. A ruleset labeled solely for discovery is never
pulled into an analysis, because no analysis includes `discovery` in its selector.
The risk is target labels leaking in, not discovery labels leaking out.

### What the pages manage

The pages manage rulesets the user owns. Two kinds are excluded.

**Rulesets shipped with the product.** A seeded ruleset carries a UUID, and the hub
treats the presence of that UUID as the definition of built-in; it already refuses
updates and deletes on them. What it does not do is tell a client which is which. The
target resource exposes exactly this distinction and the ruleset resource does not,
so the UI currently cannot separate the two. Closing that gap is one of the
[hub changes](#hub-changes). Built-in rulesets continue to run; they are simply not
listed.

**Rulesets owned by a migration target.** A custom target owns a ruleset. The hub
creates it as part of creating the target, gives it a generated name, and records the
reference from the target; deleting the target removes both. These are an
implementation detail of custom targets and belong on that page, so they are not
listed here either — which requires the hub to say which ruleset has an owner, the
second of the [hub changes](#hub-changes).

That reference also closes a hole. Deleting a ruleset directly checks only whether it
is built-in, and nothing at the storage layer prevents the delete, so the target is
left referring to a ruleset that no longer exists — and a target in that state
reports no ruleset at all, which is what the custom-target form reads when it opens
for edit. This has not mattered because nothing in the UI deletes a ruleset.
Introducing a delete action makes it reachable, so the guard comes with it.

### Reclassifying existing applications

Discovery is triggered by creating or updating an application. Nothing else starts
it. So an application discovered last month carries the tags the rules produced last
month, and a change to the rules has no effect on it until someone happens to edit
it.

This is not introduced by custom rules. It is equally true of the rules that ship
with the product: an upgrade that adds or corrects a built-in discovery ruleset does
not reclassify anything already in the portfolio. The gap is simply far more visible
once a user can author a rule themselves, press save, and watch nothing happen.

A hub-side trigger is the natural fix, and the hub already does this shape of
fan-out elsewhere: changing a credential causes the applications affected by it to be
re-triggered, which is how their discovery gets re-run. The equivalent for rulesets
would re-trigger applications when a ruleset that participates in discovery is
created, updated, or deleted, and would do the same from seeding so that rulesets
changing on upgrade are covered too.

Two things make it more than a small change, which is why it is not specified here:

- **Cost.** Re-triggering an application submits a task group for it, and the
  fan-out is the entire portfolio. On a large estate a single rule edit becomes
  thousands of analyzer runs. It needs at least batching and probably debouncing, and
  on upgrade it would fire for every seeded ruleset at once.
- **Consent.** Re-running discovery replaces an application's discovery tags
  wholesale. Doing that to the whole portfolio as a side effect of saving a form is a
  decision a user should make, not one the hub should make for them.

Filed as a follow-up Story under the parent epic. Nothing designed here depends on it
or precludes it; if the trigger lands, the pages need no change.

## Design Details

### Hub changes

**Labels on the ruleset.** A ruleset gains a set of labels of its own, stored
alongside it and settable through the API. This is the one change that touches stored
data, so it carries a migration; existing rulesets have none, which preserves every
current behavior.

**Whether a ruleset is user-created.** The hub already knows this and already acts on
it; it simply does not report it. The target resource exposes the distinction under
the name `custom`, and the ruleset resource should expose it identically — same name,
same meaning, derived rather than stored, ignored on write.

**The owning target, if there is one.** An optional reference naming the target that
owns the ruleset, absent on a free-standing one. This reverses a relationship the hub
already records in the other direction, so it is a presentation change rather than a
data model change. It lets any client distinguish an independent ruleset from one
that belongs to a target.

**Deleting an owned ruleset is refused.** When a target references the ruleset, the
delete fails and the error names that target. Deleting the *target* must continue to
remove its ruleset: that cascade is the supported way to get rid of one, and the new
rule must not catch it. Refusing it the same way the hub refuses to delete a built-in
— same class of error, same status — keeps the two consistent. A dedicated conflict
status would describe the situation more precisely, but the hub has no vocabulary for
that today and introducing one for a single case is not worth it.

**The ruleset label filter matches the union.** Filtering rulesets by label currently
considers only the labels on their rules. It has to consider the ruleset's own labels
as well, or it will disagree with the add-on about what a ruleset's labels are, and
will miss exactly the repository-backed rulesets this enhancement enables.

Everything else the UI needs already exists: ruleset create, update, and delete; file
upload and retrieval for rule content; and ruleset write permission on the architect
role.

### Add-on changes

**Resolve a ruleset's labels as the union** of its own and those of its rules, when
deciding which rulesets to fetch. Rulesets with no labels of their own behave exactly
as they do today.

**Write the ruleset's labels into the generated manifest.** Where the add-on
synthesizes a manifest for a ruleset whose rules are uploaded files, that manifest
carries the ruleset's labels, so its rules inherit them. This is what makes the
discovery opt-in sufficient on its own.

**Mark rules always-included only for rulesets with no labels of their own.** A
ruleset matched by label keeps label selection applying to its contents; a ruleset
chosen as a unit does not. This is what stops a repository-backed discovery ruleset
from running every rule in the repository and from having its own manifest labels
overwritten.

These are behavioral changes in a separately released component, so an older add-on
against a newer hub will ignore ruleset labels and silently fail to select exactly
the repository-backed rulesets this is meant to enable. Worth stating in release
notes; the symptom is a ruleset that looks correct and never runs.

### Pages

Ruleset management, with a discovery opt-in. How it is composed — a page with a
drawer, a modal, a wizard — is UXD's call; what follows is what it has to do and the
data it has to work with. The closest existing analogue is *Custom migration
targets*, which already does most of this for a different purpose and should be the
reuse baseline. It belongs next to that page, in the Developer perspective's
configuration group: ruleset and target write permissions are granted to the same
roles, so the two sit at the same level of the product.

#### Listing rulesets

Lists the rulesets the user owns — excluding built-in and target-owned, per
[What the pages manage](#what-the-pages-manage).

Available per ruleset without any further request: name, description, its labels,
the rules it contains with their labels, whether it draws from a repository, and who
created or last updated it and when. A rule count, the union of labels, and therefore
what the ruleset participates in are all derivable from that. The tags a ruleset
emits are not, because they live in file content.

Must offer: create, edit, delete, and a way to inspect a ruleset without editing it.
Each ruleset shows what it participates in, per
[What a ruleset participates in](#what-a-ruleset-participates-in). Filtering to just
the discovery rulesets should be a single toggle rather than a label query — the
mirror of the opt-in on the form.

Delete should make clear that applications keep their existing tags until discovery
next runs for them, which — per
[Reclassifying existing applications](#reclassifying-existing-applications) — may be
a while.

The empty state carries real weight. A user arriving at an empty list needs to learn
what a ruleset is for, that marking one for discovery is how custom classification
happens, and where the rule-authoring documentation is. This is the only place that
explanation can reasonably live.

#### Creating and editing a ruleset

Needs a name, an optional description, a source of rules, and the discovery opt-in.

The source of rules is either uploaded files or a repository reference with optional
credentials — the same choice the custom-target form offers, and supporting both here
is what makes repository-backed rulesets reachable at all.

The discovery opt-in sets the discovery label on the ruleset. It is the only labeling
the user has to do for uploaded files. For a repository the documentation has to say
that the rules in it still need labeling in the repository, because its own labels
are respected rather than overridden; the form should say so at the point of choosing
the repository option rather than leaving it to the docs.

Name is required and must be unique across all rulesets, not only the listed ones,
because ruleset names are globally unique and targets also create rulesets.
Validating against the full set produces an accurate error rather than an opaque
conflict from the hub; see [Open Question 3](#open-questions) on making the message
comprehensible.

Uploads reuse the rule-file upload the custom-target form already uses. The existing
file-type and YAML syntax checks are sufficient; no label validation is required,
because the opt-in is what makes the rules participate. Per-file feedback should
report the rules parsed and the tags they emit — the form holds the contents at that
moment, and showing the user the tags they are about to introduce is the point at
which a typo is cheapest to catch.

On save, one rule is recorded per file, carrying the file's name, the labels parsed
from it, and a reference to the uploaded file. Rule labels are derived from the file,
never typed: letting someone enter a label that is not in the file would recreate
exactly the divergence between recorded metadata and file content that labels on the
ruleset exist to avoid. On edit, the hub removes any rule absent from the submitted
ruleset, so files that were not touched still have to be submitted.

#### Inspecting a ruleset

Shows the ruleset's metadata, what it participates in, and its rules — for uploaded
files, the rules in each file with their labels and the tags they emit; for a
repository, the reference and credentials, since the hub holds no rules for it.

Tags require fetching file content and re-parsing, so this should happen on demand
rather than for every ruleset in a list. Offering the file for download closes the
loop for a user auditing what is actually registered.

### RBAC

No new permissions. Reading rulesets is already granted to every role that can read
the rest of the portfolio; writing them is granted to architect and admin. The UI
needs a write gate for rulesets equivalent to the one it already applies to targets:
create, edit, and delete affordances hidden without it, the list still readable.

That the architect role holds write permission matches the persona this epic serves,
so nothing needs to change to accommodate it.

One thing to note: ruleset write permission is a single permission covering both
these rulesets and the ones behind custom migration targets. Granting authorship of
one grants authorship of the other. The existing roles already pair the two, so this
introduces no new privilege, but they cannot be separated without new permissions in
the hub.

### Test Plan

**Hub.** Ruleset labels round-trip through create and update, and the migration
leaves existing rulesets with none. The two derived properties: user-created reported
correctly for a seeded and a user-created ruleset; the owning target present on an
owned ruleset and absent on a free-standing one. The label filter matches a ruleset
by its own labels, by its rules' labels, and by both, and a repository-backed ruleset
with labels is returned.

The delete guard has four cases, one of which is the regression risk:

- Deleting a target-owned ruleset is refused, and the target still resolves its
  ruleset afterward.
- Deleting the target still removes the ruleset — the case the guard must not break.
- Deleting a free-standing user-created ruleset succeeds.
- Deleting a built-in is still refused.

**Add-on.** Selection: a ruleset matched by its own labels, by its rules' labels, and
a ruleset with neither not matched. Manifest generation: a labeled ruleset's uploaded
rules run under a selector matching only the ruleset's label, with nothing in the
files carrying it — the case that makes the opt-in sufficient. Always-included
gating: an unlabeled ruleset's repository rules all run; a labeled ruleset's
repository rules are filtered by the selector and its manifest labels survive. A
regression case for each: a profile referencing a ruleset explicitly, and a ruleset
pulled in as a dependency, both still run in full.

**UI.** Rule-file parsing: tags extracted from single- and multi-document files, and
from rules with no tag action. Scoping: built-in and target-owned rulesets excluded
from the list. Participation: the derived display names technology discovery for a
discovery label and the target for a target label, and both when both are present.

Behavioral: the opt-in sets the label and the list toggle filters on it; a duplicate
name is caught in the form; write affordances are absent without ruleset write
permission; a repository-backed ruleset can be created and edited.

**End-to-end.** The test that matters is the one that proves the premise: create a
ruleset from an uploaded file emitting a novel `Category=Value` tag, with no
`discovery` label anywhere in the file, mark it for discovery, create an application
whose repository trips the rule, and assert that discovery tags it, that the tag
category and tag were created, and that an Archetype with that criteria tag picks the
application up. Repeat with the rules in a source repository rather than uploaded,
where the rules *are* labeled, to cover the other half of the asymmetry.

### Upgrade / Downgrade Strategy

The ruleset labels column is additive and empty for every existing row, so on upgrade
nothing changes what it matches: seeded rulesets, target rulesets, and profiles all
behave as before. The pages appear and are empty until a user creates something.

A ruleset created through the REST API before this enhancement, with rules labeled
`discovery`, appears in the list and is editable. That is correct — it is a
user-owned ruleset — and the union means it keeps participating in discovery whether
or not the user also sets the ruleset label.

Downgrade needs the add-on and hub to move together. An older add-on ignores ruleset
labels, so a ruleset that relied on the opt-in alone — no labels inside its files —
stops being selected, and a repository-backed one stops working entirely. Rulesets
whose rules carry the label in file content are unaffected. The derived properties
and the delete guard are safe to lose: a hub without them simply does not report
them, and losing the guard restores pre-existing behavior rather than introducing
anything new.

## Open Questions

> 1. **Reclassifying existing applications.** Discussed in
>    [its own section](#reclassifying-existing-applications) — the gap predates this
>    feature and applies equally to seeded rules, and the candidate fix is a
>    ruleset-change trigger in the hub following the credential-change precedent.
>    What is open is whether that trigger lands in the same release as these pages,
>    and whether it fires automatically or is surfaced as an explicit user action.
>    The fan-out cost on a large portfolio is the deciding factor and needs sizing.
>
> 2. **UX design.** The pages described here are specified functionally and have no
>    approved design. Per the comment on the parent epic (MTA-8042, 2026-10-27), UXD
>    work on custom discovery rules — including import and platform association — was
>    still in progress at the time of writing. This enhancement should be read as the
>    engineering half of that pair; the two need to be reconciled before the work is
>    scheduled.
>
> 3. **Name collisions with custom migration targets.** Ruleset names are globally
>    unique, and a target-owned ruleset does not keep its generated name for life.
>    It is created as `__Target(<name>)-<uuid>`, which cannot collide — but updating
>    the target passes the submitted ruleset through unchanged, and the custom-target
>    form submits the target's own name. So the ruleset is renamed to the target's
>    plain name the first time the target is edited, and from then on a user ruleset
>    of that name is blocked, and vice versa — by a ruleset the user cannot see.
>    Validating against all rulesets makes the error accurate; making it
>    comprehensible is a UX question. Whether the update should preserve the
>    generated name is a separate hub bug worth filing.
>
> 4. **A general label editor.** The discovery opt-in sets one label. Now that
>    rulesets carry labels, an editor for arbitrary ones would be more faithful to
>    the model and would serve the target-augmentation case directly rather than by
>    side effect. Deferred rather than rejected: the opt-in covers the motivating use
>    case, and a label editor needs its own validation and its own explanation of
>    what each label does.
>
> 5. **Orphaned rule files.** Replacing or removing a rule file deletes the rule but
>    leaves the uploaded file in the hub. The custom-target form has the same
>    behavior, acknowledged in a `TODO`. Out of scope here, but the new form should
>    not make it worse and the two should be fixed together.
>
> 6. **Rule-label namespacing.** `discovery` is an unnamespaced label, unlike
>    `konveyor.io/target=…`. It is what the discovery task selects on today, so it is
>    what the opt-in sets. If the label is ever namespaced, the opt-in and the task
>    definition have to move together.

## Drawbacks

This stopped being a UI change. It is a model change in the hub with a migration, a
behavioral change in a separately released add-on, and pages in the UI — three
components that have to land together, and whose skew has a quiet failure mode. The
alternative was a UI that could create rulesets but not say what they were for, and
that could not support the repository option at all, so the scope is bought rather
than accidental. It is still scope.

The always-included change is the riskiest part, because it alters behavior for
existing rulesets rather than only new ones. Any ruleset that today carries labels on
its rules *and* draws from a repository stops having all of its rules run and becomes
subject to the selector. That is the intended correction, and it is hard to construct
a case where the old behavior was wanted, but it is a behavior change to something
already deployed and deserves a release note rather than a footnote.

Cross-participation, covered in
[What a ruleset participates in](#what-a-ruleset-participates-in), is the sharp edge
a user is most likely to cut themselves on. The derived display mitigates it; it does
not remove it.

The feature also hands users a way to create tags and tag categories implicitly, at
discovery time, from strings in a rule file. A typo in a tag action silently produces
a new category. Nothing about that is new — built-in discovery rules work the same
way — but this enhancement is what makes it reachable without API access, and it will
increase the rate at which stray categories appear.

Finally, shipping these pages without
[reclassification](#reclassifying-existing-applications) makes an existing gap
conspicuous. Today nobody notices that a rule change leaves the portfolio untouched,
because nobody can change a rule. A user who registers one, saves it, and sees
nothing happen to their applications will reasonably conclude the feature does not
work — and will be hard to argue with.

## Alternatives

**Deriving the discovery opt-in from rule labels, with no model change.** The UI
would list rulesets whose rules carry the `discovery` label and validate on upload
that every uploaded file carries it. This needs no hub model change and no add-on
change, which made it attractive. Rejected because it makes a property of the ruleset
depend on the contents of files authored elsewhere: editing a rule file silently
un-registers the ruleset, the validation is a trap the user must be taught to avoid,
and repository-backed rulesets remain impossible because there are no rule labels for
the hub to see. It solves the UI problem by leaving the modeling problem in place.

**A kind marker on the ruleset.** Rulesets already carry a kind; the UI would set it
to `discovery` and filter on it. This gives declarative identity for less than a
labels column costs. Rejected because the add-on does not consult it and has no
reason to — selection is label-based, and teaching it a second, parallel mechanism
for one special case is worse than completing the one it has. It would also do
nothing for repository-backed rulesets or for target augmentation.

**Injecting the `discovery` label into uploaded rule files.** The form would rewrite
each rule before upload. Rejected because it silently diverges the hub's copy from
the user's source file. Labels on the ruleset are this idea done properly: the label
lands on the ruleset, the add-on writes it into the manifest it already generates,
and the user's file is never touched.

**A discovery-specific page rather than general ruleset management.** A page named
for discovery, listing only rulesets marked for it. Rejected because a ruleset's
participation is not exclusive — one can feed discovery and an analysis target at
once — so a discovery-named page would be lying about what it manages, and a user
whose ruleset stopped being a discovery ruleset would have nowhere to find it. The
general page costs no more to build and is honest about the model.

**Extending the custom migration targets page.** Rulesets could be a tab or a filter
there. Rejected because a target is a user-selected analysis option and a ruleset is
a unit of rules that may feed anything; they share a storage type and nothing else.
Merging them would mean a form that hides its image, provider, and ordering controls
in one mode and shows them in the other — more work than a separate page, and worse
to use.

**Doing nothing in the UI and documenting the REST API.** The discovery capability
does exist today for anyone willing to POST a ruleset with the right labels inside
their files. Rejected because the persona this epic serves is an Enterprise Architect
classifying a portfolio, not an API client — and because the repository option does
not work from the API either, so documentation alone would not close the gap.

## Implementation History

- 2026-11-03: Initial draft.
