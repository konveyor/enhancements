---
title: custom-discovery-rules
authors:
  - "@jortel"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-11-03
last-updated: 2026-11-04
status: provisional
see-also:
  - "/enhancements/custom-migration-targets/README.md"
  - "/enhancements/dependency-archetype-criteria/README.md"
---

# Custom Discovery Rules

Technology discovery annotates applications with tags, and those tags are what
Archetype criteria match on. The rules that produce them are seeded with the product,
so what discovery is able to recognize is fixed.

Extending it is nearly possible already. The discovery task selects what it runs by
label across every ruleset in the hub, including ones a user created, so a ruleset
whose rules are labeled `discovery` already participates. What is missing is a way to
create one — and, when the rules are kept in a source repository, anything for that
selection to match: a ruleset's labels come from its rules, and a repository-backed
ruleset has none recorded in the hub.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

A ruleset becomes something a user manages on its own. The UI gains create, read,
update, and delete for the rulesets a user owns, and the form carries an opt-in
marking a ruleset for technology discovery.

The change, in full:

- **UI.** Pages to manage the rulesets a user owns, with the discovery opt-in and a
  matching filter. Taking the opt-in adds `discovery` to the ruleset's declared
  labels, and that is all it does.
- **Hub.** A ruleset gains declared labels. The ruleset resource additionally
  reports whether it is user-created and which target owns it, and refuses to delete
  one an owner still references.
- **Analyzer add-on.** Ruleset selection considers a ruleset's declared labels as
  well as its rules'. That is the whole of it.

The opt-in settles one question only: whether discovery *considers* the ruleset.
Which of its rules then run is decided where it already is — by the labels in the
rule files, against the task's selector. The UI does not inspect, generate, or
modify rule content to make the two agree, and the hub takes no view on which rules
in a ruleset are for what.

That keeps a mixed ruleset working. One holding a rule labeled `discovery` and
another labeled for EAP 8 contributes the first to discovery and the second to an
EAP 8 analysis, and is selected by both. It also means the responsibility is the
user's: mark a ruleset for discovery whose rules carry no such label and it is
fetched and contributes nothing.

Most of the weight is in the UI. Declared labels and the matching filter change
exist for repository-backed rulesets, which cannot otherwise be selected; the other
hub changes are what let the pages leave shipped and target-owned rulesets alone.

Discovery is the motivating case but not the only one. A ruleset participates in
whatever selects its labels, so one labeled for a migration target augments that
target. These are therefore general ruleset pages with a discovery opt-in, not pages
that treat discovery rulesets as their own kind of thing.

Layout, controls, and wording are UXD's and out of scope here.

## Motivation

Discovery tags an application from two independent sources. Language discovery
inspects the repository directly and reports what it recognizes — `Language=Java`,
`Framework=Spring Boot`, `Tooling=Maven` and the like. Technology discovery runs
rulesets and tags from whatever their rules match. Both land on the application as
ordinary tags, and Archetype criteria are defined on top of them.

The rulesets technology discovery runs are the ones seeded with the product. Users
want to add to them — rules for technologies the seeded rules do not cover, and rules
that extend what they do cover, such as an internal variant of a framework the
product already detects. Both are the same mechanism: anything labeled `discovery`
runs alongside the seeded rulesets, so a custom ruleset extends the set rather than
replacing it. Without it, classification for those applications is manual.

A second and smaller motivation: rulesets are not manageable in the UI at all. The
only way to create one is through the REST API, or indirectly by creating a custom
migration target, which creates a ruleset as a side effect and does not show it.

### Goals

- A user can create, view, update, and delete rulesets they own, and manage the rule
  files in them.
- Marking a ruleset for technology discovery is a single act in the UI, the same one
  whether its rules are uploaded or kept in a repository.
- A ruleset may serve more than one purpose, its rules divided by their own labels.
- Repository-backed rulesets can participate in discovery.
- A user can see what each ruleset participates in.
- Shipped rulesets, and rulesets owned by a migration target, are not managed here
  and cannot be damaged from here.

### Non-Goals

