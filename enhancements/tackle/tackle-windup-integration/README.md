---
title: tackle-windup-integration
authors:
  - "rromannissen"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2021-11-19
last-updated: 2021-11-22
status: provisional
see-also:
  -   
replaces:
  -
superseded-by:
  -
---

# Tackle Analysis: Tackle Hub integration with Windup


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

- **What would be the integration mechanism between the Application Inventory/Tackle
Hub and Windup?**
  - *Tackle Analysis API*
    - Tackle Analysis is deployed as a service by the Tackle Operator.
    - The analysis is triggered by Tackle Hub via the Tackle Analysis API.
    - Only the reference to the analysis is stored in Tackle Hub.
    - The UI fetches the data from Tackle Analysis API and renders it.
  - *Windup CLI wrapper*
    - Wrap the windup command with a thin API layer that runs the windup process.
    - Add-on monitor and reflect its status in Tackle Hub task.
    - Expose the resulting HTML report to be uploaded to Tackle Hub. The wrapper is
    running in a pod that is deleted when complete. The HTML report can be displayed
    by the UI directly from Tackle Hub.


## Summary

Application analysis is one of the core use cases for the Tackle toolkit. This is
expected to be achieved by leveraging the [Windup](https://github.com/windup/windup)
project, developed by Red Hat as the upstream for the
[Migration Toolkit for Applications](https://developers.redhat.com/products/mta/overview)
product. The expected outcome of this enhancement is to have a seamless integration
of Windup within the Application Inventory user experience in the same fashion as
we currently have with the Pathfinder tool.


## Motivation

Application analysis at scale provides insight for the adoption leads to make
informed decisions and guidance for migrators about the adaptations that might
be required on applications. This helps reducing risks and making the migration
and modernization process measurable and predictable.

### Goals

- Bring application analysis capabilities into the Tackle project.
- Establish the Application Inventory as the natural integration point for all
Tackle projects.
- Have a seamless user experience when executing application analyses from the
Application Inventory.
- Provide a scalable solution that can run properly on full fledged Kubernetes
clusters or a migrator's laptop.

### Non-Goals

- Design or implement Dynamic Reports, as this is expected on later iterations.
- Integration of the Application Inventory with Git, SVN and Maven, as this is
expected to be available by the time this feature starts implementation.

## Proposal

### Personas / Actors

- **Architect**: A technical lead for the migration project that can create and
modify applications and information related to it. The Architects don’t need to
have access to sensitive information, but can consume it.
- **Migrator**: A developer that should be allowed to run assessments and analysis,
but not to create or modify applications in the portfolio.


### User Stories

#### Analysis Configuration

##### AC001

*As an Architect/Migrator I want to be able to define the source for application analysis:
source code or binary.*

##### AC002

*As an Architect I want to be able to configure if Maven dependencies are analyzed as binaries
 when using source code as source for application analysis.*

##### AC003

*As an Architect/Migrator I want to be able to use Git repositories or as input
for an analysis.*

##### AC004

*As an Architect/Migrator I want to be able to upload binaries from my workstation
as input for an analysis.*

##### AC005

*As an Architect/Migrator I want to be able to select the migration target for
my application.*

##### AC006

*As an Architect/Migrator I want to be able to specify the packages to be analyzed.
If no packages are specified, then all packages identified as 'application packages' will be analyzed.*

##### AC007

*As an Architect/Migrator I want to be able to select the migration source for
my application.*

##### AC008

*As an Architect/Migrator I want to be able to include or exclude rules tags to
explicitly define which rules get executed during the analysis*

##### AC009

*As an Architect/Migrator I want to be able to provide custom rules for the analysis.*

#### Analysis Output

##### AO001

*As a Architect/Migrator I want to get access to the result of the analysis (analysis report)
of each individual application.*

##### AO002

*As a Architect/Migrator I want to be able to track the specific version of the source code
that has been analyzed.*

##### AO003

*As a Architect/Migrator I want to see metadata about the history of the analyses at a
summary level in terms of number of issues, story points, analysis configuration
(source, target and options), for each analysis execution*


### Implementation Details/Notes/Constraints

TBD

### Security, Risks, and Mitigations

TBD

## Design Details

### Test Plan

TBD

### Upgrade / Downgrade Strategy

TBD

## Implementation History

TBD

## Drawbacks

- Due to storage constraints, only one analysis report per application will be
retained on the central graph database, although metadata about each analysis
should be maintained on Tackle Hub.

## Alternatives

TBD

## Infrastructure Needed

TBD
