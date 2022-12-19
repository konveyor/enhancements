---
title: migration-waves
authors:
  - "rromannissen"
reviewers:
  - "@jortel"
  - "@mansam"
  - "@jwmatthews"
approvers:
  - "jortel"
creation-date: 2022-12-16
last-updated: 2022-12-16
status: provisional
see-also:
  - "[tackle-jira-integration](https://github.com/konveyor/enhancements/pull/85)"   
replaces:
  -
superseded-by:
  -
---

# Migration Waves


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

- What happens with the "*Status*" compound expandable section of the Migration Waves view if no issue tracker is enabled?(See [tackle-jira-integration](https://github.com/konveyor/enhancements/pull/85))
- What format will be used to load stakeholders that don't exist in the CSV import?

## Summary

The Konveyor project aims at providing differential value at each stage of the migration process. So far, most of the focus has been put on the [assessment](https://github.com/konveyor/methodology#assess) and [rationalization](https://github.com/konveyor/methodology#rationalize) stages, offering little value to the planning required to scale out the migration process across the whole application portfolio. When dealing with a large scale portfolio of hundreds or even thousands of applications, it is simply not possible to execute the migration process on a big bang approach addressing the whole portfolio at once. A common way to tackle adoption at scale is to break the portfolio into different waves and execute the adoption effort in an iterative fashion. This enhancement aims to enable Konveyor to define migration waves with the applications in the inventory as a first step towards a more sophisticated approach that could automate calculation of these waves based on different criteria.


## Motivation

The enhancement provides the following advantages:

- Bring a new target persona for the Konveyor project: managers leading the migration initiative that need to plan the different iterations required to execute the project.
- Serve as a first step towards automated calculation of migration waves. This enhancement will introduce the basic entities and relationships that will then drive the automation on subsequent enhancements.
- Enable progress measurement in conjunction with the related [tackle-jira-integration](https://github.com/konveyor/enhancements/pull/85) enhancement.

### Goals

- Bring planning capabilities to the Konveyor project.
- Enable a new Project Manager persona to work on planning using Konveyor.
- Allow users to manually assign applications from the portfolio to Migration Waves.


### Non-Goals

- Automatically calculate waves based on a criteria. This will be the objective of a future enhancement.
- Manage integration with external task managers/issue trackers, as this is the main goal of the related [tackle-jira-integration](https://github.com/konveyor/enhancements/pull/85) enhancement.

## Proposal

### Personas / Actors

#### Architect

A technical lead for the migration project that can create and modify applications and information related to it.

#### Project Manager

**New persona to create in Konveyor with this enhancement**. A management lead for the migration project that can create and modify migration waves but can only assign an owner and contributors to an application and applications to waves in the application inventory.


### User Stories

#### Application owner and contributors

##### AOC001

*As an Architect and a Project Manager I want to be able to associate a stakeholder to an application as its owner*


##### AOC002

*As an Architect and a Project Manager I want to be able to associate stakeholders to an application as contributors*


#### Migration Waves Management

##### MWM001

*As a Project Manager I want to be able to manage (create, edit, update, delete) migration waves*

##### MWM002

*As a Project Manager I want to assign start and end dates to migration waves*

##### MWM003

*As a Project Manager I want to assign stakeholders and stakeholder groups to migration waves*

##### MWM004

*As a Project Manager I want to assign applications to migration waves*


#### Migration Waves Searchability

##### MWS001

*As an Architect, Migrator or Project Manager I want to be able to search applications based on the Migration Wave they were assigned to*


### Functional Specification

#### Updates in the Application Creation view

##### Related Use Cases

- [AOC001](#AOC001)
- [AOC002](#AOC002)

##### Involved Personas

- [Architect](#architect)


##### Description

The application creation view will include two new fields:

- Owner: Dropdown, single selection. The dropdown will include all available Stakeholders in the system. It would be good to make it searchable to avoid loading all stakeholders at once.

- Contributors: Dropdown, multiple selection. The dropdown will include all available Stakeholders in the system. It would be good to make it searchable to avoid loading all stakeholders at once.

![Application Creation view](images/app-creation-fields.png?raw=true "Application Creation view")

Since the architect role is the only one allowed to create applications, Project Managers won't be able to access this view.

#### Updates in the Application Edition view

##### Related Use Cases

- [AOC001](#AOC001)
- [AOC002](#AOC002)

##### Involved Personas

- [Architect](#architect)
- [Project Manager](#project-manager)


##### Description

The application creation view will include two new fields:

- Owner: Dropdown, single selection. The dropdown will include all available Stakeholders in the system. It would be good to make it searchable to avoid loading all stakeholders at once.

- Contributors: Dropdown, multiple selection. The dropdown will include all available Stakeholders in the system. It would be good to make it searchable to avoid loading all stakeholders at once.


![Application Creation view](images/app-edition-fields.png?raw=true "Application Creation view")


It is important to note that Project Managers should only be able to edit the Owner and Contributor fields, and the rest of the fields should appear disabled. Architects will have full access.


#### Migration Waves Management


##### Related Use Cases

- [MWM001](#MWM001)
- [MWM002](#MWM002)
- [MWM003](#MWM003)
- [MWM004](#MWM004)


##### Involved Personas

- [Project Manager](#project-manager)


##### Description

###### Migration Waves main view

A new option "*Migration Waves*" will be included in the left menu for the Developer Perspective. Clicking on it will navigate to the main view for Migration Waves. The view will revolve around compound expandable table with the following fields and sections per row:

- **Name**: Name of the migration wave. This field is optional and could appear empty.
- **Start date**: Start date for the migration wave.
- **End date**: End date for the migration wave.
- **Application**: Compound expandable section with a nested table that includes the list of applications assigned to the migration wave. Each row will have a kebab menu with just one option, "*Remove*", to remove the application belonging to the row from the list of assigned applications to the migration wave. Each row will include the following fields from the application:
  - *Application name*
  - *Description*
  - *Business service*
  - *Owner*

![Migration Waves view - Applications](images/migration-waves-applications-compound.png?raw=true "Migration Waves view - Applications")

- **Stakeholders**: Compound expandable section with a nested table that includes the list of stakeholders associated with a migration wave. This list will be automatically populated with all the owners and contributors to all the applications associated with the migration wave, and with any additional stakeholders that could have been associated directly to the migration wave on creation/edition or indirectly through the association of a determined stakeholder group. Just to clarify, **if a stakeholder group is associated to a migration wave on creation/edition, all of its members will appear in this list**. Each row will include the following fields:

  - *Name*
  - *Job Function*
  - *Role*: Owner, Contributor or empty if the stakeholder was associated directly or through a stakeholder group.
  - *Email*
  - *Stakeholder groups*: Groups the stakeholder belongs to.

![Migration Waves view - Stakeholders](images/migration-waves-stakeholders-compound.png?raw=true "Migration Waves view - Stakeholders")

- **Status**: Compound expandable section for the migration waves to reflect status data. This section is explained in depth in the related [tackle-jira-integration](https://github.com/konveyor/enhancements/pull/85) enhancement.

![Migration Waves view - Status](images/migration-waves-status-compound.png?raw=true "Migration Waves view - Status")


The different compound expandable sections can only be expanded when the value on the section is greater than zero:

![Migration Waves view - Empty section](images/migration-waves-compound-empty.png?raw=true "Migration Waves view - Empty section")

###### Create new waves

New migration waves can be created by clicking on the button "Create new" at the top section of the Migration Waves view. This will open a modal view with the following fields:

- **Name**: Optional. String with the name of the wave.
- **Start date**: Mandatory. Date picker.
- **End date**: Mandatory. Date picker.
- **Stakeholders**: Optional. Dropdown, multiple selection. Values will be the list of available stakeholders in the instance. Stakeholders to associate directly to the migration wave. When an application is assigned to a migration wave, all of its associated stakeholders get transitively associated to the wave. By associating them directly through this field, they will appear with no role in the Stakeholders compound expandable section.
- **Stakeholder Groups**: Optional. Dropdown, multiple selection. Values will be the list of available stakeholder groups in the instance. Stakeholder groups to associate directly to the migration wave. By associating a Stakeholder Group to a migration wave through this field, all the associated stakeholders in the group will appear with no role in the Stakeholders compound expandable section.

![Migration Waves view - Create wave](images/migration-waves-create.png?raw=true "Migration Waves view - Create wave")

There are some considerations regarding the Start date and End date fields. First, the Start date date picker field won't show any dates in the past, as it makes no sense to create new migration waves happening before the present day. First day to be displayed, will be the present day, highlighted in light blue:

![Migration Waves view - Create wave](images/migration-waves-create-start-date.png?raw=true "Migration Waves view - Create wave")

The End date date picker field will show the start date day that was picked and will leave a light blue highlight between that and the selected end date day.

![Migration Waves view - Create wave](images/migration-waves-create-end-date.png?raw=true "Migration Waves view - Create wave")

###### Edit waves

Migration waves can be edited by clicking on the "*Edit wave*" option available in the kebab menu for each migration wave row in the main table from the Migration Waves view. Clicking on the button will open a view with the same fields as the migration waves creation view:

![Migration Waves view - Edit wave](images/migration-waves-edit.png?raw=true "Migration Waves view - Edit wave")

The "*Stakeholders*" field will only display the stakeholders that were associated directly to the migration wave, and won't include the transitive association of stakeholders coming from assigned applications. The same applies for the "*Stakeholder groups*" field.

###### Manage applications

The assignment of applications to migration waves will be managed as an option in the Migration Waves view. This option will be called "*Manage Applications*" and will be included in the kebab menu available in each migration wave row:

![Migration Waves view - Manage Applications](images/manage-applications-option.png?raw=true "Migration Waves view - Manage Applications")

Clicking in the option will open a modal window with a table displaying all applications that haven't been previously assigned to another migration wave. This table should include the same filters available in the Application Inventory view except the "*Migration waves*" one:

![Migration Waves view - Manage Applications](images/manage-applications-modal.png?raw=true "Migration Waves view - Manage Applications")

The "*Selected wave*" field on top won't be modifiable and will only display the name or the range for the selected wave in case it doesn't have a name. As in the "*Applications*" coumpound expandable session from the main table in the Migration Waves view, the table will display the following fields on each application row:

- *Application name*
- *Description*
- *Business service*
- *Owner*


#### Updates in the Application Inventory View


##### Related Use Cases

- [MWS001](#MWS001)



##### Involved Personas

- [Architect](#architect)
- [Migrator](#migrator)
- [Project Manager](#project-manager)

##### Description

The Application Inventory view should include an additional filter "Migration Waves". Values for the filter should be the available Migration Waves in the system and "Unassigned" for those applications that haven't been assigned to a Migration Wave yet.

Also, the expandable section for each application will include information about the assigned Migration Wave, if any:


![Application Inventory view - Application details](images/app-inventory-wave.png?raw=true "Application Inventory view - Application details")

This field should be visible in both the Assessment and Analysis tabs.

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
