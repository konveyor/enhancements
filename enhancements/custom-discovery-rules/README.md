---
title: custom-discovery-rules
authors:
  - "@jortel"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-11-03
last-updated: 2026-11-05
status: provisional
see-also:
  - "/enhancements/custom-migration-targets/README.md"
  - "/enhancements/centralized-configuration-management/README.md"
  - "/enhancements/dependency-archetype-criteria/README.md"
---

# Custom Discovery Rules

Technology discovery annotates applications with tags, and those tags are what
Archetype criteria match on. The rules that produce them are seeded with the
product and selected by a fixed configuration the operator installs, so what
discovery recognizes — and whether it runs at all — is not something a user can
influence.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

Technology discovery becomes configurable. The hub and UI provide two capabilities:

1. **Configure which rules participate in technology discovery.** The selection is
   presently fixed in the task definition the operator installs. It becomes
   something an architect controls, including the degenerate case of selecting
   nothing, which is how discovery is turned off.

2. **Contribute custom rules to technology discovery.** A user supplies their own
   rules — rules for technologies the shipped rules do not cover, and rules that
   extend what they do cover — and discovery runs them alongside the shipped ones.

The second follows the pattern the product already establishes for custom
migration targets and analysis profiles: rules are authored as YAML and either
uploaded or kept in a source repository the hub fetches, with credentials where
the repository needs them. This enhancement introduces no new way to author or
package rules, and no new concept for a user to learn beyond the one they already
meet when they define a custom target.

