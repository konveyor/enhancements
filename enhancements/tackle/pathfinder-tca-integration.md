---
title: pathfinder-tca-integration
authors:
  - "@rofrano"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2022-03-01
last-updated: 2022-03-01
status: provisional
see-also:
  - ""  
replaces:
  - ""
superseded-by:
  - ""
---

# Pathfinder / Tackle Container Advisor Integration

This is the title of the enhancement. Keep it simple and descriptive. A good
title can help communicate what the enhancement is and should be considered as
part of any review.

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

None  

## Summary

[Tackle Pathfinder](https://github.com/konveyor/tackle-pathfinder) is a tool for determining what path to take to modernize workloads on Kubernetes. [Tackle Container Advisor](https://github.com/konveyor/tackle-container-advisor) (TCA) is a tool to help standardize technology descriptions and determine if a workload can be run in a container, and if so, what container should be used.

This enhancement proposes to have Pathfinder integrate with Tackle Container Advisor for two purposes:

1. The technology standardization of TCA can be used to take application descriptions and create consistent tags in Pathfinder

2. The output from TCA can be additional input to the report that Pathfinder generates to determine the disposition of a workload in the cloud.

## Motivation

Creating tags and tagging applications can be time consuming and error prone. When tagging a few application it's not a lot of work but when tagging 1000's of applications for an enterprise migration to Kubernetes it can be a daunting task. Many times application descriptions are vague and use inaccurate terms like "rhel, db2, java ee, tomcat". TCA has the ability to turn that string into "RedHat Enterprise Linux, IBM DB2, Java J2EE, Apache Tomcat". These more standardized terms can be used to create tags and then tag applications consistently throughout pathfinder.

A prerequisite to deploying applications on Kubernetes is knowing if the application can be containerized. This is critical input to the pathfinder report yet it is missing. TCA can provide an evaluation to determine if the technologies used by an application can be run in a container and which container to suggest be used when deploying the application. This will help the users of pathfinder determine which workload disposition (i.e., retire, retain, rehost, replatform, refactor, rewrite) to use.

### Goals

- Enhance the capabilities of Tackle Pathfinder with the standardization technology from Tackle Container Advisor for the purpose of generating tags and tagging applications consistently.
- Enhance the reporting capabilities of Tackle Pathfinder with the container advisory technology from Tackle Container Advisor to recommend which applications can be containerized and which container images to use.

### Non-Goals

- Embedding technologies. This may be the first use case for Tackle having a set of callable services running in Kubernetes that can be called by other tools.

## Proposal

This is where we get down to the nitty gritty of what the proposal actually is.

### Personas / Actors

#### Migration Architect

The Migration Architect is responsible for determining teh disposition of each application with regard to whether it should be retired, retained, rehosted, replatformed, refactored, rewritten.

### User Stories

#### Story 1

**As a** Migration Architect  
**I need** to be able to easily tag applications with their technologies  
**So that** I can categorize applications for migration based on the technologies that they use  

#### Story 2

**As a** Migration Architect  
**I need** to be able to understand which applications can be containerized  
**So that** I can determine the feasibility of deploying them onto Kubernetes

### Implementation Details/Notes/Constraints [optional]

What are the caveats to the implementation? What are some important details that
didn't come across above. Go in to as much detail as necessary here. This might
be a good place to talk about core concepts and how they relate.

### Security, Risks, and Mitigations

**Carefully think through the security implications for this change**

What are the risks of this proposal and how do we mitigate. Think broadly. How
will this impact the broader OKD ecosystem? Does this work in a managed services
environment that has many tenants?

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
