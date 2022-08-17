---
title: automated integration tests for forklift
authors:
  - "@rokkbert"
reviewers:
  - TBD
  - TBD
approvers:
  - TBD
  - TBD
creation-date: 2022-08-17
last-updated: 2022-08-17
status: provisional
see-also:
  -   
replaces:
  - 
superseded-by:
  - 
---

# Automated Integration Tests for Forklift

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

See below.

## Summary

This enhancement proposes to create processes, infrastructure and code for automated
integration tests. These tests should run unsupervised and test all the steps from
deployment of forklift, over the creation of providers, mappings and migration plans,
to the actual migration of virtual machines.

## Motivation

Having such tests, which take considerable effort to execute manually, would help
to guard against regressions when changes are made or new features are implemented.
They can also be extended as needed to verify specific bug fixes.

They can serve the secondary purpose of documenting what the expected behaviour of
the system is.

### Goals

The goal is to have an automated job that builds or collects all forklift components
and then:
* deploys them to a cluster (e.g. [minikube](https://minikube.sigs.k8s.io/docs/)
or [CRC](https://crc.dev/crc/))
* creates (& modifies/deletes) providers for VMware and Red Hat Virtualization
* creates (& modifies/deletes) appropriate network- and storage mappings
* creates (& modifies/deletes) migration plans
* migrates VMs
* verifies that the migration succeeded

Most of these can be done with command line tools but it would be good if the
basic workflow would also be tested through the web-GUI using something like
[cypress](https://www.cypress.io/) or [selenium](https://www.selenium.dev/).

To meet the goal we need:
* ressources and processes to provide the infrastructure
  * kubernetes cluster to run forklift
  * kubernetes cluster with kubevirt as migration target
  * vSphere, with VMs
  * oVirt, with VMs
  * platform that executes the test, which might or might not be possible on github
* test scripts that go through the aforementioned steps
  * should ideally be easy to configure so that anybody can run them on other systems too

### Non-Goals

Unit tests are not covered by this enhancement because they are easy and quick
to run by developers by themselves and don't need complex infrastructure.
Only end-to-end tests are considered here.

## Proposal

### User Stories

#### Test Release

Before a release version is tagged the tests are triggered by the package maintainer
to check for any problems that might have been overlooked.

#### Test Merge

If it's fast and cheap enough the tests can automatically run after a PR was merged.

#### Local Tests

A developer or quality engineer can utilize the test scripts to run the same tests
on their own infrastructure to verify any work in progress version they are interested
in.

### Implementation Details/Notes/Constraints

#### Open Questions

* Can the tests run on GitHub?
  * On a GitHub-hosted runner or somewhere else?
* What is the best k8s-variant to test on?
* Can/should the k8s-cluster[s] be instantiated dynamically by the
test or be persistent somewhere?
* What tool/framework should be used to write the tests and the setup scripts?
* Where can vSphere run?
  * How much space does it need?
  * License?
  * Access control?
* Where can oVirt run?
  * How much space does it need?
  * Access control?
* Which GUI-test tool should be used?
* How fast can/should the tests be?
* When do the tests run, manually, automatically, periodically?
* Where do the forklift packages come from? Are they specified when running
the tests or are the tests part of the build?

Details about what VMs should be migrated and what configurations should be used
are out of scope for this proposal. That will already be part of the "doing" and
should not produce any critical questions.

### Security, Risks, and Mitigations

If any infrastructure is used which persists outside of the test execution (i.e.
anything that is not set up before and destroyed after the test) then the access
to that needs to be protected somehow to prevent denial of service issues, data
manipulation as well as potentially big cloud service fees.
Maybe this prevents totally autonomous test runs because it might require a person
to enter credentials, which can not be stored alongside the tests.

## Design Details

### Test Plan

### Upgrade / Downgrade Strategy

Dependencies regarding versions and configuration of external systems, as long as they
are not created by the tests themselves, need to be considered and documented and, if
possible, verified by the tests before running the actual tests.

## Implementation History

We can start with a small set of tests and add more later to cover more options.

## Drawbacks

Compared to unit-tests, which can execute in a free github-runner (see
[Billing for GitHub Actions](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions)),
these e2e-tests will be very resource intensive. The will need much more CPU-time
and memory and thus be expensive, depending on where they run. The time and trouble
that is saved by having them needs to be more valuable than the cost of these resources.

For VMware licensing costs need to be considered.

Tests, especially GUI-tests, always add some amount of maintenance work on top, 
otherwise they quickly become obsolete again.

## Alternatives

Instead of having a central process for such tests anybody who wants can do testing
on their own systems, either manually or with their own scripts.

If the questions regarding the required infrastructure turn out to not have good
answers it could still be useful to have an agreed upon set of test scripts and data
sets that verify the behaviour of forklift.

## Infrastructure Needed

* At least one kubernetes cluster on which to deploy/run forklift
* Optional second kubernetes cluster for testing three-way-migration where the target
it not the same as where forklift runs
* VMware vSphere with 1+ source VMs (the more the better, to test different cases)
* RHEV/oVirt with 1+ source VMs (the more the better, to test different cases)
* GitHub-runner that executes the setup and the tests

The k8s-clusters could be created dynamically by the test and don't need to exist all
the time. That would also ensure reproducability. On the other hand, clean un-/re-deployment
of forklift should be tested anyway.
The source clusters would be more difficult to create from scratch each time, especially
because they contain a lot of data in the form of the virtual machines. In theory these
could also be downloaded dynamically but it sounds more troublesome.
