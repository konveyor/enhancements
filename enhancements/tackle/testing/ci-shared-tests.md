---
title: ci-shared-tests
authors:
  - "@aufi"
reviewers:
  - @dymurray
approvers:
  - TBD
creation-date: "2025-07-11"
last-updated: "2025-07-11"
status: provisional|implementable|implemented|deferred|rejected|withdrawn|replaced
see-also:
  - "/enhancements/testing/high-level-summary.md"
  - https://github.com/konveyor/enhancements/pull/98
  - https://github.com/konveyor/enhancements/pull/103
---

# Konveyor CI shared test cases

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

Testing of Konveyor is critical to keep its quality, release stability and happiness of community. 

In order to provide consistency and stability across Konveyor components, we want implement shared test cases provided by CI repository and make main components use them. Primary focus are E2E API test suite and CLI, both with tier0 analysis tests.

Those shared tests should provide more stability of Konveyor project, faster identification of problems and easier debugging of failures.

## Motivation

There are several test suites focusing application analysis within Konveyor - API, UI and CLI. Those tests have different test cases, sometimes different results and are different in their execution schedule, failure reporting and being kept up-to-date.

We want make clear what is minimal analysis test suite and be able run it on all relevant components.

### Goals

The main goal is to have a consistent test suite covering Konveyor user-facing components that provide application analysis feature, so we're sure that primary API and CLI analyses provide the same and consistent results.

This test suite should have minimal relevant test cases amount, but should strictly require verified analysis results.

### Non-Goals

Touch unit tests or repository-specific test that don't focus on full analysis process.

## Proposal

#### 1. Create shared test cases in konveyor/ci repository

#### 2. Make go-konveyor-test tier0 use those shared test cases

#### 3. Make kantra-cli-tests tier0 use those shared test cases

#### 4. Decide and implement what UI (with static-report) should check in analysis

### Responsibilities

- Konveyor as project is responsible to create and maintain shared analysis test cases (applications w/dependencies if needed, analysis parameters and expected analysis results) in `konveyor/ci` repository.

- Dev&QE team focusing on given components (API, CLI, UI) are responsible to consume shared test cases from `konveyor/ci` repository, transform them to format relevant for their components, automatically execute those as part of tier0 tests and report failures to `konveyor/ci` repository README badges and to team Slack channel. 

### Security, Risks, and Mitigations

Upstream test suite must not contain any company internal information or customer data (not even as a samples).

## Design Details

### Test cases format

- Given different languages/frameworks used for testing different components (API golang, UI typescript cypress, CLI python pytest), the format need to be generic and easily transformable.

- The tests results might become pretty large, so it should consist more likely from multiple files instead of large one with all test cases.

A proposal is to use one directory per test-case with following content:   (Note, raw, might be updated)

```
└── shared_tests
    └── analysis_tackle_testapp_public_deps
        ├── dependencies.yaml   # analyzer-like dependencies output
        ├── output.yaml         # analyzer-like analysis output
        ├── tc.yaml             # analysis test parameters (targets, etc.)
        └── <input_app>         # TBD, test application linked from another repos or binaries here?
```

### Test Plan

Nightly execution of those tests with github actions is expected. Tests statuses should be visible on `konveyor/ci` README page and notification sent to team Slack channel.

### Upgrade / Downgrade Strategy

Tests are dependent on Konveyor application, so might need to be branched/tagged and executed together and specifically for given Konveyor version.

No upgrade/downgrade actions for upstream test suite are expected.

## Implementation History

This is a follow-up Testing overview enhancement https://github.com/konveyor/enhancements/pull/98, Testing responsibilities enhancement https://github.com/konveyor/enhancements/pull/103 and recent work on tier0 analysis tests in CLI test suite https://github.com/konveyor-ecosystem/kantra-cli-tests.

## Drawbacks and Alternatives

There might be a struggle with automated test cases conversion, if that becomes too hard, we might consider syncing test cases manually. But hopefully, this will not happen. 

### A minimal working alternative

We should have this working for API and CLI components first. UI part is secondary and subject of discussion.

## Infrastructure Needed

Github actions with Minikube should be enough for upstream tests executions.
