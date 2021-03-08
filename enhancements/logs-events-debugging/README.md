---
title: logs-events-debugging
authors:
  - "@djwhatle"
reviewers:
  - "@jortel"
  - "@shawnhurley"
  - "@alaypatel07"
approvers:
  - "@jortel"
  - "@shawnhurley"
  - "@alaypatel07"
creation-date: 2021-03-08
last-updated: 2021-03-08
status: provisional
see-also:
  - "/enhancements/debug/README.md"  
  - "/enhancements/dynamic-pipeline/README.md"  
  - "/enhancements/progress-reporting/README.md"  
replaces:
  - n/a
superseded-by:
  - n/a
---

# MTC Logging and Events improvements for Debugging (2.0)

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

This is where to call out areas of the design that require closure before deciding
to implement the design.  For instance, 
 > 1. Is it acceptable to modify our mig-controller codebase so that we _always_ have access to a logger with contextual key-value pairs attached? 
 > 2. Should we move away from having a pkg level logger object to enable concurrent reconciles without disturbing the per-reconcile logged UID?
 > 3. Should we carry a patch in the Velero codebase so that it would log a migration name or UID  we would pass in through an annotation? 

## Summary

At the time of this writing, MTC doesn't consistently give _enough_ __understandable feedback__ for its users (cluster admins) who:

 - May have a __migration stuck waiting__ on something, but it's __unclear what is being waited on__
 - Are accustomed to using __logs and events__ for __granular feedback__ on controller activities
 - Would like to __understand what the controller is thinking__ at any given point in the migration
 - Would like to __understand all changes being made to their clusters__, e.g. any API request modifying etcd store
 - Expect to be able to easily filter for __all logs for a particular migration__ across all system components 
 - Need __context__ with all log messages so that they are able to locate whatever that log message is related to across __clusters__, __namespaces__, and __kinds__.
 - Can't be expected to interpret or fix their own problem with a __stack trace__, can likely only __send it to us__
 - May want to bump verbosity to get __lower-level log messages__ e.g. from controller-runtime if they are seeing network-level issues etc.
 - Don't necessarily know __how to recover__ from a problem with the system __even if__ they discovered what the problem was
 - Don't know the set of CLI commands required to __traverse our hierarchical API__ relationships looking for feedback on migration progress
 - Don't know __which Pod or CR to look at__  for an error message for the problem they are encountering
 - Don't know __which cluster to investigate on__, since migration operations span up to 3 (host, src, target) clusters per migration

## Motivation

Due to the large _matrix of cluster configurations, network conditions, and cluster activity_, it's unlikely that a users first migration attempt will succeed 100% of the time.

We should work to __enhance discoverability__ by giving our users feedback in the places they would naturally expect to find it. When a problem occurs, we should provide clues on where to look next. 

Cluster admins should be capable to __navigate our system with OpenShift knowledge they already have__.

### Goals

How will we know that this has succeeded?

- When a problem occurs, and the user looks at (logs, conditions, events, UI), they should naturally be able to discover enough context to dig further into the problem.
- It will be clear at all times what MTC (especially mig-controller) is "thinking" and  doing, and what is preventing further progress.
- The user has a view into the API of the system at that exposes its natural structure, and quickly pinpoints where current activity is happening and where warnings/errors exist.
- Additional logs and events reveal the internal decision making processes of mig-controller and other componenets (velero plugins, maybe others).


### Non-Goals

- If feasible, avoid complete re-architecture of how we pass errors around in mig-controller.
- Do not provide logs or events on things that will never be useful in a troubleshooting situation

# Below is work in progress...

## Proposal

This is where we get down to the nitty gritty of what the proposal actually is.

#### Story 1

#### Story 2

### Implementation Details/Notes/Constraints [optional]

What are the caveats to the implementation? What are some important details that
didn't come across above. Go in to as much detail as necessary here. This might
be a good place to talk about core concepts and how they relate.

### Risks and Mitigations

What are the risks of this proposal and how do we mitigate. Think broadly. For
example, consider both security and how this will impact the larger OKD
ecosystem.

How will security be reviewed and by whom? How will UX be reviewed and by whom?

Consider including folks that also work outside your immediate sub-project.

## Design Details

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to...
  -  keep previous behavior?
  - make use of the enhancement?

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.
