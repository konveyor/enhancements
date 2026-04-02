---
title: Migrate the Konveyor UI to PatternFly 6
authors:
  - "@sjd78"
reviewers:
  - "TBD"
approvers:
  - "TBD"
creation-date: 2026-04-02
last-updated: 2026-04-02
status: implementable
see-also:
  - "https://github.com/konveyor/enhancements/issues/270"
  - "https://github.com/konveyor/tackle2-ui/issues/2172"
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Migrate the Konveyor UI to PatternFly 6


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created


## Open Questions

- Exact sequencing versus other v0.10.0 UI initiatives (for example [modern OIDC](https://github.com/konveyor/enhancements/issues/268)) to avoid conflicting large refactors on the same release branch.
- Whether visual regression baselines (Percy or equivalent) should be expanded before or during the PF6 cutover.
- Secondary alignment with OpenShift console look-and-feel may be owned largely by [enhancements#269](https://github.com/konveyor/enhancements/issues/269); this document treats that as related but not blocking PF6 adoption.


## Summary

This enhancement implements [RFE issue #270](https://github.com/konveyor/enhancements/issues/270): move the Konveyor UI from PatternFly 5 to **PatternFly 6**, replace use of deprecated components, and clear related front-end technical debt so follow-on work (dashboards, [PatternFly quick starts](https://www.patternfly.org/patterns/quick-starts/), guided tours) can build on a current design system. Execution and day-to-day tracking live in [konveyor/tackle2-ui](https://github.com/konveyor/tackle2-ui), starting with epic [tackle2-ui#2172](https://github.com/konveyor/tackle2-ui/issues/2172).


## Related work in tackle2-ui

The table below links GitHub issues that are **in progress or directly support** the PF6 migration (as of the enhancement draft). Update this section as issues close or split.

| Issue | Role |
| ----- | ---- |
| [tackle2-ui#2172](https://github.com/konveyor/tackle2-ui/issues/2172) | **Parent tracking issue** — “Upgrade to PatternFly 6”; milestone v0.10.0; `triage/accepted`, `kind/dependency-change`. |
| [tackle2-ui#3042](https://github.com/konveyor/tackle2-ui/issues/3042) | **Preparation** — migrate components already deprecated under PF5 to supported PF5 APIs before the PF6 bump. |
| [tackle2-ui#3150](https://github.com/konveyor/tackle2-ui/issues/3150) | **PF6 upgrade** — dependency and application updates toward PatternFly 6 (including related tooling). |
| [tackle2-ui#1318](https://github.com/konveyor/tackle2-ui/issues/1318) | **Tables** — adopt `ActionsColumn` and related PF table patterns ahead of or during the migration. |
| [tackle2-ui#3148](https://github.com/konveyor/tackle2-ui/issues/3148) | **Tests** — stabilize Cypress/selectors (e.g. `ouiaId`) for components affected by PatternFly upgrades. |
| [tackle2-ui#1932](https://github.com/konveyor/tackle2-ui/issues/1932) | **Follow-on UX** — PF quick starts (enabled more easily once on PF6; not a prerequisite for the version bump itself). |

Additional PRs and issues will appear under the v0.10.0 milestone or dependabot “patternfly” groups; link them in **Implementation History** as they merge.


## Motivation

The UI currently depends on PatternFly 5 (`@patternfly/patternfly`, `@patternfly/react-core`, `@patternfly/react-table`, `@patternfly/react-tokens`, etc., at 5.x in [client/package.json](https://github.com/konveyor/tackle2-ui/blob/main/client/package.json)). PatternFly 6 is the active major line; staying on 5 accumulates deprecated APIs and diverges from current documentation, examples, and community extensions. [Issue #270](https://github.com/konveyor/enhancements/issues/270) also calls out **tech debt and library upgrades** that naturally bundle with a design-system migration.

### Goals

- Upgrade PatternFly packages and global styles to **PF6** with a coherent visual result (design refresh).
- Eliminate or replace **deprecated** PF5 component usage called out by PatternFly and by static analysis where applicable.
- Keep **accessibility** and **internationalization** behavior at least on par with PF5 (WCAG-oriented components, existing i18n keys).
- Unblock **future UX** called out in #270: dashboards, quick starts, guided tours on supported PF6 patterns.
- Coordinate **supporting dependency** updates (for example charts, code editor, tokens) on compatible major lines with PF6.

### Non-Goals

- Full **OpenShift console** visual parity in this enhancement alone (see [enhancements#269](https://github.com/konveyor/enhancements/issues/269) where that is tracked).
- Rewriting product UX flows solely for aesthetics; scope is migration and necessary API/CSS adjustments unless explicitly agreed.
- Backporting PF6 to older Konveyor release branches (forward-looking v0.10.0 work).


## Decisions

| Topic | Decision |
| ----- | -------- |
| Target design system | **PatternFly 6** per [patternfly.org](https://www.patternfly.org/) migration guidance; PF5 remains supported by PatternFly until PF7, but Konveyor targets PF6 for v0.10.0. |
| Primary implementation repo | **[konveyor/tackle2-ui](https://github.com/konveyor/tackle2-ui)**; this enhancement document lives in [konveyor/enhancements](https://github.com/konveyor/enhancements) under `enhancements/ui-patternfly-6/README.md` when contributed upstream. |
| Preparation phase | Address PF5 deprecations first where required by the PF6 migration path (see [tackle2-ui#3042](https://github.com/konveyor/tackle2-ui/issues/3042)). |
| Tracking | Single umbrella issue [tackle2-ui#2172](https://github.com/konveyor/tackle2-ui/issues/2172); link child issues and merged PRs in **Implementation History**. |
| Status | `implementable`; active work is already filed and triaged on tackle2-ui. |


## Proposal

### High-level approach

```mermaid
flowchart TB
  prep [PF5_deprecation_cleanup]
  deps [Bump_PatternFly_and_peers]
  app [Fix_breakages_styles_a11y]
  test [Unit_e2e_visual_regression]
  prep --> deps --> app --> test
```

1. **Inventory** — List `@patternfly/*` and dependent packages; run PatternFly migration notes against the codebase (deprecated imports, token renames, layout changes).
2. **Pre-migration cleanup** — Resolve PF5 deprecation warnings where they block or complicate PF6 (tracked under [tackle2-ui#3042](https://github.com/konveyor/tackle2-ui/issues/3042) and related PRs).
3. **Dependency upgrade** — Move `@patternfly/patternfly`, `@patternfly/react-core`, `@patternfly/react-table`, `@patternfly/react-tokens`, `@patternfly/react-charts`, `@patternfly/react-code-editor`, and aligned versions in one or more coordinated PRs ([tackle2-ui#3150](https://github.com/konveyor/tackle2-ui/issues/3150) and successors).
4. **Application fixes** — Update imports, props, CSS variables/tokens, layout wrappers, and custom styling that depended on PF5 internals.
5. **Verification** — Jest/RTL, Cypress E2E ([tackle2-ui#3148](https://github.com/konveyor/tackle2-ui/issues/3148) and suite updates), and optional visual regression; fix lint caps if PF6 surfaces new warnings.

### User-visible outcome

- Users see the **PF6 design refresh** (spacing, typography, components) without losing core workflows.
- Developers use **current PatternFly docs** and examples when adding UI.

### Implementation details / notes

| Area | Notes |
| ---- | ----- |
| Entry styles | Global PatternFly CSS imports and any webpack style pipeline ([client/config](https://github.com/konveyor/tackle2-ui/tree/main/client/config)). |
| Tokens | Widespread `@patternfly/react-tokens` usage; verify token renames and chart colors. |
| Tables | `react-table` + `ActionsColumn` patterns ([tackle2-ui#1318](https://github.com/konveyor/tackle2-ui/issues/1318)). |
| Third-party | `@monaco-editor/react`, charts, and other packages pinned with PatternFly in dependabot groups may need coordinated bumps. |
| Cypress | Selectors and OUIA attributes may need updates alongside component upgrades. |


## Design Details

### Test plan

- **Unit / component** — Existing RTL tests updated for changed props/DOM; add tests where PF6 changes behavior (menus, modals, pagination).
- **E2E** — Full Cypress suite green; prioritize flows touching tables, wizards, and forms ([tackle2-ui#3148](https://github.com/konveyor/tackle2-ui/issues/3148)).
- **Visual** — Manual pass on primary pages; use project visual regression tooling if present; document known intentional visual deltas.
- **Accessibility** — Spot-check with keyboard and screen reader on representative pages after the upgrade.

### Risks

- Large diff surface → merge conflicts with parallel UI features; prefer short-lived branches or stacked PRs tied to [tackle2-ui#2172](https://github.com/konveyor/tackle2-ui/issues/2172).
- Subtle **CSS** regressions in custom layouts.
- **Bundle size** or **performance** shifts from PF6; monitor if the project tracks bundle metrics.


## Implementation History

| Date | PR / issue |
| ---- | ---------- |
| TBD | Link merged tackle2-ui PRs here (for example work under [tackle2-ui#3150](https://github.com/konveyor/tackle2-ui/issues/3150)). |

Umbrella: [tackle2-ui#2172](https://github.com/konveyor/tackle2-ui/issues/2172). Product RFE: [enhancements#270](https://github.com/konveyor/enhancements/issues/270).


## Drawbacks

- Significant review load and churn across the client tree.
- Designers and contributors must relearn moved or renamed APIs versus PF5 habits.


## Alternatives

- **Stay on PF5** — Rejected for #270; increases deprecation debt and blocks PF6-only ecosystem features.
- **Big-bang single PR** — Possible but risky; phased PRs with a running main branch are preferred unless the team agrees otherwise.


## Upstream contribution

When opening the pull request in [konveyor/enhancements](https://github.com/konveyor/enhancements), place this document at:

`enhancements/ui-patternfly-6/README.md`

PR description should reference:

- [enhancements#270](https://github.com/konveyor/enhancements/issues/270)
- [tackle2-ui#2172](https://github.com/konveyor/tackle2-ui/issues/2172) and any active child issues from the **Related work in tackle2-ui** table

Optional: screenshots of key screens before/after PF6, and a short list of breaking PatternFly API changes encountered in tackle2-ui.
