---
title: neat-enhancement-idea
authors:
  - "@dymurray"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2023-02-08
last-updated: 2023-02-09
status: provisional|implementable|implemented|deferred|rejected|withdrawn|replaced
---

# Konveyor Testing Initiative

It is desired to add upstream CI testing to provide regression and integration
testing of our project. Building out and maintaining a robust test suite will
give us confidence on code submissions and released that the builds are stable
and we haven't introduced any regressions.


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

 > 1. What tools do we have to do upstream UI testing?
 > 2. Where should this test suite live?

## Summary

This enhancement defines the steps towards establishing a robust CI test suite
that is extensible and capable of providing gate testing for PRs as well as
release verification. The establishment of this test suite will enable
integration testing as well as component testing for all Konveyor components.

## Motivation

There are a number of reasons that this goal should be pursued, but the primary
motivation is to allow us to release more often with less regressions.
Establishing a test suite that can provides basic regression testing gives us
confidence when a release is cut that the release is stable.

Additionally, once this test suite is established developers can easily add
testing for features they develop providing verification of the feature prior
to merge and regression protection for that feature in the future.

### Goals

* Enable cross-team collaboration between engineering and QE
* Upstream-first test suite development
* Make it easy to test analyzers and add-ons in a common test suite
* Enable developers to easily add test cases for new features and bugs
* Test suite must be able to be run locally

### Non-Goals

* Deciding on frameworks to be used in the test suites

## Proposal

I propose we make this a phased approach. The first phase will be building upon
te existing work done by Shveta and Mayaan (see Alternatives section) to
establish a test suite which provides full API test coverage. I propose we
write this test suite in Golang so that all API contributors can provide tests
for features they contribute. As we build out the test suite for phase 1 we
should be focused on writing reusable test fixtures that can be used to write
integration tests in phase 2.

For phase 2, we will take the test fixtures written during phase 1 to write
more complex integration tests. These tests should include integration tests of
different components in Konveyor such as addons, analyzers, and their
interactions with the hub.

To begin phase 1, I will work to replicate the functionality of the Python test
suite in a Golang version using a [table-driven
testing](https://dave.cheney.net/2019/05/07/prefer-table-driven-tests)
approach. 

### User Stories [optional]

#### Story 1

As a developer I am extending an existing API endpoint and want to provide
additional tests to cover the change in functionality. I should be able to add
a basic test case to an existing test suite covering this API endpoint.

#### Story 2

As a developer I am adding a new API endpoint for a new feature I am
developing. I should be able to add a new test suite following a common
template/format to provide coverage of this API.

#### Story 3

As a developer I am adding a new feature to Konveyor. I want to provide an
associated integration test. I would like to extend an existing integration
test suite to support my new functionality.

#### Story 4

As a testing engineer I wish to add test coverage for a new or existing
feature. I should be able to add test cases or update existing test cases to
guarantee no regressions are introduced in the future.

### Implementation Details/Notes/Constraints [optional]

These tests should be written in Golang. Golang is the language of choice for
our project and we want to encourage all community memebers to contribute
tests. The best way to do this would be to keep the test suite in Go. The
framework to be used inside of the test suite doesn't need to be determined
ahead of time, but a framework such as Ginkgo might be worthwhile to explore
for integration testing.

The API test suite should take in a Tackle hub API URL and perform basic
testing of all endpoints and error out if any API tests fail.


## Design Details

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.


## Alternatives

1. [Konveyor API Tests in Python](https://github.com/konveyor/tackle-api-tests)
    * These tests are a great starting point to begin building a robust test suite that covers the API endpoints for the hub. Using this repo as a guide to begin building test fixtures feels like a good path forward, but we could choose to continue building this suite in Python.
2. [Konveyor API Tests in Venom](https://github.com/aufi/konveyor-ci-playground)
    * Equivalent functionality as python tests using a tool called [Venom](https://github.com/ovh/venom). Benefits to this approach is ease of use and human readable tests in YaML.

## Infrastructure Needed [optional]

In order to succeed, we need a robust CI system. This robust CI system needs to
capable of providing reliable infrastructure to run basic tests against. Right
now we have existing scripts that install Konveyor on Minikube.

These scripts should follow the unix philosophy so that we have small reusable
components that can act as building blocks for each Konveyor project to reuse
them for their own purpose. For example, the Windup addon repo may want to
reuse the konveyor installation script and a separate script that configures
Windup.