- Detailed UX design. Page composition, controls, and copy are out of scope.
- Authoring rules in the UI. Rules are written as YAML and uploaded, or fetched from
  a repository.
- Managing shipped rulesets or those owned by migration targets, or enabling and
  disabling the shipped discovery rulesets.
- Reclassifying applications that already exist; see
  [Reclassifying existing applications](#reclassifying-existing-applications).
- Repository-backed rulesets augmenting an analysis target. Declaring a target label
  on one would mechanically work, but no part of this design exercises that path end
  to end.
- Changing the label selector language, the order rules run in, or how results are
  reported.
- A CLI surface. The hub's client bindings already expose ruleset CRUD.

## Proposal

### Background: how rules are selected and applied

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

Four facts from the existing implementation, each of which this design depends on.

**Selection happens in two passes.** To resolve the included labels, the add-on asks
the hub for every ruleset and keeps those whose labels match — where a ruleset's
labels are the union of the labels recorded on its rules. The matched rulesets' rule
files are then fetched, and the same selector is applied a second time by the
analyzer, against the contents of those files.

**A ruleset can also be named explicitly.** Task data may carry a list of ruleset
IDs, fetched regardless of labels. The analysis wizard populates it with the ruleset
of every selected target — which is how a repository-backed custom target works at
all, since such a target ends up with no labels of its own.

**Rules fetched from a repository are force-included.** Before handing rules to the
analyzer, the add-on rewrites the manifest in certain directories to mark their rules
always-included, so they run whatever the selector says. It applies this to the
profile's own uploaded files, the profile's own repository, and every selected
ruleset's repository checkout — replacing the manifest's `labels` key outright. The
effect that matters here: when a ruleset's rules come from a repository, selecting
the ruleset runs everything in it. Rules uploaded to a ruleset are not treated this
way and remain subject to the selector. A repository-backed custom target depends on
this and the preceding point together — named by ID, then force-included, it
contributes rules despite carrying no labels of its own.

**Tags come from the rules that survive both passes.** They take the form
`Category=Value`; the tagger creates the category and tag if absent and replaces the
application's tags for the `tech-discovery` source — its own, separate from the one
language discovery writes under. From there Archetype matching proceeds normally.

> **Where this behavior lives.** For reviewers wanting to confirm the above: the task
> definition is in `tackle2-operator`
> (`roles/tackle/templates/customresource-task.yml.j2`); selection, the force-include
> rewrite, and the tagger are in `tackle2-addon-analyzer` (`cmd/rules.go`,
> `cmd/tagger.go`); the ruleset resource, the label filter, the built-in guards, the
> target cascade, and the discovery trigger are in `tackle2-hub`
> (`internal/api/ruleset.go`, `internal/api/target.go`,
> `internal/trigger/application.go`); and the analysis wizard's task data, including
> the target ruleset refs, is built in `tackle2-ui`
> (`client/src/app/pages/applications/analysis-wizard/useTaskGroupManager.ts`).

### The label model

Labels appear in three places, and most of this design is a consequence of keeping
them straight.

**In the rule files.** A ruleset manifest carries labels, each rule carries labels,
and a rule's effective set is the union of the two — a ruleset's labels are inherited
by its rules. This is analyzer-lsp's own model, and it is the only one that decides
**what runs**. The selector is evaluated against these.

**`Rule.Labels` in the hub.** Extracted from the rule files when the ruleset is
created. Used for **selection** — deciding which rulesets the add-on fetches — and by
the hub's ruleset label filter. It has no part in deciding what runs.

**`RuleSet.Labels` in the hub.** *Declared* by the caller rather than extracted from
anything. Also used for **selection** only. It is strictly necessary for one case —
rules that live in a source repository, where the hub has nothing to extract from —
but the opt-in sets it on any ruleset, so that marking one for discovery is a single
act whatever its rules are.

A ruleset's selection labels are the union of `RuleSet.Labels` and the labels of its
rules.

So the hub's two label fields share a single purpose — getting the right rulesets
fetched — and neither has any bearing on which rules then run. That is settled
afterwards by the analyzer, against the files themselves. Holding those two questions
apart is what keeps the rest of this design small.

### Marking a ruleset for discovery

The opt-in adds `discovery` to the ruleset's declared labels. Nothing else happens:
no rule file is read to check it, none is generated, and none is modified.

That is sufficient for the ruleset to be *selected*. What then runs is a separate
question, answered by the labels in the rule files against the task's selector, and
it differs by shape.

**Uploaded rule files.** The rules carrying the `discovery` label run — whether they
carry it individually or inherit it from a `ruleset.yaml` in the upload. Rules
carrying other labels do not, which is what lets one ruleset serve more than one
purpose; see [Mixed rulesets](#mixed-rulesets). If no rule carries the label, the
ruleset is fetched and contributes nothing. Putting the label where it belongs is
the user's job, and the documentation's.

**A source repository.** Everything in it runs, because rules fetched from a
repository are force-included. Labeling them is unnecessary, though harmless and
probably good practice. To take only part of a repository, point the ruleset at a
subdirectory — a ruleset's repository reference carries a path, so a repository
holding `discovery/` and `migration/` is served by two rulesets.

A repository-backed ruleset with no declared labels is inert: nothing will ever
select it. The form should say so rather than let a user create one and wonder.

#### Mixed rulesets

A ruleset is not required to serve a single purpose. One holding a rule labeled
`discovery` and another labeled `konveyor.io/target=eap8`, with the opt-in taken, is
selected by discovery on its declared label and by an EAP 8 analysis on its rules' —
and each run executes only the rules that match it.

This is why the opt-in stops at selection. Anything that made it guarantee execution
— generating a manifest, or having the add-on inject the label — would push
`discovery` onto every rule in the ruleset through manifest inheritance, and the EAP
8 rule would start running during discovery. The UI would be silently changing what
the user's rules mean, and the label fields would acquire a second job beyond
selection, which [the label model](#the-label-model) exists to prevent.

The shape works for uploaded rules. It does not work for a repository, where
force-include takes everything; split by path instead.

### What a ruleset participates in

A ruleset's labels determine how it is used. The `discovery` label makes it part of
technology discovery. Labels matching a migration target's make it part of analyses
selecting that target. A ruleset may carry both, and take part in both.

The second follows from what a target is: a named set of labels, plus the
presentation that makes it selectable. Selecting a target contributes nothing but its
labels to the analysis, and the target's own ruleset gets no special path — it is
matched like anything else carrying those labels. There is no distinction in the
system between "the target's rules" and anyone else's rules carrying the target's
label, and there never has been.

That makes augmenting a shipped target straightforward for uploaded rules. Label them
for EAP 8 and they join every analysis selecting EAP 8, with no custom target to
create and no second tile for the user to remember to select.

Because labels decide this, the pages show what each ruleset participates in, derived
from the union of its labels. Someone who starts from a shipped rule and keeps its
target label can then see that the ruleset takes part in that target's analyses as
well as in discovery.

### What the pages manage

Rulesets the user owns. Two kinds are excluded, and each needs the hub to report
something it currently keeps to itself.

**Shipped rulesets.** A seeded ruleset carries a UUID, and the hub treats that as the
definition of built-in, already refusing updates and deletes. It does not report the
distinction to clients, though the target resource does — so the ruleset resource
gains the same property. Shipped rulesets continue to run; they are not listed.

**Rulesets owned by a migration target.** A custom target creates a ruleset as part
of creating itself, and deletes it along with itself. These are an implementation
detail of that page, so the ruleset resource reports its owner and these pages
exclude the ones that have one.

The owner reference also closes a hole. Deleting a ruleset checks only whether it is
built-in, and nothing at the storage layer prevents it, so the target is left
pointing at a ruleset that no longer exists — and then reports no ruleset at all,
which is what the custom-target form reads when it opens for edit. Nothing in the UI
deletes rulesets today; adding a delete action makes this reachable, so the guard
comes with it.

### User Stories

#### Registering a rule for an internal framework

An architect has written `acme-platform.yaml`, a rule labeled `discovery` that
detects their in-house service framework and emits
`Internal Framework=Acme Platform`. They create a ruleset, upload the file, take the
discovery opt-in, and save.

The next time an application is created or edited, discovery runs, the rule matches,
and the application is tagged. An Archetype with that criteria tag then associates
the matching applications automatically, and they inherit its assessment and target
profiles.

#### Rules maintained in a repository

The organization keeps its discovery rules in a git repository, reviewed and released
like any other code. The architect creates a ruleset pointing at it rather than
uploading files, takes the opt-in, and saves. Discovery clones the repository on each
run, so the rules stay current without anyone re-uploading anything.

Later the repository grows a second directory of rules meant for analysis rather than
discovery. Rather than labeling around it, the architect edits the ruleset to point
at the `discovery/` subdirectory.

#### Extending a built-in target without forking it

An architect wants their own checks to run whenever anyone analyzes for EAP 8. They
upload rules carrying the EAP 8 target label, without taking the discovery opt-in or
creating a custom target. The rules join any analysis selecting that target, and the
ruleset's detail says so.

#### Retiring a rule

A framework has been decommissioned and the architect deletes its ruleset. Discovery
stops applying the tag. Applications keep the tags they have until each is
rediscovered, because the tagger replaces the full set for its source rather than
removing tags on delete.

### Reclassifying existing applications

Discovery runs when an application is created or updated, and nothing else starts it,
so changing a rule does not reclassify applications already discovered. This predates
custom rules and applies equally to seeded ones — an upgrade that corrects a built-in
discovery ruleset reclassifies nothing. It is simply more visible once a user can
author a rule, save it, and watch nothing happen.

The hub already does the right shape of fan-out for credentials: changing one
re-triggers the applications it affects. The equivalent for rulesets, and for
seeding, is filed as a follow-up Story. What is open is cost — the fan-out is the
whole portfolio — and whether it should fire automatically at all, given that
re-running discovery replaces an application's tags wholesale. Nothing designed here
depends on it or precludes it.

## Design Details

### Hub changes

**Declared labels on the ruleset.** Stored and settable through the API, per
[The label model](#the-label-model). This is the one change touching stored data, so
it carries a migration.

**Whether a ruleset is user-created.** The hub already knows this and acts on it but
does not report it. The target resource exposes the distinction as `custom`; the
ruleset resource should expose it identically — derived, not stored, ignored on
write.

**The owning target, if there is one.** An optional reference, absent on a
free-standing ruleset. It reverses a relationship the hub already records in the
other direction, so it is a presentation change rather than a data model change.

**Deleting an owned ruleset is refused**, with the owning target named in the error.
Deleting the *target* must continue to remove its ruleset: that cascade is the
supported way to get rid of one and must not be caught by the new rule. Refusing it
the way the hub refuses to delete a built-in keeps the two consistent.

**The ruleset label filter matches the union** of declared and rule labels, so the
hub agrees with the add-on about what a ruleset's labels are and the pages can find
repository-backed rulesets.

### Add-on changes

**Selection considers declared labels.** A ruleset's labels become the union of its
declared labels and its rules', rather than its rules' alone. That is the only
change, and it is purely additive: rulesets with no declared labels behave as they do
now. Force-include is untouched, and the design relies on it as it stands.

### Pages

Ruleset management with a discovery opt-in. How it is composed — a page with a
drawer, a modal, a wizard — is UXD's call; what follows is what it has to do and the
data it has to work with. The closest existing analogue is *Custom migration
targets*, which already does most of this for a different purpose and should be the
reuse baseline. It belongs beside that page, in the Developer perspective's
configuration group, since ruleset and target write permissions are held by the same
roles.

#### Listing rulesets

Lists the rulesets the user owns, excluding shipped and target-owned ones.

Available per ruleset without a further request: name, description, its declared
labels, the rules it contains with their labels, whether it draws from a repository,
and who created or last updated it and when. A rule count, the union of labels, and
therefore what the ruleset participates in all derive from that. The tags a ruleset
emits do not, because they live in file content.

Must offer create, edit, delete, and a way to inspect a ruleset without editing it,
and must show what each ruleset participates in. Filtering to the discovery rulesets
should be a single toggle rather than a label query — the mirror of the opt-in on the
form. Delete should make clear that applications keep their existing tags until
discovery next runs for them.

The empty state carries real weight: it is the only place a user can learn what a
ruleset is for, that taking the discovery opt-in is how custom classification
happens, and where the rule-authoring documentation is.

#### Creating and editing a ruleset

Needs a name, an optional description, a source of rules, and the discovery opt-in.

The source is either uploaded files or a repository reference with optional
credentials — the same choice the custom-target form offers.

The opt-in adds `discovery` to the ruleset's declared labels and does nothing else,
for either source. What the form has to carry instead is the explanation, per
[Marking a ruleset for discovery](#marking-a-ruleset-for-discovery): that the rules
intended for discovery must be labeled as such, and that a repository participates
whole, narrowed by pointing at a subdirectory.

Name is required and must be unique across all rulesets, not only the listed ones,
because ruleset names are globally unique and targets also create rulesets.
Validating against the full set gives an accurate error rather than an opaque
conflict from the hub; see [Open Question 3](#open-questions).

Uploads reuse the rule-file upload the custom-target form already uses, with its
existing file-type and YAML syntax checks. Per-file feedback should report the rules
parsed and the tags they emit, which the form has in hand while it holds the
contents, and which is the cheapest moment to catch a typo.

On save, one rule is recorded per file with the file's name, the labels parsed from
it, and a reference to the upload. Rule labels are derived from the file, never
typed. On edit, the hub removes any rule absent from the submitted ruleset, so
untouched files still have to be submitted. Note that replacing a file leaves the old
upload behind, as the custom-target form already does; worth fixing in both together.

The hub permits a ruleset with both uploaded rules and a repository, and the add-on
handles it by treating the two as separate rule directories with separate manifests.
The form need not offer the combination, but the detail view should render both
sections, since the API can produce one.

#### Inspecting a ruleset

Shows the ruleset's metadata, what it participates in, and its rules — for uploaded
files, the rules in each file with their labels and emitted tags, fetched and parsed
on demand; for a repository, the reference and credentials, since the hub holds no
rules for it. Offering the file for download
closes the loop for a user auditing what is registered.

### RBAC

No new permissions. Reading rulesets is already granted to every role that can read
the rest of the portfolio; writing them is granted to architect and admin, which
matches the persona this epic serves. The UI needs a write gate for rulesets
equivalent to the one it applies to targets: create, edit, and delete hidden without
it, the list still readable.

Ruleset write is a single permission covering both these rulesets and the ones behind
custom migration targets, so granting authorship of one grants the other. The
existing roles already pair them, so this introduces no new privilege.

### Test Plan

**Hub.** Declared labels round-trip through create and update, and the migration
leaves existing rulesets with none. User-created is reported correctly for a seeded
and a user-created ruleset; the owning target is present on an owned ruleset and
absent otherwise. The label filter matches by declared labels, by rule labels, and by
both.

The delete guard has four cases, the second being the regression risk: deleting a
target-owned ruleset is refused and the target still resolves its ruleset; deleting
the target still removes the ruleset; deleting a free-standing user-created ruleset
succeeds; deleting a built-in is still refused.

**Add-on.** Selection by declared labels, by rule labels, by both, and no match with
neither. Plus a regression test that a repository-backed ruleset selected on its
declared labels still runs its repository in full, which is what makes the opt-in
sufficient without labeling anything in the repository — and which a future narrowing
of force-include would break.

**UI.** The opt-in adds the declared label and leaves uploaded content untouched,
including an upload that carries no `discovery` label at all. Shipped
and target-owned rulesets
excluded from the list. Participation derived correctly for a discovery label, a
target label, and both. Duplicate names caught; write affordances absent without
permission; a repository-backed ruleset can be created and edited.

**End-to-end.** The test proving the premise: create a ruleset from a single uploaded
rule file emitting a novel `Category=Value` tag, with no `discovery` label in the
file, take the opt-in, create an application whose repository trips the rule, and
assert that discovery tags it, that the category and tag were created, and that an
Archetype with that criteria tag picks the application up. Repeat with the rules in a
source repository, carrying no `discovery` label anywhere, to cover the other shape.

### Upgrade / Downgrade Strategy

The declared labels column is additive and empty for every existing row, so nothing
changes what it matches on upgrade. No existing behavior is modified — the add-on
change only widens what selection considers. The pages appear and are empty until a
user creates something.

A ruleset created through the REST API beforehand, with rules labeled `discovery`,
appears in the list and keeps participating — it is selected on its rule labels, as
it always was.

Downgrade requires the add-on and hub to move together. An older add-on ignores
declared labels, so a repository-backed ruleset stops being selected; rulesets whose
rules carry labels in file content are unaffected. The derived properties and the
delete guard are safe to lose.

## Open Questions

> 1. **When reclassification lands.** Whether the trigger described in
>    [Reclassifying existing applications](#reclassifying-existing-applications)
>    belongs in this release depends on sizing the fan-out cost.
>
> 2. **UX design.** These pages are specified functionally and have no approved
>    design. Per the comment on the parent epic (MTA-8042, 2026-10-27), UXD work on
>    custom discovery rules — including import and platform association — was still
>    in progress at the time of writing; the two need reconciling before scheduling.
>
> 3. **Name collisions with custom migration targets.** Ruleset names are globally
>    unique, and a target-owned ruleset does not keep its generated name for life:
>    created as `__Target(<name>)-<uuid>`, it is renamed to the target's plain name
>    the first time the target is edited. From then on a user ruleset of that name is
>    blocked by a ruleset the user cannot see. Validating against all rulesets makes
>    the error accurate; making it comprehensible is a UX question, and whether the
>    rename should happen at all is a separate hub bug worth filing.
>
> 4. **Declared labels beyond discovery.** The opt-in sets one label, but the field
>    holds a set, so a repository-backed ruleset could in principle declare a target
>    label and augment an analysis target — closing the gap that currently makes
>    repository-backed rulesets useful only for discovery. Nothing here exercises
>    that path and it is listed as a non-goal, but it is the natural next step and
>    the reason the field is a label set rather than a flag.

## Drawbacks

The change spans the hub, the analyzer add-on, and the UI, which have to land
together. An older add-on against a newer hub ignores declared labels and silently
fails to select a repository-backed ruleset — a quiet failure mode in a separately
released component.

The opt-in can be taken on a ruleset whose rules carry no `discovery` label, and
nothing will say so. The ruleset is fetched on every discovery run and contributes
nothing. This is the deliberate cost of keeping labels to a single meaning — the UI
would have to read, and in some cases rewrite, rule content to prevent it, which
would break [mixed rulesets](#mixed-rulesets). It makes the documentation
load-bearing in a way a validating form would not.

A repository is coarser still: it participates whole, so splitting discovery rules
from others means separate rulesets or separate paths rather than labeling. A user
with one mixed repository will feel that.

## Alternatives

**The hub's discovery trigger naming rulesets by ID.** The trigger could resolve the
participating rulesets and list their IDs in the task data, needing no declared
labels. Rejected because the explicit-ID channel already means "run all of this" —
that is what makes repository-backed custom targets work — so injected refs would
need a second, contradictory meaning on the same field. It also cannot extend to
analysis: a task that names a profile has its task-data rules discarded and rebuilt
from the profile, which has no ruleset list.

**A kind marker, or a list of permitted usages, on the ruleset.** Mark the ruleset
`discovery` rather than labeling it. Rejected because it is a second selection
mechanism alongside the one that exists, and because it cannot carry a parameter: a
ruleset that should augment EAP 8 must name *which* target, which a usage enum cannot
express and a label does naturally.

**A discovery-specific page.** Named for discovery, listing only rulesets marked for
it. Rejected because participation is not exclusive — one ruleset can feed discovery
and an analysis target at once — so the page would misdescribe what it manages, and a
ruleset that stopped being a discovery ruleset would have nowhere to be found.

## Implementation History

- 2026-11-03: Initial draft.
