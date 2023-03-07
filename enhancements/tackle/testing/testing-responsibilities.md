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

Testing of Konveyor is critical to keep its quality, release stability and happyness of community. In order to accomplish that, it is needed to create, execute and maintain several kinds of tests like end-to-end and components-specific test suites.

This enhancement should provide an introduction to Konveyor project testing strategy and define basic responsibilities for community members including developers as well as QE.

## Motivation

As we build an upstream community project that consists of multiple components, it is needed to ensure quality and stability of the project. One of important parts to achieve this is to have a test suite.

There are multiple level of tests like unit, integration or end-to-end, those could be executed on several levels from a single component unit tests to the Konveyor end-to-end UI test suite executed by automated CI.

The motivation for this enhancement is to do it in the most effective and engineers-friendly way.

### Goals

The main goal is to have a consistent test suite covering most of Konveyor application features that can be executed automatically and together with component tests, results will be reported to Konveyor CI.

An important part of this effort is to set expectations and basic responsibilities for Konveyor upstream community members regarding to test creation, execution and maintenance.

### Non-Goals

Cover downstream testing.

## Proposal





### Implementation Details/Notes/Constraints [optional]

technologies

What are the caveats to the implementation? What are some important details that
didn't come across above. Go in to as much detail as necessary here. This might
be a good place to talk about core concepts and how they relate.

### Security, Risks, and Mitigations

Upstream test suite must not contain any company internal information not customer data (not even as sample data).

## Design Details

### Test Plan

Open Konveyor upstream CI at https://github.com/konveyor/ci and see its status (green hopefully).

### Upgrade / Downgrade Strategy

Tests are dependent on Konveyor application, so might need to be branched/tagged and executed together and specifically for given Konveyor version.

No upgrade/downgrade actions for upstream test suite are expected.

## Implementation History

This is a follow-up on Dylan's Testing overview enhancement https://github.com/konveyor/enhancements/pull/98. 

## Drawbacks and Alternatives

Since testing could be considered as QE's responsibility, developers and Konveyor component maintainers don't need to care, so we might:
- not run upstream tests, leave it for downstream product builders OR
- not formalize upstream testing responsibilities too much to not make it over-engineered. 

## Infrastructure Needed

Github actions with Minikube should be enough for upstream tests executions.
