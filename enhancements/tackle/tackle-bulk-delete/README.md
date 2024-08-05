---
title: tackle-bulk-delete
authors:
  - "rromannissen"
reviewers:
  - "jortel"
approvers:
  - "jortel"
creation-date: 2022-06-14
last-updated: 2022-06-14
status: provisional
see-also:
  -  
replaces:
  -
superseded-by:
  -
---

# Tackle Hub: Bulk delete of applications


## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created


## Summary

This enhancement aims to provide bulk deletion of applications using the Application Inventory interface.

## Motivation

Tackle Hub allows importing applications at scale using CSV files. If for some reason the import was incorrect and required the deletion of all or some of the imported applications, that would require doing this deletion on a per application basis, thus leading to a very cumbersome process. It feels reasonable that if bulk creation of applications through the import is allowed, there should also be a way to bulk delete applications in case of error or any other reason.

### Goals

- Enable users to delete applications at bulk.

### Non-Goals

- Other bulk operations aside from deleting applications.

## Proposal

### Personas / Actors

#### Administrator

The administrator for the tool that has access to some application-wide configuration parameters that other users can consume but not change or browse. Administrator have full control of the application portfolio, including importing, creating, modifying and deleting applications, one by one or at bulk.

#### Architect

A technical lead for the migration project that can import, create, modify and delete applications, one by one or at bulk.

#### Migrator

A developer that should be allowed to run analysis, but not to assess, create, modify or delete applications in the portfolio.

### User Stories

#### Bulk Delete

##### BD001

*As an Administrator or Architect I want to be able to delete applications at bulk*

##### BD002

*As an Administrator or Architect I want to have manual confirmation when executing a bulk delete of applications*

##### BD002

*As an Administrator or Architect I want to get a notification when the bulk delete has been completed*

### Functional Specification

#### Bulk Delete

##### Related Use Cases

- [BD001](#BD001)
- [BD002](#BD002)
- [BD003](#BD003)


##### Involved Personas

- [Administrator](#administrator)
- [Architect](#architect)

##### Description

The bulk delete process will start by selecting one or many applications in the application inventory. This selection would naturally happen after applying some kind of filtering to the inventory, although that is not an actual requirement. Selecting the applications would enable the "Delete" option in the top Kebab menu:

![Bulk Delete](images/bd-select.png?raw=true "Bulk Delete")

Once the user clicks on the "Delete" option, a confirmation screen will be displayed:

![Bulk Delete](images/bd-confirm.png?raw=true "Bulk Delete")

Clicking on the "Delete" button will launch a synchronous process to delete the selected applications. This means the user will stay in this screen until applications have been deleted. Once the process is done, the user will navigate back to a reloaded view of the application inventory, displaying a notification that indicates the number of applications that were deleted:

![Bulk Delete](images/bd-notification.png?raw=true "Bulk Delete")


### Implementation Details/Notes/Constraints [optional]

- The bulk delete applications should be synchronous.

### Security, Risks, and Mitigations

None

## Design Details

### Test Plan

TBD

### Upgrade / Downgrade Strategy

Seamless upgrade through the Tackle operator

## Implementation History

TBD

## Drawbacks

None

## Alternatives

None
