---
title: must-gather-from-ui
authors:
  - "@aufi" / maufart
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2021-07-16
last-updated: 2021-07-16
status: provisional
see-also:
  - "/enhancements/must-gather-rest-service/README.md"
  - "https://bugzilla.redhat.com/show_bug.cgi?id=1944402"
---

# Must-gather from UI

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

It should be possible to use OpenShift must-gather feature directly from Forklift migration UI.

## Motivation

When a VM migration fails, Forklift user needs quick access to migration debugging informations and other details provided by must-gather feature. It should help to identify where is a problem and to make a successful migration once the problem is resolved.

To make it easy for user, it should be supported to trigger must-gather directly from UI and don't have to switch to CLI.

### Goals

User should be able to execute must-gather from Forklift UI and get archive with results.

### Non-Goals

This enhancement doesn't cover backend or must-gather execution itself, just provide features of must-gather-service enhancement in UI.

## Proposal

The Forklift UI should allow user to execute OpenShift must-gather and provide the result archive to user.

The must-gather should be possible to execute for
- Plan
- VM

A Command parameter of must-gather to specify targets for gathering (e.g. ```PLAN=plan1 /usr/bin/targeted```) will be prepared on UI side into a string passed to must-gather-service API call.

### Risks and Mitigations

The UI needs to be intuitive for user, will be discussed with UX team.

## Design Details

### Test Plan

After a VM migration from VMware or RHV to OpenShift Virtualization, click on button TBD and wait until an archive is downloaded. It should not take more than 5 minutes. The archive should contain following directory structure:

- TBD

## Drawbacks

There is existing feature in OpenShift command line, so user could use CLI.

## Alternatives

User could switch to command line and do the same with existing set of features. In some projects (Crane) there is quite detailed way to get details on Migration directly in UI.