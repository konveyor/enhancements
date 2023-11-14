---
title: testing-responsibilities
authors:
  - "@aufi"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: "2023-03-07"
last-updated: "2023-03-07"
status: provisional|implementable|implemented|deferred|rejected|withdrawn|replaced
see-also:
  - "/enhancements/testing/high-level-summary.md"
  - https://github.com/konveyor/enhancements/pull/98
---

# Konveyor testing responsibilities

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

Testing of Konveyor is critical to keep its quality, release stability and happiness of community. In order to accomplish that, it is needed to create, execute and maintain several kinds of tests like application level end-to-end and components-specific test suites.

This enhancement should provide an introduction to Konveyor project testing strategy and define basic responsibilities for community members including developers as well as QE.

## Motivation

As we build an upstream community project that consists of multiple components, it is needed to ensure quality and stability of the project. One of important parts to achieve this is to have a test suite.

There are multiple level of tests like unit, integration or end-to-end, those could be executed on several levels from a single component unit tests to the Konveyor end-to-end UI test suite executed by automated CI.

The motivation for this enhancement is to do it in the most effective, user-focused and engineers-friendly way.

### Goals

The main goal is to have a consistent test suite covering most of Konveyor application features that can be executed automatically and together with component tests, results will be reported to Konveyor CI.

An important part of this effort is to set expectations and basic responsibilities for Konveyor upstream community members regarding to test creation, execution and maintenance.

### Non-Goals

Cover downstream testing.

## Proposal

Let's start with kinds of tests relevant for us:
- Component/repository specific **unit tests**
  - Tests **logic** of a single Konveyor component (hub, UI, …), doesn’t require running whole Konveyor installation or other components (like Keycloak SSO) to be executed, but either doesn’t need it or mocking it.
  - Test technology/framework depends on technology used in a given component.
  - Unit tests are optional and if existed should be placed in the component repository.

- Component/repository specific **integration tests**
  - Tests **logic** of features from a single Konveyor component (hub, UI, …), but requires other Konveyor components to be running in order to pass the test (e.g. testing addon via requests to Hub to trigger analysis).
  - Tests should be placed in the component repository. Even the test might require other Konveyor components to test given component (e.g. addon might execute integration test sending request to Hub API which triggers the tested addon action, so it is practically an end-to-end test), the main point here is to test the component, other required Konveyor components are considered to be just dependencies required to run the test.
  - As these tests focus on basic feature logic, assertions like expected HTTP response code, only partial check of returned data, etc. might be acceptable here.

- Konveyor **application level end-to-end tests** (with UI or API)
  - Tests basic **use-cases** of the Konveyor application (scenarios which real users would expect to be working, ideally matching to Polarion test cases).
  - Requires running Konveyor installation and covers functions of multiple components.
  - Test scenarios should follow expected usage of the Konveyor by its users.
  - Assertions in tests require higher certainity that application works from user point of view than integration tests. E.g. Application creation test might look on HTTP response code too, but must try to retrieve the created application by its returned ID as a check if it was really created.

These tests should be executed and maintained as described in following section.

### Why such structure

Component integration tests and Application level E2E (API) tests are technically nearly the same, but they serves to slightly different purposes. At some point, PRs with new feature or fix should contain a test, the test written by the PR author should prove it basically works. There might be multiple PRs across Konveyor components for given feature, each component might test just it's part (focusing if it works, don't have to care too much about setting up other data to real use-case scenario).

So, even Hub might have integration test using an addon, it might not care about all different options of using the feature (e.g. for an analysis, if the setup options like RWX, different kinds of identities, etc. might matter), but that's a stuff which developer's integration tests don't have to care much.

Once the feature backend/API work was (mostly) completed, QE comes to play writing tests for it. They might use similar/shared methods with integration tests, but the tests focus on building real user test flows with relevant test data matching to Polarion test steps (if possible).

An ideal workflow on developers&QE cooperation on a new feature work:

|Feature started ->|||||
|---|---|---|---|---|
|Backend dev|Push PRs to e.g.addon/analyzer including tests<br>Push PRs to Hub including tests||||
|UI dev||Make UI for the feature|||
|QE||Work on test steps&application E2E API test|Write UI tests<br>Run sanity checks||
|||||-> Feature ready for release process|

### Responsibilities

#### Overview matrix

| Kind of test | Primary Responsible | Presence | Executed on | Trigger (min.required) | Source code in |
|---|---|---|---|---|---|
| **unit** | Developers | optional | Component | PRs, push | Component repo |
| **integration** | Developers | required | Component+Konveyor | PRs, push | Component repo |
| **application E2E** | QE | required | Running Konveyor | PR, push, nightly | E2E test suite repos |

#### More specific matrix as a starting point for Konveyor Hub and E2E tests

| Kind of test | Responsible | Description | Tests code | Trigger (min.required) |
|---|---|---|---|---|
| **integration** Hub | Hub developers | REST API coverage tests, applications import, [more](https://github.com/konveyor/tackle2-hub/discussions/241) | https://github.com/konveyor/tackle2-hub/... | PR, push, nightly |
| **application E2E** API | QE&Developers | Golang API test suite (WIP), focusing on sanity checks | https://github.com/konveyor/go-konveyor-tests | PR, push, nightly |
| **application E2E** UI | QE&Developers | Existing QE-maintained UI test suite using cypress framework | https://github.com/konveyor/tackle-ui-tests | PR, push, nightly |

### What Konveyor org expects from its components
- Decide if unit tests are relevant for given component, if so, write it and maintain it.
- Be primarily responsible for maintaining component integration tests.
- Setup test actions on their components repositories and report it to Konveyor CI repo.
- For new implemented features, create E2E application level test (or work with QE on it) to ensure the feature is fully working and is ready for real-world usage.

### What Konveyor components should expect from Konveyor org
- Provide tools for automated Konveyor installation setup (locally as well as with github actions) ready for running their integration tests on PRs (use Konveyor CI repo as a starting point).
- Provide working&stable Konveyor builds (partially ensured by e2e tests).

### Execution and project status - CI

Konveyor CI repository is https://github.com/konveyor/ci

The CI repo doesn't execute test suites itself, but provides re-usable workflows that can orchestrate Konveyor build, execute relevant tests and display overall status.

Part of the CI are also "gate" jobs for PRs in component repositories that executes tests on Konveyor built with given PR changes.

The CI functionality should be provided for main as well as other supported release branches.

### Security, Risks, and Mitigations

Upstream test suite must not contain any company internal information not customer data (not even as a sample data).

## Design Details

### Test Plan

Open Konveyor upstream CI at https://github.com/konveyor/ci and tests statuses should be visible (green-ish hopefully).

### Upgrade / Downgrade Strategy

Tests are dependent on Konveyor application, so might need to be branched/tagged and executed together and specifically for given Konveyor version.

No upgrade/downgrade actions for upstream test suite are expected.

## Implementation History

This is a follow-up on Dylan's Testing overview enhancement https://github.com/konveyor/enhancements/pull/98. 

## Drawbacks and Alternatives

Since testing could be considered as QE's responsibility, developers and Konveyor component maintainers don't need to care, so we might:
- not run upstream tests, leave it for downstream product builders OR
- not formalize upstream testing responsibilities too much to not make it over-engineered, just put some integration tests to Hub.

### A minimal working alternative

As a minimal change to current state which would move us a step forward is: **Put integration (E2E) tests to Hub as a developers effort and start require tests on new features PRs.**

Other things like Dev/QE cooperation on tests, application level API tests, requirements on type of tests, etc. would not be implemented.

## Infrastructure Needed

Github actions with Minikube should be enough for upstream tests executions.
