---
title: UI and UXD redesign after PatternFly 6
authors:
  - "@sjd78"
reviewers:
  - "TBD"
approvers:
  - "TBD"
creation-date: 2026-04-02
last-updated: 2026-04-02
status: provisional
see-also:
  - "https://github.com/konveyor/enhancements/issues/269"
  - "https://github.com/konveyor/enhancements/issues/270"
  - "https://github.com/konveyor/tackle2-ui/issues/2172"
  - "https://github.com/konveyor/enhancements/issues/202"
  - "https://github.com/konveyor/tackle2-ui/issues/1932"
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# UI and UXD redesign after PatternFly 6

> **Living document** — Visuals and detailed flows will be appended here as the UXD team finalizes designs. Update `last-updated` in the front matter when this file changes.


## Release Signoff Checklist

- [ ] Enhancement is `implementable` (expected after UX signoff)
- [ ] Design details documented from UXD deliverables
- [ ] Test plan defined (including Quick Start / tour completion flows)
- [ ] User-facing documentation updated


## Summary

This enhancement tracks [RFE issue #269](https://github.com/konveyor/enhancements/issues/269): **after** the UI is on PatternFly 6 ([enhancements#270](https://github.com/konveyor/enhancements/issues/270), [tackle2-ui#2172](https://github.com/konveyor/tackle2-ui/issues/2172)), introduce a **dashboard-first landing experience**, **Quick Starts** for the two most common user actions, and **guided tours** for multi-page workflows. Prior exploration lives in [enhancements#202](https://github.com/konveyor/enhancements/issues/202) and [tackle2-ui#1932](https://github.com/konveyor/tackle2-ui/issues/1932).


## Dependency

| Prerequisite | Reference |
| ------------ | --------- |
| PatternFly 6 migration | [enhancements#270](https://github.com/konveyor/enhancements/issues/270), enhancement draft [ui-patternfly-6](../ui-patternfly-6/README.md), [tackle2-ui#2172](https://github.com/konveyor/tackle2-ui/issues/2172) |

This work should not block the PF6 upgrade; it **starts** once PF6 is in place so Quick Starts, tours, and dashboard patterns can use PF6-only components and extensions.


## Motivation

Today users land on **application inventory** by default. A purpose-built **dashboard** can orient new and returning users, surface status at a glance, and anchor onboarding. **Quick Starts** and **guided tours** reduce time-to-value and align with [PatternFly quick starts](https://www.patternfly.org/extensions/quick-starts/) once the stack is current (see historical proposal in [enhancements#202](https://github.com/konveyor/enhancements/issues/202)).


### Goals

- Replace default post-login landing with a **dashboard** (content TBD with UXD).
- Ship **two** Quick Starts covering the highest-impact common actions (exact tasks TBD with UXD).
- Add **guided tours** that span multiple pages for core use patterns (scope TBD with UXD).
- Align visual language with **OpenShift console** expectations where practical (may overlap themes from [enhancements#270](https://github.com/konveyor/enhancements/issues/270) follow-ups).

### Non-Goals

- Implementing this enhancement **before** PF6 migration is complete.
- Full visual parity with OpenShift console in one release (incremental).
- Rewriting all product workflows unrelated to landing, onboarding, or the chosen tour scopes.


## Proposal (high level)

### 1. Dashboard landing

- New default route after authentication: **dashboard** instead of application inventory.
- Widgets, links, and empty states: **to be specified by UXD**.

**Design placeholder** — add `images/dashboard-wireframe-placeholder.png` when UXD provides the asset. Inline preview:

```markdown
![Dashboard wireframe](images/dashboard-wireframe-placeholder.png?raw=true)
```

### 2. Quick Starts

- Integrate [PatternFly Quick Starts](https://www.patternfly.org/extensions/quick-starts/) (or supported PF6 extension).
- Two starts, prioritized by analytics or product input; copy and steps from UXD / prior doc work linked from [enhancements#202](https://github.com/konveyor/enhancements/issues/202).

**Design placeholder** — add `images/quick-starts-catalog-placeholder.png` when ready:

```markdown
![Quick Start catalog](images/quick-starts-catalog-placeholder.png?raw=true)
```

### 3. Guided tours

- Multi-step tours across pages (library TBD: e.g. PatternFly-aligned tour pattern or product choice).
- Cover “golden path” patterns agreed with UXD.

**Design placeholder** — add `images/guided-tour-placeholder.png` when ready:

```markdown
![Guided tour step](images/guided-tour-placeholder.png?raw=true)
```


## Prior references (external)

From [enhancements#202](https://github.com/konveyor/enhancements/issues/202) (may be revised for PF6):

- Quick Start copy and structure (Google Doc linked from that issue).
- Figma prototype linked from that issue.
- [tackle2-ui#1932](https://github.com/konveyor/tackle2-ui/issues/1932) tracks PF Quickstarts implementation work in the UI repo.


## Open Questions

- Exact **dashboard** modules and metrics (requires UXD + product).
- Which two **Quick Starts** ship in v0.10.0 vs later.
- **Tour** tooling vs custom implementation; accessibility requirements for tours.
- Relationship to **OCP alignment** and any separate enhancement work.


## Design deliverables (checklist)

Update as files land in `images/`:

- [ ] `images/dashboard-wireframe-placeholder.png` — replace with final dashboard wireframe or mockup
- [ ] `images/quick-starts-catalog-placeholder.png` — replace with Quick Start entry points mockup
- [ ] `images/guided-tour-placeholder.png` — replace with tour step example


## Test plan (sketch)

- E2E: land on dashboard when auth enabled; deep links still work.
- E2E: complete each Quick Start happy path; resume / dismiss behavior.
- E2E or integration: tour start, skip, and completion without breaking navigation.
- Accessibility: keyboard and screen reader for dashboard, Quick Start drawer, and tour focus management.


## Implementation History

| Date | Note |
| ---- | ---- |
| TBD | Link tackle2-ui PRs when implementation starts post-PF6. |

Tracking: [enhancements#269](https://github.com/konveyor/enhancements/issues/269).


## Upstream contribution

When opening the PR in [konveyor/enhancements](https://github.com/konveyor/enhancements), use path:

`enhancements/ui-uxd-redesign/README.md`

Commit UXD exports into `enhancements/ui-uxd-redesign/images/` and reference them from this README (use `?raw=true` on GitHub if needed for inline display).
