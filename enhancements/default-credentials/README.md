---
title: default-credentials
authors:
  - "rromannissen"
reviewers:
  - "@dymurray"
  - "@jortel"
  - "@sjd78"
approvers:
  - "@dymurray"
  - "@jortel"
  - "@sjd78"
creation-date: 2025-05-22
last-updated: 2025-05-22
status: provisional
see-also:
  -    
replaces:
  -
superseded-by:
  -
---

# Default Credentials


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

- How can a credential that was defined as default be explicitly defined as the credential for certain applications?

## Summary

This enhancement aims at enabling Konveyor to manage default credentials to streamline the language and technology discovery operations when credentials haven't been explicitly associated with applications in the portfolio.


## Motivation

In a common enterprise scenario, application repositories are secured. That means that in a large application import, considering that credentials can't be included in the CSV file, the first language and technology discovery operations will fail until the user manually associates credentials with each application. This introduces a manual step in a process that should be as automated as possible to tackle large application portfolios effectively and start surfacing information as early as possible.

A different scenario that makes this more of a problem would be related to [application discovery](https://github.com/konveyor/enhancements/tree/master/enhancements/assets-generation#discovery), as it would be important to automate the insights collection as much as possible as soon as applications are discovered if the source platform contained information about the application repository.


### Goals

- Enable administrators in Konveyor to define default values for each credential type.

### Non-Goals

- Redefine or reimplement how credentials are managed in Konveyor.
- Add a new type of credentials.

## Proposal

### Personas / Actors

#### Administrator

The administrator for the tool that has access to some application-wide configuration parameters that other users can consume but not change or browse.


### User Stories

##### DC001

*As an Administrator, I want to be able to define default values for each credential type*

##### DC002

*As an Administrator, I want the default credential values to be associated with applications*


### Functional Specification

#### Credentials management view

Default credentials should follow the following rules:

- Default credentials will only be available for Source Control and Maven Settings file credential types, at least on this first iteration.
- There can only be one default credential per credential type.
- Once a credential is set as default, it will be implicitly associated to all applications that don't have a credential of that type explicitly associated.
- Changing the default credential for a certain credential type **should trigger discovery for all applications that don't have a credential of that type explicitly associated**.

In the credentials management view, the _Edit_ and _Delete_ buttons for each credential row on the table will be replaced with a kebab menu with the following items:

- _Edit_
- _Delete_
- _Set as default_ (only if the credential is not default already)
- _Remove default_ (only if the credential is default already)

Clicking on _Set as default_ will set the current credential as default, which will be signaled in the credential row with the label _Default_. If there was a previously defined default credential of that credential type, a confirmation modal will be displayed with the text _This will replace the credential $CREDENTIAL as the default credential for the $CREDENTIAL_TYPE credential type_ with options _Accept_ and _Cancel_, where $CREDENTIAL is the credential that was previously defined as default and $CREDENTIAL_TYPE is the credential type for both the new and the former default credential.

Once the new default credential type is set, discovery will be triggered for all applications that don't have a credential of that type explicitly associated.



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

TBD

## Alternatives

TBD

## Infrastructure Needed

TBD
