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

At the time of this writing, mig-controller doesn't consistently give _enough_ understandable feedback for its users (cluster admins) who:
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


## Below is work in progress...

The YAML `title` should be lowercased and spaces/punctuation should be
replaced with `-`.

To get started with this template:
1. **Pick a domain.** Find the appropriate domain to discuss your enhancement.
1. **Make a copy of this template.** Copy this template into the directory for
   the domain.
1. **Fill out the "overview" sections.** This includes the Summary and
   Motivation sections. These should be easy and explain why the community
   should desire this enhancement.
1. **Create a PR.** Assign it to folks with expertise in that domain to help
   sponsor the process.
1. **Merge at each milestone.** Merge when the design is able to transition to a
   new status (provisional, implementable, implemented, etc.). View anything
   marked as `provisional` as an idea worth exploring in the future, but not
   accepted as ready to execute. Anything marked as `implementable` should
   clearly communicate how an enhancement is coded up and delivered. If an
   enhancement describes a new deployment topology or platform, include a
   logical description for the deployment, and how it handles the unique aspects
   of the platform. Aim for single topic PRs to keep discussions focused. If you
   disagree with what is already in a document, open a new PR with suggested
   changes.

The `Metadata` section above is intended to support the creation of tooling
around the enhancement process.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

This is where to call out areas of the design that require closure before deciding
to implement the design.  For instance, 
 > 1. This requires exposing previously private resources which contain sensitive
  information.  Can we do this? 

## Summary

The `Summary` section is incredibly important for producing high quality
user-focused documentation such as release notes or a development roadmap. It
should be possible to collect this information before implementation begins in
order to avoid requiring implementors to split their attention between writing
release notes and implementing the feature itself. 

A good summary is probably at least a paragraph in length.

## Motivation

This section is for explicitly listing the motivation, goals and non-goals of
this proposal. Describe why the change is important and the benefits to users.

### Goals

List the specific goals of the proposal. How will we know that this has succeeded?

### Non-Goals

What is out of scope for this proposal? Listing non-goals helps to focus discussion
and make progress.

## Proposal

This is where we get down to the nitty gritty of what the proposal actually is.

### User Stories [optional]

Detail the things that people will be able to do if this is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.

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
