---
title: tech-discovery-dependencies
authors:
  - "@jortel"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-11-18
last-updated: 2026-11-18
status: provisional
see-also:
  - "/enhancements/dependency-archetype-criteria/README.md"
---

# Dependency Reporting from Tech-Discovery

Tech-discovery is the cheap pass that tags an application during onboarding. It runs the
analyzer in source-only mode, applies tags, and returns. It does not report dependencies,
and it does not create a report at all.

This enhancement makes tech-discovery resolve dependencies and report them with an empty
insight set.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] Transitive closure in source-only mode confirmed
- [ ] User-facing documentation is created

## Summary

Two behaviors in `konveyor/tackle2-addon-analyzer` suppress dependency data during
tech-discovery: it does not resolve dependencies, and it finishes by applying tags without
creating a report. Both change. The addon already uploads report metadata, insights, and
dependencies as separate parts, so this populates the last and leaves the others as they
are.

The change is confined to the addon. No hub schema change, no API change, no sequencing.

**The consequence is not confined to the addon.** The hub treats the newest report per
application as current, so a tech-discovery report supersedes whatever preceded it —
including a full analysis, and including that analysis's insights. Because tech-discovery
is triggered by an edit to an application, **editing an application will discard its
insights.** That is a behavior change for every user, including those who never consume
dependency data, and it is the substance of this proposal. See
[Superseding an Analysis](#superseding-an-analysis).

## Motivation

Dependency data is produced only by a full analysis today. A full analysis is expensive
and is not routinely run across a whole portfolio, so for most applications in most
estates there is no dependency data at all.

That is the binding constraint on anything built over dependencies.
[Dependency-Based Archetype Criteria](../dependency-archetype-criteria/README.md) is the
immediate consumer: without this change, criteria built on dependencies would match
nothing for applications that have been onboarded but not analyzed — which is the common
state during onboarding, and precisely when automatic classification is most valuable.

Tech-discovery is already run broadly and already runs the analyzer. The dependency
information exists in the provider output and is discarded.

There is a second benefit independent of any consumer. Tech-discovery is triggered by an
edit to an application, so once it reports, an application's dependency set is refreshed
whenever the application changes. Today a dependency list can only be as current as the
last full analysis.

### Goals

- Report dependencies from tech-discovery, with an empty insight set.
- Keep the change inside `tackle2-addon-analyzer`.
- Establish the semantics of a report that carries dependencies and no insights.

### Non-Goals

- **Reporting insights from tech-discovery.** Tech-discovery does not evaluate rules, and
  this does not change that.
- **A discriminator on `Analysis` for the producing run.** See
  [Superseding an Analysis](#superseding-an-analysis).
- **Per-profile control of dependency collection.** Tech-discovery is not a profile
  setting; it arrives through the task data. Varying collection by profile would require
  extending the analysis profile mode first.
- **Changing how the hub selects the current report.** The newest-wins behavior is
  existing and is relied on here, not modified.

## Proposal

### User Stories

#### TDD001

*As an Architect, I want dependencies reported for applications that have been onboarded
but not analyzed, so that features built on dependency data work across the portfolio
rather than on the analyzed subset of it.*

#### TDD002

*As an Architect, I want an application's dependency list to refresh when the application
is edited, so that it reflects what the application currently points at.*

#### TDD003

*As an Architect, I want to be able to tell whether an application has zero insights
because it was analyzed and found clean, or because it was never analyzed, so that I do
not read "no findings" as "no problems".*

### The Change

During tech-discovery, the addon resolves dependencies rather than skipping that step, and
creates a report rather than finishing at tagging. The report carries:

| part | tech-discovery | full analysis |
|---|---|---|
| dependencies | populated | populated |
| insights | empty | populated |
| report metadata | populated | populated |

Tagging behavior is unchanged.

### Superseding an Analysis

This is the part that needs agreement before anything is written.

The hub already treats the newest report per application as the current one. Creating a
report from tech-discovery therefore supersedes whatever came before it, including a full
analysis with insights. Since tech-discovery is triggered by an edit, the effect is:

> **Editing an application discards its insights.**

The position of this proposal is that this is correct, not a side effect to be worked
around.

An edit can change what the application points at — a different repository, a different
branch, a different path. The insights of the previous analysis then describe a codebase
the application no longer refers to, and continuing to report them asserts findings about
the wrong thing. "Latest report has dependencies and no insights" accurately states *we
know what it depends on; we have not evaluated rules against this version*.

This also closes an existing gap. Today an application can be edited and retain a stale
analysis indefinitely, because nothing invalidates it. The insights survive the edit and
are silently wrong.

**No `addon-mode` discriminator is added to `Analysis`.** Marking the producing run on the
report would exist only to defeat the superseding behavior described above, and it would
make "the latest report" mean different things to different consumers — some filtering to
analyzer-produced reports, some not.

**The counter-argument, stated plainly.** Not every edit changes what the application
points at. Renaming an application, or editing its business service, discards insights
that were still perfectly valid, and the user gets no warning. If that is judged too
costly, the alternative is narrowing the tech-discovery trigger to edits that change the
application's source coordinates — which is a change to the trigger, not to this proposal,
and is worth considering on its own merits.

### The Clean-Application Ambiguity

After a tech-discovery, the UI shows zero insights. That is indistinguishable from a full
analysis of a genuinely clean application. One means "not evaluated"; the other means
"evaluated, nothing found". Reporting an application as clean when it was never checked is
a meaningful misstatement (TDD003).

Content cannot separate them — a clean application also has dependencies and zero
insights. Provenance is available outside the report, in the application's task history,
which distinguishes an `analyzer` run from a `tech-discovery` run. That is where the UI
should resolve it, rather than denormalizing the producing run onto the report.

This is listed as an [open question](#open-questions) because it requires UI work that is
not in this proposal, and because shipping the hub side first leaves a window where the
misstatement is live.

### Security, Risks, and Mitigations

Dependency resolution reaches the network for ecosystems that resolve against a remote
index. Tech-discovery currently does not, so applications that could previously be
tech-discovered in an isolated environment may now fail or stall. Behavior under existing
proxy and credential configuration needs to be confirmed, since those paths are exercised
by full analysis today but not by tech-discovery.

Resolution also makes tech-discovery slower, which matters because it is triggered by
ordinary application edits rather than by an explicit user action.

## Design Details

### To Confirm During Implementation

**Whether dependency resolution returns a complete transitive closure in source-only
mode.** Full analysis mode is selected only when dependencies are requested; without it
the addon is doing source analysis alone. If the dependency list is incomplete in that
mode, tech-discovery must either enable full mode — spending the time tech-discovery
exists to avoid — or mark the reported dependencies partial.

Marking the data is a narrower discriminator than marking the producing run, so it does
not conflict with the position in
[Superseding an Analysis](#superseding-an-analysis), but it would make the
direct/transitive distinction unreliable for consumers.

This blocks `implementable`: if the closure is incomplete and full mode is required, the
cost/benefit of the whole proposal changes.

### Test Plan

| area | cases |
|---|---|
| reporting | a tech-discovery produces a report with dependencies and an empty insight set; tagging behavior is unchanged |
| supersession | a tech-discovery after a full analysis supersedes it entirely — insight reporting returns nothing, dependency queries return the new set; a full analysis after a tech-discovery likewise supersedes it |
| edit trigger | editing an application runs tech-discovery and the resulting report replaces the prior analysis |
| no regression | an application that has only ever had a full analysis is unaffected |
| resolution | dependencies reported from tech-discovery match those from a full analysis of the same source, or the difference is characterized (see above) |
| provenance | task history distinguishes an `analyzer` run from a `tech-discovery` run for the same application |

### Upgrade / Downgrade Strategy

No schema change and no migration. The addon change takes effect for runs after upgrade;
existing reports are untouched.

Downgrade returns tech-discovery to tagging without reporting. Reports already produced by
a tech-discovery remain and continue to be treated as current, so an application whose
most recent report came from a tech-discovery will show dependencies and no insights until
a full analysis runs. No data is lost and nothing needs reconciling.

**The upgrade hazard is behavioral and is not reversible by downgrade:** between upgrade
and downgrade, any application that was edited has lost its insights. Release notes should
say so directly.

## Open Questions

1. **Does the UI distinguish "not evaluated" from "evaluated, nothing found" before this
   ships, or after?** *This does not block `implementable`, but it does gate the release
   note.* The data needed is in the task history and no model change is required. Shipping
   the addon change first leaves a window in which an un-analyzed application reads as
   clean.

2. **Should the tech-discovery trigger narrow to edits that change source coordinates?**
   Discarding insights is correct when an edit repoints the application and
   over-aggressive when it renames it. Narrowing the trigger would preserve the correctness
   argument while removing most of the cost. It is out of scope here because it changes
   the trigger rather than the reporting, but it is the obvious mitigation if the cost is
   judged too high.

## Drawbacks

**Editing an application destroys its insights, and the user is not warned.** This is the
substance of the proposal rather than an incidental cost. It is defensible when the edit
repoints the application and hard to defend when it does not — see open question 2.

Tech-discovery becomes slower and acquires a network dependency it did not have, on a path
triggered by ordinary edits rather than by an explicit action.

A report that carries dependencies and no insights is a new shape in the system. Consumers
that assume a report implies evaluated rules will be wrong, and the ambiguity above is the
user-facing form of that.

If source-only resolution turns out to be incomplete, the proposal either loses its cost
advantage or ships dependency data that is quietly partial.

## Alternatives

**Report dependencies without creating a report.** Not possible as the hub is built —
dependencies hang off an `Analysis`, so reporting them means creating one.

**Add an `addon-mode` discriminator to `Analysis`** so consumers can prefer the latest
*analyzer* report for insights and the latest *any* report for dependencies. Rejected: it
exists only to defeat superseding, and it makes "the latest report" ambiguous. It also
preserves the existing gap where an edited application keeps insights describing a
codebase it no longer points at.

**Keep insights and merge dependencies into the existing report.** Rejected for the same
reason — it asserts findings about a codebase that may have changed, and it makes report
content depend on the history of runs rather than on one run.

**Run a full analysis on edit instead.** Correct insights and correct dependencies, at a
cost tech-discovery exists to avoid. Not viable as a default; an explicit re-analysis
action already covers users who want it.

**Leave it alone and require a full analysis for dependency data.** The status quo. It
confines dependency-based features to the analyzed subset of a portfolio, which for most
estates is a small fraction, and leaves dependency lists as stale as the last full run.

## Implementation History

- 2026-11-18: Extracted from
  [Dependency-Based Archetype Criteria](../dependency-archetype-criteria/README.md), where
  it was a sub-section. Separated because the superseding behavior is a behavior change
  affecting all users, including those who never use dependency criteria, and because
  bundling it meant a reviewer objecting to it had to block the consuming feature.
