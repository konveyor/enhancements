---
title: archetype-target-platforms
authors:
  - "@sjd78"
reviewers:
  - "@shawn-hurley"
  - "@mansam"
  - "@djzager"
  - "@jortel"
  - "@rromannissen"
  - "@JustinXHale"
approvers:
  - "@shawn-hurley"
  - "@mansam"
  - "@ibolton336"
  - "@djzager"
  - "@jortel"
  - "@rromannissen"
creation-date: 2024-08-26
last-updated: 2024-08-26
status: provisional
see-also:
  - "/enhancements/assessment-module/README.md"
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Add target platforms to application archetypes

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

N/A


## Summary

Application archetypes will be enhanced to be able to store groups of migration
targets together as a __target platform__.  The __target platforms__ will only
exist on the archetype and are not intended to be first class entities.

Various tools that generate analysis tasks will be able to use the target platforms
from the application's archetype to inform what analysis targets should be applied
without requiring a user to select them individually.

This enhancement only calls for adding CRUD functionality to the archetype edit screen
for target platforms as groupings of known analysis targets.


## Motivation

Allowing a Migrator user to select specific migration targets for analysis is not
useful at scale.  At scale, for Konveyor to be useful, there needs to be a way for
Architect users to define a group of migration targets to be used together.

This will also allow the developer hub integration to present a bundle of analysis
targets to a user, allowing a simplification of analysis configuration.


### Goals

  - Allow an archetype to be configured with a set of target platforms that exist only
    for that archetype.

  - Allow multiple migration targets to be collected together under an archetype's
    target platform.


### Non-Goals

  - Changes to the analysis wizard or to the task groups that setup an analysis task.

  - Utilizing target platforms in the UI or HUB.


## Proposal

Full CRUD functionality for target platforms on archetypes will be available to Architect
users.  Migrator users will be able to view configured target platforms.


### User Stories

Detail the things that people will be able to do if this is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.


#### Story 1

As an __architect__, I want to manage target platforms defined on an archetype.


#### Story 2

As an __architect__, I want to manage the migration targets contained within an
archetype's target platform.


#### Story 3

As a __migrator__, I want to view an archetype's target platforms and what migration
targets are contained in each.


### Implementation Details/Notes/Constraints

Only definition of the target platforms on the archetype will be supported.  Consideration
of target platforms on the wider system will not be considered.

#### HUB

  - The archetype endpoint needs to be enhanced to record a set of target platforms
    per archetype.

  - Since the target platforms are simply attributes of an archetype, an explicit
    `/targetplatform` endpoint is not required.

  - Allowing editing of the target platforms via a PATCH request would help simplify
    the CRUD functions on the UI and avoid potential collisions with archetype
    changes that are not target platform related.

Potential partial structure on the archetype entity:
```json
{
  "id": 45,
  "name": "Archetype Sierra Delta",
  "targetPlatforms": [
    {
      "name": "Kubernetes",
      "targets": [
        "konveyor.io/target=cloud-readiness",
        "konveyor.io/target=openjdk21",
        "konveyor.io/target=eap8"
      ]
    },
    {
      "name": "Traditional Standalone",
      "targets": [
        "konveyor.io/target=openjdk21",
        "konveyor.io/target=eap8"
      ]
    }
  ]
}
```


#### UI

  - The existing archetype table and edit modals will remain.

  - An extra section on the edit modal will be added to support the target
    platform functions.

  - The archetype table could have a new column added to display a count of
    target platforms defined.

  - Since the target platforms are unique to each archetype, search filters looking
    inside the target platforms (name, target, etc) would be more complex to add.

  - The archetype details drawer can have a new tab added to display a summary of
    the archetype's target platforms and what migration targets they contain.


### Security, Risks, and Mitigations

Authorization checks to limit the migration target functions available to the user
is based on user persona only.


## Design Details


### Test Plan

In general, the API and UI testing would be simple CRUD testing against an archetype's
details.

_Additional details pending._


### Upgrade / Downgrade Strategy

As this will be a new feature, the upgrade strategy will be handled with a combination
of sql migrations and new container images.

The downgrade will follow a similar approach, the database will need to be rolled back,
we will lose all target platform definitions.


## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.


## Drawbacks

N/A


## Alternatives

Referring to "Target Platform" as a "Target Group" could be more generic and indicative
of how the configurations flow without relying on the extra semantics of "platform".
