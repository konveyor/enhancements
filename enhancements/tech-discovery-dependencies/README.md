---
title: tech-discovery-dependencies
authors:
  - "@jortel"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-11-18
last-updated: 2026-11-19
status: provisional
see-also:
  - "/enhancements/dependency-archetype-criteria/README.md"
---

# Dependency Reporting from Tech-Discovery

Tech-discovery is the cheap pass that tags an application during onboarding. It runs the
analyzer in source-only mode, applies tags, and returns without creating a report.

This enhancement makes it resolve dependencies and send an analysis to report them, with
an empty insight set. It applies only when the addon runs in discovery mode. Analysis mode
is untouched.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Transitive closure in source-only mode confirmed
- [ ] User-facing documentation is created

## Summary

Two behaviors in `konveyor/tackle2-addon-analyzer` suppress dependency data in discovery
mode: it does not resolve dependencies, and it finishes at tagging without creating a
report. Both change. Report metadata, insights, and dependencies already upload as
separate parts, so this populates the last and leaves the others alone.

Dependencies are reported by sending an analysis, because that is the only carrier the hub
provides for them. The analysis becomes the application's current one, superseding any
previous analysis. That is existing hub behavior, and it is the one consequence worth
weighing before approval. See
[Superseding the Previous Analysis](#superseding-the-previous-analysis).

## Motivation

Dependency data is produced only by a full analysis, which is not routinely run across a
whole portfolio, so for most applications there is none. That is the binding constraint on
anything built over dependencies.
[Dependency-Based Archetype Criteria](../dependency-archetype-criteria/README.md) is the
immediate consumer: dependency criteria would match nothing for an application that has
been onboarded but not analyzed, which is the common state during onboarding and precisely
when automatic classification is most valuable.

Tech-discovery already runs broadly and already runs the analyzer, and the dependency
information is present in the provider output and discarded. A secondary benefit is that
dependencies become as current as the last tech-discovery rather than the last full
analysis.

### Goals

- Resolve and report dependencies in discovery mode, with an empty insight set.
- Keep the change inside `tackle2-addon-analyzer`.

### Non-Goals

- **Changing anything in analysis mode.** Full-analysis behavior is unchanged.
- **Reporting insights from discovery mode.** The addon does not evaluate rules there.
- **What invokes tech-discovery.** The addon reports for the run it is given; triggering is
  hub and UI behavior.
- **A discriminator on `Analysis` for the producing run.** See
  [Superseding the Previous Analysis](#superseding-the-previous-analysis).
- **Per-profile control of dependency collection.** Discovery mode arrives through task
  data rather than profile settings.
- **Changing how the hub selects or archives analyses.** Relied on here, not modified.

## Proposal

### User Stories

#### TDD001

*As an Architect, I want dependencies reported for applications that have been onboarded
but not analyzed, so that features built on dependency data work across the portfolio
rather than on the analyzed subset of it.*

#### TDD002

*As an Architect, I want an application's dependency list refreshed by tech-discovery, so
that keeping it current does not require a full analysis.*

### The Change

In discovery mode, the addon resolves dependencies rather than skipping that step, and
sends an analysis rather than finishing at tagging:

| part | discovery mode *(changes)* | analysis mode *(unchanged)* |
|---|---|---|
| dependencies | populated | populated |
| insights | empty | populated |
| report metadata | populated | populated |

Only the first column changes. Tagging is unchanged in both modes.

One consequence for consumers: the existence of an analysis no longer implies that rules
were evaluated, so an empty insight set now means either "evaluated, nothing found" or
"not evaluated".

### Superseding the Previous Analysis

Dependencies hang off an `Analysis`, so reporting them means creating one, and creating
one has two existing effects in the hub. Selection is newest-wins, so the per-application
insight and dependency endpoints resolve to the most recent analysis. Creation also
archives the prior ones, keeping an aggregate insight summary and deleting their `Insight`
and `TechDependency` rows, with no summary equivalent for dependencies. This is gated by
`ANALYSIS_ARCHIVER_ENABLED`, which defaults to true.

A discovery-mode analysis therefore becomes the application's current one. Insights read
empty, dependencies read as whatever tech-discovery resolved, and values derived from the
latest analysis follow — `Application.effort` goes to zero, since there are no insights to
sum.

Zeroing the insights is correct, not a side effect to be worked around. When an
application is re-discovered, the state of its code is unknown; carrying the prior
insights forward would assert findings that may no longer hold. Reporting dependencies
with no insights states what is actually known: *we know what it depends on; we have not
evaluated rules against this version.* For the same reason, no `addon-mode` discriminator
is added to `Analysis` — it would exist only to defeat superseding, and would make "the
latest analysis" mean different things to different consumers.

How often supersession occurs depends on what invokes tech-discovery, which is hub and UI
behavior and outside this proposal.

### Security, Risks, and Mitigations

Dependency resolution reaches the network for ecosystems that resolve against a remote
index, which discovery mode does not do today, so proxy and credential behavior needs
confirming on this path. Failure is bounded by existing behavior, since dependency
reporting can fail without failing the task, so tagging still succeeds in a
restricted-network environment. Resolution also makes tech-discovery slower.

## Design Details

### To Confirm During Implementation

**Whether dependency resolution returns a complete transitive closure in source-only
mode.** Full analysis mode is selected only when dependencies are requested. An incomplete
closure is worse than a data-quality problem here, because the reported set replaces the
complete set from any prior full analysis and nothing of it is retained. If it is
incomplete, the options are to enable full mode, spending the time tech-discovery exists
to avoid, or to mark the dependencies partial, which makes the direct/transitive
distinction unreliable and does not prevent the replacement. This blocks `implementable`.

**Applications without source** already no-op in the addon. That path must continue to
return without sending an analysis, since an empty one would supersede a real analysis
while asserting nothing.

### Test Plan

| area | cases |
|---|---|
| reporting | a tech-discovery sends an analysis with dependencies and an empty insight set; tagging is unchanged |
| analysis mode | a full analysis resolves and reports exactly as it does today |
| supersession | a tech-discovery after a full analysis becomes current, and a full analysis after a tech-discovery likewise; the prior analysis is archived with its summary intact and its insight and dependency rows removed |
| derived values | reported effort follows the latest analysis — zero after a tech-discovery, restored by a subsequent full analysis |
| resolution | dependencies from discovery mode match those from a full analysis of the same source, or the difference is characterized |
| failure | resolution failure does not fail the task or suppress tagging, including with no route to a package index |
| no source | an application without a repository, or analyzed from an uploaded binary, no-ops without sending an analysis |
| no regression | an application that is never tech-discovered again is unaffected |

### Upgrade / Downgrade Strategy

No schema change and no migration. The addon change takes effect for runs after upgrade,
and existing analyses are untouched until the next tech-discovery for that application.
Downgrade returns discovery mode to tagging without reporting, and analyses it already
sent remain current until a full analysis runs.

Supersession is not reversible by downgrade, so an application tech-discovered in between
keeps only what archiving left of its prior analysis. The release note should say so.

## Open Questions

None.

## Drawbacks

- A tech-discovery clears the insights of a prior full analysis, and archiving reduces
  them to a summary; recovering the detail requires running the analysis again.
- Tech-discovery becomes slower and acquires a network dependency.
- Consumers that assume an analysis implies evaluated rules will be wrong until updated.
- If source-only resolution is incomplete, the proposal either loses its cost advantage or
  replaces complete dependency data with quietly partial data.

## Alternatives

**Merge the resolved dependencies into the latest analysis** rather than sending a new
one, preserving its insights.

This is the only practical alternative, and it has been discussed and rejected. Zeroing
the insights on re-discovery is the expected outcome, not a cost to be avoided: the state
of the application code is unknown at that point, so insights carried forward from an
earlier analysis would assert findings that may no longer hold.

## Implementation History

- 2026-11-18: Extracted from
  [Dependency-Based Archetype Criteria](../dependency-archetype-criteria/README.md), where
  it was a sub-section. Separated because superseding affects all users, including those
  who never use dependency criteria, and because bundling it meant a reviewer objecting to
  it had to block the consuming feature.
- 2026-11-19: Corrected the description of supersession — the hub archives prior analyses
  destructively by default, not only newest-wins selection. Scoped the document to addon
  behavior; what invokes tech-discovery is out of scope.