Both capabilities are stated here as requirements rather than as a design. The
resource that carries the configuration, how it reaches the analyzer add-on, and
how the UI presents it are design details, sketched in
[Design Details](#design-details) and open to change.

Layout, controls, and wording are UXD's and out of scope here.

## Motivation

Discovery tags an application from two independent sources. Language discovery
inspects the repository directly and reports what it recognizes — `Language=Java`,
`Framework=Spring Boot`, `Tooling=Maven` and the like. Technology discovery runs
rules and tags from whatever they match. Both land on the application as ordinary
tags, and Archetype criteria are defined on top of them.

The rules technology discovery runs are the ones seeded with the product, and
which of them run is decided by configuration installed with the operator. Two
consequences, and they are the whole motivation.

**Users cannot extend what discovery recognizes.** Organizations have internal
frameworks, internal variants of frameworks the product already detects, and
conventions worth classifying on. None of it is reachable, so classification for
those applications is manual, and the Archetype automation built on top of
discovery tags does not reach them.

**Users cannot adjust what discovery does.** The selection is fixed, so an
organization cannot narrow discovery to the rules it cares about, and cannot turn
it off — on a large portfolio it runs on every application create and update
whether or not anyone uses the result.

A secondary observation: rules are already a first-class thing a user can supply.
Custom migration targets accept uploaded rule files or a repository reference, and
analysis profiles do the same. Discovery is the one place rules matter where the
user cannot reach.

### Goals

- An architect can configure which rules participate in technology discovery.
- An architect can turn technology discovery off.
- An architect can contribute custom rules to technology discovery, authored as
  YAML and either uploaded or kept in a source repository.
- Contributing custom rules works the way contributing rules to a custom target or
  an analysis profile works. A user who has done one recognizes the other.
- Custom rules extend the shipped rules rather than replacing them, unless the
  configuration says otherwise.
- Shipped rules continue to work untouched for anyone who configures nothing.

### Non-Goals

- Detailed UX design. Page composition, controls, and copy are out of scope.
- Authoring rules in the UI. Rules are written as YAML and uploaded, or fetched
  from a repository.
- Editing the content of shipped rules.
- Configuring language discovery, which inspects repositories directly and runs no
  rules.
- Contributing custom rules to an *analysis* target. Analysis profiles already
  accept uploaded files and a repository; nothing here extends that.
- General management of rulesets as user-owned objects. An earlier revision of this
  enhancement proposed create/read/update/delete for rulesets; see
  [Alternatives](#alternatives).
- Reclassifying applications that already exist; see
  [Reclassifying existing applications](#reclassifying-existing-applications).
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

Three facts about the existing implementation, each of which bears on what is
being asked for.

**Rules are chosen by label.** The add-on asks the hub for the available rulesets
and keeps those matching the task's label selector; the analyzer then applies the
same selector to the rule files themselves. Everything about which rules run
follows from that selector and from the labels rule authors write. This
enhancement makes the selector configurable; it does not change the selector
language or how labels work.

**Rules can also be supplied directly.** An analysis profile carries rule files and
a repository reference of its own, and rules supplied that way run without being
selected — the add-on marks them always-included. This is the mechanism custom
targets and analysis profiles already rely on, and it is why contributing custom
rules to discovery is a matter of giving discovery somewhere to carry them rather
than inventing anything.

**Discovery is triggered, not launched.** The hub submits discovery tasks when an
application is created or updated. There is no manual launch, in the UI or
elsewhere, which is why turning discovery off has to be configuration rather than
simply not pressing a button.

**Tags come from the rules that run.** They take the form `Category=Value`; the
tagger creates the category and tag if absent and replaces the application's tags
for the `tech-discovery` source — its own, separate from the one language
discovery writes under. From there Archetype matching proceeds normally.

> **Where this behavior lives.** For reviewers wanting to confirm the above: the
> task definition is in `tackle2-operator`
> (`roles/tackle/templates/customresource-task.yml.j2`); rule selection, the
> always-include rewrite, and the tagger are in `tackle2-addon-analyzer`
> (`cmd/rules.go`, `cmd/tagger.go`); the discovery trigger is in `tackle2-hub`
> (`internal/trigger/application.go`), and analysis profiles — the closest existing
> analogue for carrying user-supplied rules — are in `internal/api/profile.go`.

### Configuring which rules participate

The hub holds the configuration that today lives in the installed task definition,
and the UI exposes it. What an architect sets is which rules technology discovery
runs, expressed the way it is expressed today — by label.

The configuration is global. Technology discovery is a property of the
installation, not of an application or an archetype, and nothing in the current
trigger or task definition is parameterized per application.

Turning discovery off is part of this capability rather than a separate feature.
Whether it is expressed as selecting nothing or as an explicit switch is a design
question; what matters is that an architect can stop technology discovery from
running, and that doing so is obvious rather than a side effect of emptying a
field.

Defaults preserve present behavior. An installation where nobody configures
anything runs exactly the rules it runs today.

### Contributing custom rules

An architect supplies rules of their own and discovery runs them. The rules are
YAML, and they are supplied either as uploaded files or as a reference to a source
repository with optional credentials — the same two choices, and ideally the same
UI components, as a custom migration target and an analysis profile.

Custom rules extend rather than replace: they run alongside the shipped rules, so
adding a rule for an internal framework does not cost an organization the
classification it already gets.

Custom rules are not separately labeled or selected by the user. Supplying a rule
to discovery is the act that makes it participate. This is deliberate, and it is
the main lesson taken from the earlier revision of this enhancement, where
contributing a rule and getting it selected were two steps and a user could
plausibly do the first without the second.

### What this does not change

Rules are authored as YAML by hand, outside the product, as they are today. The
label selector language, the order rules run in, and how a rule produces a tag are
untouched. Shipped rules keep running and are not editable. Language discovery is
unaffected. Archetype matching proceeds exactly as it does now, from tags it cannot
distinguish by origin.

### User Stories

#### Registering a rule for an internal framework

An architect has written `acme-platform.yaml`, a rule that detects their in-house
service framework and emits `Internal Framework=Acme Platform`. They add it to the
discovery configuration and save.

The next time an application is created or edited, discovery runs, the rule
matches, and the application is tagged. An Archetype with that criteria tag then
associates the matching applications automatically, and they inherit its assessment
and target profiles.

#### Rules maintained in a repository

The organization keeps its discovery rules in a git repository, reviewed and
released like any other code. The architect points the discovery configuration at
the repository rather than uploading files. Discovery fetches it on each run, so
the rules stay current without anyone re-uploading anything.

#### Narrowing what discovery runs

An organization cares about a handful of technologies and finds the rest of the
shipped classification noise. The architect narrows the configuration to the rules
they want, and discovery stops producing the tags they do not.

#### Turning discovery off

An organization does not use technology discovery and does not want it running
across a large portfolio on every application change. The architect turns it off.
Language discovery continues, and analyses are unaffected.

#### Retiring a rule

A framework has been decommissioned and the architect removes its rule from the
configuration. Discovery stops applying the tag. Applications keep the tags they
have until each is rediscovered, because the tagger replaces the full set for its
source rather than removing tags when a rule goes away.

### Reclassifying existing applications

Discovery runs when an application is created or updated, and nothing else starts
it, so changing the configuration does not reclassify applications already
discovered. This predates custom rules and applies equally to the shipped ones — an
upgrade that corrects a built-in discovery rule reclassifies nothing.

What changes is how visible it is. Today nobody can alter discovery, so nobody
expects an alteration to take effect; once an architect can save a rule, they will
expect their portfolio to reflect it, and watching nothing happen is a poor answer.

The hub already does the right shape of fan-out for credentials: changing one
re-triggers the applications it affects. The equivalent for discovery
configuration is the obvious counterpart, and a global, singular configuration
gives it a far crisper trigger point than a scattering of user-owned rulesets would
have. What is open is cost — the fan-out is the whole portfolio — and whether it
should fire automatically or be an explicit action the architect takes. See
[Open Questions](#open-questions).

## Design Details

This section is deliberately thin. The capabilities above are the requirement; what
follows is the shape the design is expected to take, recorded so reviewers can
judge feasibility, and expected to be replaced by a concrete design before this
enhancement is `implementable`.

### Candidate shape: a discovery profile

The natural fit is a hub resource resembling an analysis profile, reduced to the
parts that concern rules: a label selector, uploaded rule files, a repository
reference with optional credentials, and a way to express that discovery is off.
Singular, since the configuration is installation-wide.

Its attraction is that nearly all of it already exists. Analysis profiles carry
exactly these fields; the add-on already fetches a profile and applies it over the
task data, already fetches rule files and clones a rule repository, and already
marks rules supplied that way always-included. The hub-side work is a resource and
a migration, the add-on work is a second profile type on an existing path, and the
UI is a form built from components the custom-target and analysis-profile pages
already use.

Points to settle when the design firms up, none of which change the requirement:

- **Whether the configuration replaces or merges with the installed task
  definition.** Replacing is how analysis profiles behave, and it makes "run only
  my rules" expressible; merging is harder to get wrong. The choice determines
  whether an architect who clears a field silently disables the shipped rules.
- **Where "off" is enforced** — in the hub's trigger, which creates no task at all,
  or in the add-on, which needs no hub change but schedules a no-op per application
  change. The task definition already labels the technology-discovery kind
  distinctly, so the trigger can tell it apart from language discovery.
- **One repository or several.** An analysis profile carries a single repository
  reference. An organization keeping discovery rules in two repositories is not
  served by that, and it is worth knowing whether that is real before the shape is
  fixed.
- **Whether a singular resource stays singular.** If discovery configuration might
  later vary by archetype or platform, that is cheaper to allow for now than to
  retrofit.
- **Naming.** "Discovery" spans both language and technology discovery in the hub,
  and only the latter is configured here.

### RBAC

Configuring discovery is an architect's act, and discovery affects every
application in the portfolio. Whatever resource carries the configuration should be
readable by every role that reads the rest of the portfolio and writable by
architect and admin, matching how analysis profiles are already granted. Turning
discovery off installation-wide is the one piece with an argument for being
admin-only; see [Open Questions](#open-questions).

The UI needs a write gate equivalent to the one it applies to custom targets:
configuration hidden or read-only without it.

### Test Plan

**Hub.** The configuration round-trips through the API and survives upgrade.
Defaults reproduce present behavior for an installation that configures nothing.
Permissions are enforced per role.

**Add-on.** Discovery runs the configured rules: shipped rules only, shipped plus
custom, custom only, and none. Uploaded rules and repository-sourced rules each
produce tags. A configuration selecting nothing produces no tags rather than
failing.

**Disabled.** Technology discovery does not run; language discovery still does;
existing tags are unaffected; analyses are unaffected.

**UI.** The configuration can be read, edited, and saved; uploads and repository
references behave as they do on the custom-target form; write affordances are
absent without permission.

**End-to-end.** The test proving the premise: supply a single rule file emitting a
novel `Category=Value` tag, create an application whose repository trips the rule,
and assert that discovery tags it, that the category and tag were created, and that
an Archetype with that criteria tag picks the application up. Repeat with the rule
in a source repository to cover the other shape.

### Upgrade / Downgrade Strategy

Nothing changes for an installation that configures nothing: the defaults are the
behavior shipped today, and discovery keeps running the rules it runs now.

The hub, add-on, and UI land together. An older add-on against a newer hub ignores
the configuration and runs discovery as it always did — a quiet failure mode in a
separately released component, and one the design should try to make noisy.

Downgrade loses the configuration and returns discovery to its installed default,
including re-enabling it where an architect turned it off.

## Open Questions

> 1. **Whether reclassification is in scope.** Per
>    [Reclassifying existing applications](#reclassifying-existing-applications),
>    changing discovery configuration does not re-tag existing applications, and
>    users will expect it to. Whether the fan-out lands here, and whether it is
>    automatic or an explicit action, depends on sizing the cost against a full
>    portfolio.
>
> 2. **Whether turning discovery off is an admin act.** It is installation-wide and
>    affects every application, which argues for admin; it is of a piece with the
>    rest of the configuration, which argues for architect alongside everything
>    else here.
>
> 3. **Whether discovery configuration should ever be scoped.** Stated as global,
>    which matches the trigger and task definition. If per-archetype or
>    per-platform discovery is foreseeable, it is worth knowing now.
>
> 4. **UX design.** These capabilities are specified functionally and have no
>    approved design. Per the comment on the parent epic (MTA-8042, 2026-10-27),
>    UXD work on custom discovery rules — including import and platform association
>    — was still in progress at the time of writing; the two need reconciling
>    before scheduling.

## Drawbacks

The change spans the hub, the analyzer add-on, and the UI, which have to land
together.

Configuration that silences shipped rules is easy to get wrong and quiet when it
is. An architect who narrows discovery, or turns it off, gets no error — just
fewer tags, on applications discovered after the change, while applications
discovered before it keep the tags they had. That asymmetry is the existing
reclassification gap, and this enhancement makes it reachable.

Rules remain hand-authored YAML. The product gains a way to supply a rule and no
help writing one, so the documentation carries weight that a richer authoring
surface would not ask of it.

## Alternatives

**Ruleset management as the delivery vehicle.** An earlier revision of this
enhancement proposed create/read/update/delete for user-owned rulesets, with an
opt-in that marked a ruleset for discovery by adding a label to it. It works, and
it generalizes past discovery, but it answers the question at the wrong level: the
user's requirement is to add rules to discovery, and that design asked them to
create a ruleset, understand labels, take an opt-in that governed only whether the
ruleset was *considered*, and separately ensure their rule files carried the label
that governed whether the rules actually *ran*. A user could reasonably do all of
it and get nothing. General ruleset management may still be worth doing; it is not
how this capability should be reached.

**Shipping the configuration in the operator.** The task definition is already
configuration, and exposing it through the operator's CR would cost very little.
Rejected because it puts an architect's routine act behind a cluster-admin edit,
offers no way to supply rule files, and is invisible from the product.

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
