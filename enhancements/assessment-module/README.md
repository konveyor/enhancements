---
title: assessment-module
authors:
  - "rromannissen"
reviewers:
  - "@jortel"
  - "@mansam"
  - "@jwmatthews"
  - "@ibolton336"
approvers:
  - "jortel"
  - "@ibolton336"
  - "@jwmatthews"
creation-date: 2023-08-02
last-updated: 2023-08-02
status: provisional
see-also:
  -   
replaces:
  - The original assessment module in Konveyor that was based in the [Pathfinder project](https://github.com/redhat-cop/pathfinder).
superseded-by:
  -
---

# Enhanced Assessment Module


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

TBD

## Summary

Based on the feedback provided for the current assessment module, it has been determined that, although useful, its capabilities don't fully cover the differences that organizations might have between them. The general aspects of application containerization are addressed, but sometimes organizations have some particularities that can be key for assessing the suitability of a given application or application type/archetype. The very fact of assessments only covering application containerization matters has also been deemed insufficient by some users, that would like to have the flexibility of putting together their own questionnaires or expand on what the tool provides out of the box.

Another aspect, affecting UX, is the fact that assessments have a one to one relationship with applications. As large scale application modernization/migration projects are all about classifying application in different application types or archetypes to then come up with suitable migration strategies, it feels cumbersome that the assessment process has been designed to only have one application in sight. Copying assessments is also seen as clunky, and has problems when making changes to assessments at scale, which could be a situation when new information or stakeholders appear in the modernization lead's radar.

Finally, there have been some voices requesting more intelligence in the way questions get asked. For some users, there is a feeling of detachment between the information that gets collected in the application inventory and the assessment questionnaire. The general feedback is that if an application has been tagged in a certain way, the questions that get asked should be aligned to that, and if important information gets surfaced during the assessment, that should revert back to the inventory as well.


## Motivation

The Konveyor user experience should be fully centered in allowing users to manage applications and surface information about them at scale, which is not the case with the current assessment module. The original assessment module, briefly known as Pathfinder during the Tackle days, was built based on [a Red Hat consulting tool with the same name](https://github.com/redhat-cop/pathfinder) that didn't include the notion of an application inventory, thus the detachment described before. The tool wasn't conceived with scalability in mind, and was more oriented towards providing guidance on the early stages of a modernization/migration project. Nevertheless, [the possibility of loading custom question was available in the original tool](https://github.com/redhat-cop/pathfinder#using-custom-questions), but got lost somehow in the refactor that ported it to the Konveyor suite. This has led to users having to hack their way through the Pathfinder API or even the database to load their own questionnaires, which is far from [the fully integrated and seamless user experience that Konveyor aims to have](https://github.com/konveyor/enhancements/tree/master/enhancements/unified_experience#unified-experience-overview). In some extreme cases, users have opted to not use the assessment module at all, which led to [the requirement of being able to review an application without running an assessment](https://github.com/konveyor/enhancements/issues/88).

This enhancement aims to provide the following solution to these problems:

- The possibility of organizing applications in the portfolio in different Archetypes based on a series of tags.
- Assessing both individual applications or archetypes, allowing the process to work better at a large scale.
- Custom questionnaires with a new YAML syntax.
- Association of multiple questionnaires to an application or archetype.

### Goals

- Have a seamless user experience between the application inventory and the assessment module.
- Provide flexibility for the user to define and arrange their own questionnaires to tackle different migration and modernization scenarios.
- Improve the scalability of the assessment process for large portfolios.
- Introduce the notion of a standard YAML syntax for questionnaire definition.


### Non-Goals

- Have a fully UI supported way to author questionnaires.

## Proposal

### Personas / Actors

#### Administrator

The administrator for the tool that has access to some application-wide configuration parameters that other users can consume but not change or browse.

#### Architect

A technical lead for the migration project that can create and modify applications and information related to them.


### User Stories


#### Archetype Management

##### ARM001

*As an Architect I want to be able to manage archetypes*

##### ARM002

*As an Architect I want to be able to define an archetype by a series of tags, stakeholders and stakeholder groups.*

##### ARM003

*As an Architect I want to be able to add meta information about an archetype in the shape of comments*

##### ARM004

*As an Architect I want to be able to get the applications in the inventory automatically associated to an archetype if its associated tags match with the ones associated to the archetype*


#### Assessment management

##### ASM001

*As an Administrator I want to be able to manage questionnaires*

##### ASM002

*As an Administrator I want to be able to define which questionnaires need to be answered for an application to be considered as assessed*

##### ASM003

*As an Administrator I want to be able to import custom questionnaires using a defined YAML syntax*

##### ASM004

*As an Administrator I want to be able to export existing questionnaires as YAML files*

##### ASM005

*As an Administrator I want to be able to get a YAML template of a questionnaire*

#### Assessment process

##### ASP001

*As an Architect I want to be able to assess both applications and archetypes*

##### ASP002

*As an Architect I want to be able to individually assess an application that is associated with an archetype and override the assessment for that archetype*

##### ASP003

*As an Architect I want to be able to answer multiple questionnaires to assess an application*

##### ASP004

*As an Architect I want to be able to view an already answered questionnaire*

##### ASP005

*As an Architect I want to be able to re-take an already answered questionnaire*

##### ASP006

*As an Architect I want to be able to delete the answers for an already answered questionnaire*

##### ASP007

*As an Architect I want to be able to view archived questionnaires that were already answered*


### Functional Specification

#### Archetype management view

##### Related Use Cases

- [ARM001](#ARM001)
- [ARM002](#ARM002)
- [ARM003](#ARM003)
- [ARM004](#ARM004)


##### Involved Personas

- [Architect](#architect)


##### Description

###### Main view

A new top level option called "Archetypes" will be added in the left menu from the Migration perspective. This will lead to a new view that will allow architects to manage archetypes:

![Archetypes view](images/archetypes-main.png?raw=true "Archetypes view")

Each row will represent an existing archetype, containing the following information:

- **Name**: Name of the archetype
- **Description**: Brief description of the archetype
- **Tags**: The list of tags that define the archetype.
- **Maintainers**: List of stakeholders and stakeholder groups that maintain the archetype in the organization. This is metainformation that could be useful to navigate the organization and reach out to key people when more information about an archetype is needed.
- **Applications**: List of applications that are associated with the archetype. Association happens automatically via a match of the tags that define with the archetype with the ones that are associated with the application.

The header of the archetypes table will include filters that include Name, Description, Tags and Maintainers as criteria. Alongside the filtering, a button will be displayed to create a new archetype.

Each row will have a menu with the following options:

- **Assess**: Launches the assessment for a given archetype.
- **Edit**: Opens a modal window to edit an existing archetype.
- **Delete**: Deletes the archetype.


###### Creating new archetypes

New archetypes can be created by clicking on the "Create new archetype" button at the top menu. This will open a modal window with the following fields:

- **Name**: String
- **Description**: String (Optional)
- **Tags**: Dropdown, multi selection. All selected options will be displayed as removable labels.
- **Stakeholders**: Dropdown, multi selection. All selected options will be displayed as removable labels. (Optional)
- **Stakeholder group(s)**: Dropdown, multi selection. All selected options will be displayed as removable labels. (Optional)

![New archetype modal](images/archetype-create-filling.png?raw=true "New archetype modal")

All dropdown fields will have an autocomplete behavior. Tags appearing in the Tags field will be arranged by their corresponding categories:

![New archetype modal](images/archetype-create-tags.png?raw=true "New archetype modal")

Once values are entered in the dropdown, they will be displayed as removable labels below their corresponding field.

![New archetype modal](images/archetype-create-full.png?raw=true "New archetype modal")

Clicking on "Create archetype" will create the archetype, close the modal and refresh the archetypes view to include the newly created archetype.

**Creating a new archetype should trigger a process to calculate the applications the archetype is associated with.** If the user remains in the Archetype view, it should be automatically refreshed once the process ends to reflect the associated applications.


###### Editing archetypes

Existing custom migration targets can be edited by clicking on the kebab at the right side of the corresponding row and selecting the option "Edit".

This will open a modal window similar to the one used for creating archetypes, but with a "Save" button instead of the "Create archetype" one. **If the list of tags that define the archetype changes, the process to calculate the applications the archetype is associated with should be triggered again**.

###### Deleting archetypes

Existing archetypes can be deleted by clicking on the kebab at the right side of the corresponding row and selecting the option "Delete". This will open a modal window asking for confirmation with the following message:

*This action cannot be undone. Deleting an archetype will delete any associated assessment, bringing all associated applications to the "Unassessed" state. Do you wish to continue?*

As stated in the message, **deleting an archetype will delete any associated assessment, bringing all associated applications to the "Unassessed" state**.

###### Archetype association logic

Applications are automatically associated with archetypes based on their associated tags. Those tags can come from any source, so as long as all tags associated to an archetype are present in an application in any of its sources, the archetype should be associated to that application. The only exception to this happens when an archetype contains a subset of the tags of another archetype, in which case the application will only be associated to the archetype with tags that match with the full list of tags of the archetype.

For example, if archetype 1 is defined by tags (a,b), archetype 2 is defined by tags (a,b,c), archetype 3 is defined by tags (d,e) and application A is associated to tags (a,b,c,d,e), then application A would be automatically associated to archetypes 2 and 3, but not with 1 as its tags are a subset of the ones from 2.

#### Assessment management view

##### Related Use Cases

- [ASM001](#ASM001)
- [ASM002](#ASM002)
- [ASM003](#ASM003)
- [ASM004](#ASM004)
- [ASM005](#ASM005)


##### Involved Personas

- [Administrator](#administrator)


##### Description

###### Main view

A new top level option called "Assessment" will be added in the left menu from the Administration perspective. This will lead to a new view that will allow administrators to manage questionnaires:

![Set targets screen](images/assessment-management-main.png?raw=true "Set targets screen")

The Assessment management view provides administrators with a range of actions and important information about their questionnaires. Each row represents a questionnaire and includes the following fields:

- **Required**: Administrators can turn questionnaires on or off to define the questionnaires required to consider an application as assessed. At least one questionnaire should be required.
- **Name**: Name of the questionnaire. A lock icon will be displayed alongside the name for system bundled questionnaires that cannot be deleted, ensuring their integrity.
- **Questions**:  The number of questions within each questionnaire.
- **Rating**: The different percentage thresholds to consider the risk level of an application to belong to a certain risk category (Red, Yellow, Green and Unknown).
- **Date imported**: The date of questionnaire importation is shown, helping track the timeline of updates.

The header of the table will include the following:

- **Search**: Users can search for specific questionnaires based on their names. 
- **Filter**: Quick filter that allows the user to display required, available (non required) or all questionnaires.
- **Import Questionnaire**: The option to import new questionnaires is available, allowing administrators to add customized questionnaires.
- **Download YAML Template**: A link is provided to download the YAML questionnaire template.

Each row in the table will include a kebab menu that offers the following options:

- **Export**: Exports the given questionnaire in YAML.
- **View**: Renders a detailed view of the given questionnaire.
- **Delete**: Deletes the questionnaire.

>**Note**: Deleting a questionnaire will cascade into the deletion of all answered questionnaires associated to applications and/or archetypes.

>**Note**: If while a questionnaire is required it gets answered for any application and/or archetype, and then the questionnaire transitions to non required, the answered questionnaires for those applications and/or archertypes will remain stored in the system and will be considered *archived* and accessible in view only mode.


### Implementation Details/Notes/Constraints

TBD


### Security, Risks, and Mitigations

TBD

## Design Details

TBD

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
