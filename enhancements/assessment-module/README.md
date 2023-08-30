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
last-updated: 2023-08-30
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

- Have a fully UI-supported way to author questionnaires.

## Proposal

### Personas / Actors

#### Administrator

The administrator for the tool that has access to some application-wide configuration parameters that other users can consume but not change or browse.

#### Architect

A technical lead for the migration project that can create and modify applications and information related to them.

#### Migrator

A developer that should be allowed to run analysis, but not to assess, create or modify applications in the portfolio.


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

##### ASM006

*As an Administrator I want to be able to define answers that automatically tag an application*

##### ASM007

*As an Administrator I want to be able to define answers that should be automatically chosen based on the existing tags associated to an application or archetype*

##### ASM008

*As an Administrator I want to be able to define conditional questions based on the existing tags associated to an application or archetype*

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


##### ASP008

*As an Architect I want to be able to view reports for all different required questionnaires*


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

###### Archetypes Main view

A new top level option called *Archetypes* will be added in the left menu from the Migration perspective. This will lead to a new view that will allow architects to manage archetypes:

![Archetypes view](images/archetypes-main.png?raw=true "Archetypes view")

Each row will represent an existing archetype, containing the following information:

- **Name**: Name of the archetype
- **Description**: Brief description of the archetype
- **Criteria Tags**: The list of tags that define the archetype.
- **Maintainers**: List of stakeholders and stakeholder groups that maintain the archetype in the organization. This is metainformation that could be useful to navigate the organization and reach out to key people when more information about an archetype is needed.
- **Applications**: List of applications that are associated with the archetype. Association happens automatically via a match of the criteria tags that define with the archetype with the ones that are associated with the application.

The header of the archetypes table will include filters that include Name, Description, Criteria Tags and Maintainers as criteria. Alongside the filtering, a button will be displayed to create a new archetype.

Each row will have a menu with the following options:

- **Assess**: Launches the assessment for a given archetype.
- **Review**: Navigates to the review view for the archetype.
- **Edit**: Opens a modal window to edit an existing archetype.
- **Delete**: Deletes the archetype.

Clicking on a row will open a side drawer that will include additional fields:

![Archetypes view](images/archetypes-drawer.png?raw=true "Archetypes view")

- **Archetype Tags**: The list of tags that will be applied to applications that are associated to the archetype in the *Archetype* source.
- **Assessment Tags**: The list of tags, coming from the assessment of the Archetype via automated tagging, that will be applied to applications that are associated to the archetype in the *Assessment* source.


###### Creating new archetypes

New archetypes can be created by clicking on the *Create new archetype* button at the top menu. This will open a modal window with the following fields:

- **Name**: String
- **Description**: String (Optional)
- **Criteria Tags**: Dropdown, multi selection. All selected options will be displayed as removable labels.
- **Archetype Tags**: Dropdown, multi selection. All selected options will be displayed as removable labels.
- **Stakeholders**: Dropdown, multi selection. All selected options will be displayed as removable labels. (Optional)
- **Stakeholder group(s)**: Dropdown, multi selection. All selected options will be displayed as removable labels. (Optional)

![New archetype modal](images/archetype-create.png?raw=true "New archetype modal")

All dropdown fields will have an autocomplete behavior. Tags appearing in the Tags field will be arranged by their corresponding categories:

![New archetype modal](images/archetype-create-tags.png?raw=true "New archetype modal")

Once values are entered in the dropdown, they will be displayed as removable labels below their corresponding field.

![New archetype modal](images/archetype-create-full.png?raw=true "New archetype modal")

Clicking on "Create archetype" will create the archetype, close the modal and refresh the archetypes view to include the newly created archetype.

**Creating a new archetype should trigger a process to calculate the applications the archetype is associated with based on the Criteria Tags.** If the user remains in the Archetype view, it should be automatically refreshed once the process ends to reflect the associated applications.

>**Note**: Tags from sources Archetype and Assessment will be ignored when calculating the archetype match for applications.


###### Editing archetypes

Existing custom migration targets can be edited by clicking on the kebab at the right side of the corresponding row and selecting the option *Edit*.

This will open a modal window similar to the one used for creating archetypes, but with a *Save* button instead of the *Create archetype* one. **If the list of Criteria Tags that define the archetype changes, the process to calculate the applications the archetype is associated with should be triggered again**.

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
- [ASM006](#ASM006)
- [ASM007](#ASM007)
- [ASM008](#ASM008)


##### Involved Personas

- [Administrator](#administrator)


##### Description

###### Main view

A new top level option called *Assessment* will be added in the left menu from the Administration perspective. This will lead to a new view that will allow administrators to manage questionnaires:

![Assessment Management](images/assessment-management-main.png?raw=true "Assessment Management")

The Assessment management view provides administrators with a range of actions and important information about their questionnaires. Each row represents a questionnaire and includes the following fields:

- **Required**: Administrators can turn questionnaires on or off to define the questionnaires required to consider an application as assessed. At least one questionnaire should be required.
- **Name**: Name of the questionnaire (unique). A lock icon will be displayed alongside the name for system bundled questionnaires that cannot be deleted, ensuring their integrity.
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

###### Importing Questionnaires

Administrators can import YAML questionnaires by clicking on the *Import questionnaire* button at the header of the questionnaires table.

![Assessment Management](images/assessment-management-import.png?raw=true "Assessment Management")

This will open a modal window with a single file upload field. The uploaded YAML file will be validated for the following:

- Correct YAML syntax. See the [YAML Syntax](#yaml-syntax) section for more details.
- The previous existence of tags referenced by the questionnaire in conditional questions or autotag answers. If any of the referenced tags doesn't exist in the system, validation will fail and the user will be prompted with a validation message explaining it and providing the list of missing tags.
- Correctness in the percentages expressed in the *thresholds* section.

Once the file is validated, the "Import" button will be enabled and the user will be able to import the file as a new questionnaire.

###### Exporting Questionnaires

Existing questionnaires will be exportable as YAML files. For that, the user will have to click on the *Export* option from the kebab menu available on the row that represents the existing questionnaire the user wants to export. After clicking on *Export*, the system will generate the YAML file and initiate the download in the browser.

###### Editing Questionnaires

For practical reasons, **it won't be possible to edit existing questionnaires**. Administrators that want to modify a questionnaire will have to export it, make modifications and reimport it with a new name. Then, to replace one questionnaire with the other for architects assessing applications from the portfolio, the original one should be marked as non required (*Required*: Disabled) and the modified one should be marked as required (*Required*: Enabled). This will archive any previously answered questionnaires from the original one, which would be accessible in read only mode, and prompt the modified one as not answered.

###### Viewing Questionnaires

It will be possible for the Administrator to get a rendered view of the questionnaire once it listed in the questionnaires table. Clicking on the *View* option from the kebab menu of a row from the questionnaires table will navigate to a new view that renders the given questionnaire:

![Assessment Management](images/assessment-management-view.png?raw=true "Assessment Management")

As in the assessment view from the application inventory, questions are arranged in different sections accessible through a set of vertical tabs. Each question in a section is represented as an expandable section, which once expanded will display the available answers for that question:

![Assessment Management](images/assessment-management-view-expanded.png?raw=true "Assessment Management")

Risk levels (Red, Yellow, Green) will be displayed as icons on each answer under the *Risk* column. Answers that imply autotagging will have an additional section below the choice text that will include the list of tags to be applied using labels to render each tag. The same applies for answers that would be automatically selected, rendering the list of tags that would trigger the selection of the answer. Conditional questions will have a *Conditional* label before the formulation text:

![Assessment Management](images/assessment-management-view-conditional.png?raw=true "Assessment Management")

Upon expanding a conditional question, an additional section will be displayed above the list of answers, providing information about the condition. This could be either *Include if present* or *Skip if present* followed by a list of tags rendered using labels.

###### Deleting Questionnaires

Questionnaire can be deleted by selecting the *Delete* option from the kebab menu in the questionnaire row. This should open a modal window displaying the following text:

*Deleting a questionnaire will cascade into the deletion of all answered questionnaires associated to applications and/or archetypes. This action cannot be undone. Please enter the name of the questionnaire below to confirm:*

Below that, a text field your be displayed for users to enter the name of the questionnaire to be deleted. The *Delete* button in the modal won't be enabled until the user enters the right name.

###### YAML Syntax

The YAML syntax for questionnaire definition aims at helping users simplify questionnaire authoring, while providing additional tools that allow a more dynamic behavior for the questionnaire based on tags associated to the target application or archetype. The questionnaire is structured with the following fields and sections:

- **name**: *String. Required. Unique, questionnaire import will fail if the another questionnaire with the same name exists already.* Name of the questionnaire.
- **description**: *String. Optional.* A short description of the questionnaire.
- **sections**: *List. Required.* List of sections that the questionnaire will include. Each section is defined with the following fields:
  - **name**: *String. Required.* Name to be displayed for the section.
  - **questions**: *List. Required.* List of questions that belong to the section. Each question is defined with the following fields:
   - **formulation**: *String. Required.* Formulation of the question to be asked.
   - **explanation**: *String. Optional.* Additional explanation for the question.
   - **include_if_tags_present**: *List. Optional.* Defines that a question should be displayed only if any of the tags included in the list is present in the target application or archetype (OR logic). Each tag is defined by the following:
      - **category**: *String. Required.* Category of the target tag.
      - **tag**: *String. Required.* Tag definition for the target tag.
    - **skip_if_tags_present**: *List. Optional.* Defines that a question should be skipped if any of the tags included in the list is present in the target application or archetype (OR logic). Each tag is defined by the following:
      - **category**: *String. Required.* Category of the target tag.
      - **tag**: *String. Required* Tag definition for the target tag.
    - **answers**: *List. Required.* List of answers for the given question. Each answer is defined by the following fields:
      - **choice**: *String. Required.* The actual answer for the question.
      - **risk**: *String, limited values: red, yellow, green, unknown. Required*. The risk level the current answer implies.
      - **rationale**: *String. Optional.** An explanation of the rationale behind the answer being considered a risk.
      - **mitigation**: *String. Optional.* An explanation of the potential mitigation strategy for the risk implied by this answer.
      - **autotag**: *List. Optional.* Defines a list of tags to be automatically applied to the assessed application or archetype (as transitive tags) if this answer is selected. Each tag is defined by the following:
        - **category**: *String. Required.* Category of the target tag.
        - **tag**: *String. Required* Tag definition for the target tag.
      - **autoanswer_if_tags_present**: *List. Optional.* Defines a list of tags that will lead to this answer being automatically selected when the application or archetype is assessed. Each tag is defined by the following:
        - **category**: *String. Required.* Category of the target tag.
        - **tag**: *String. Required.* Tag definition for the target tag.
- **thresholds**: *Map. Required.* The threshold definition for each risk category for the application or the archetype to be considered affected by that risk level. The higher risk level always takes precedence. For example, if yellow threshold is established in 30% and red in 5%, and the answers for an application or archetype have 35% yellow and 6% red, the risk level for the application or archetype will be red. The threshold map is defined by the following fields:
    - **red**: *Numeric. Percentage. Required.* Percentage of red answers the questionnaire can have until the risk level is considered red.
    - **yellow**: *Numeric. Percentage. Required.* Percentage of yellow answers the questionnaire can have until the risk level is considered yellow.
    - **unknown**: *Numeric. Percentage. Required.* Percentage of unknown answers the questionnaire can have until the risk level is considered unknown.
- **risk_messages**: *Map. Required.* The messages to be displayed in reports for each risk category. The risk_messages map is defined by the following fields:
  - **red**: *String. Required* The message to display in reports for the red risk level.
  - **yellow**: *String. Required* The message to display in reports for the yellow risk level.
  - **green**: *String. Required* The message to display in reports for the green risk level.
  - **unknown**: *String. Required* The message to display in reports for the unknown risk level.

The following is a sample questionnaire using the previously defined YAML syntax:

```yaml
name: Cloud Native
description: |
  Questionnaire that includes the Twelve Factor Application principles.
sections:
  - name: Application technologies
    questions:
      - formulation: What is the main technology in your application?
        explanation: What would you describe as the main framework used to build your application.
        answers:
          - choice: Unknown
            rationale: This is a problem because of the uncertainty.
            mitigation: Gathering more information about this is required.
            risk: unknown
          - choice: Quarkus
            risk: green
            autoanswer_if_tags_present:
              - category: Technology
                tag: Quarkus
            autotag:
              - category: Technology
                tag: Quarkus
          - choice: Spring Boot
            risk: green
            autoanswer_if_tags_present:
              - category: Technology
                tag: Spring Boot
            autotag:
              - category: Technology
                tag: Spring Boot
          - choice: Java EE
            rationale: This might not be the most cloud friendly technology.
            mitigation: Maybe start thinking about migrating to Quarkus or Jakarta EE.
            risk: yellow
            autoanswer_if_tags_present:
              - category: Technology
                tag: Java EE
            autotag:
              - category: Technology
                tag: Java EE
          - choice: J2EE
            rationale: This is obsolete.
            mitigation: Maybe start thinking about migrating to Quarkus or Jakarta EE.
            risk: red
            autoanswer_if_tags_present:
              - category: Technology
                tag: J2EE
            autotag:
              - category: Technology
                tag: J2EE
      - formulation: "What version of Java EE does the application use?"
        explanation: "What version of the Java EE specification is your application using?"
        answers:
          - choice: Below 5.
            rationale: This technology stack is obsolete.
            mitigation: Consider migrating to at least Java EE 7.
            risk: red
          - choice: 5 or 6
            rationale: This is a mostly outdated stack.
            mitigation: Consider migrating to at least Java EE 7.
            risk: yellow
          - choice: "7"
            risk: green
        include_if_tags_present:
          - category: Technology
            tag: Java EE
      - formulation: Does your application use any caching mechanism?
        answers:
          - choice: Yes
            rationale: This could be problematic in containers and Kubernetes.
            mitigation: Review the clustering mechanism to check compatibility and support for container environments.
            risk: yellow
            autoanswer_if_tags_present:
              - category: Caching
                tag: Infinispan
              - category: Caching
                tag: Datagrid
              - category: Caching
                tag: eXtreme Scale
              - category: Caching
                tag: Coherence
          - choice: No
            risk: green
          - choice: Unknown
            rationale: This is a problem because of the uncertainty.
            mitigation: Gathering more information about this is required.
            risk: unknown
      - formulation: What implementation of JAX-WS does your application use?
        answers:
          - choice: Apache Axis
            rationale: This version is obsolete
            mitigation: Consider migrating to Apache CXF
            risk: red
          - choice: Apache CXF
            risk: green
          - choice: Unknown
            rationale: This is a problem because of the uncertainty.
            mitigation: Gathering more information about this is required.
            risk: unknown
        skip_if_tags_present:
          - category: Technology
            tag: Spring Boot
          - category: Technology
            tag: Quarkus
thresholds:
  red: 1%
  yellow: 30%
  unknown: 15%
risk_messages:
  red: Application requires deep changes in architecture or lifecycle
  yellow: Application is Cloud friendly but requires some minor changes
  green: Application is Cloud Native
  unknown: More information about the application is required
```

#### Changes in the Assessment process

##### Related Use Cases

- [ASP001](#ASP001)
- [ASP002](#ASP002)
- [ASP003](#ASP003)
- [ASP004](#ASP004)
- [ASP005](#ASP005)
- [ASP006](#ASP006)
- [ASP007](#ASP007)
- [ASP008](#ASP008)

##### Involved Personas

- [Architect](#architect)


##### Description

###### Assessment flow for applications

Since this enhancement introduces the possibility of requiring multiple questionnaires to be answered for an application to be considered assessed, it requires the introduction of some changes in the overall process. Clicking on the *Assess* button for an application will now lead to an intermediate view in which the user will be presented with the list of required questionnaires.


![Assessment Process](images/assessment-process-start.png?raw=true "Assessment Process")

If no questionnaire has been answered yet, only the *Take* button will be available. Clicking on it will kickstart the usual assessment flow with no major changes from the previous single questionnaire approach, aside from the fact that visual cues will be offered for those questions that have been automatically answered (**Mockup missing**). Once a questionnaire is completed, more options will be available for the user for it:

 ![Assessment Process](images/assessment-process-completed.png?raw=true "Assessment Process")

 - *Retake* will prompt the questionnaire again with all the previous answers already selected, similar to editing an already answered questionnaire.
 - *View* will render a read only view of the questionnaire highlighting the selected questions.
 - *Delete* will, upon confirmation, delete the provided answers for a questionnaire and bring it back to the original unanswered state.

 As stated before, both applications and archetypes can be assessed. Applications associated with an archetype will *inherit* its assessment(s). Nevertheless, individual applications that are associated to an archetype can also have a dedicated assessment that overrides the one(s) inherited from the archetype. Clicking on "Assess" for an application that is associated to an archetype will open a confirmation modal to override the inherited assessment:

  ![Assessment Process](images/assessment-process-override.png?raw=true "Assessment Process")

Deleting the dedicated assessment of an application associated to an assessed archetype will bring that application back to its previous state of inheriting the archetype assessment(s).

###### Assessment flow for archetypes

Archetypes will be assessable by clicking on the *Assess* option from the kebab menu from a row in the table from the [archetypes view](#archetypes-main-view). Doing that will lead to the questionnaire list view and kickstart the assessment process in the exact same way described before for applications.

> **Note**: Conditional questions on a questionnaire belonging to the assessment of an archetype will only take into account tags directly associated with the archetype in both Criteria and Archetype tags, and will ignore tags coming from applications associated to that archetype.

> **Note**:  Tags in the assessment source of an application coming from the assessment of an associated archetype are transitive. If the application stops being associated to the archetype or the archetype is deleted, the tags will be removed from the application.

###### Assessment questionnaire

The assessment questionnaire displayed to the user to provide answers will remain mostly the same, aside from the fact that visual cues will be included to inform the user about autoanswered questions:

  ![Assessment Process](images/assessment-autoanswer.png?raw=true "Assessment Process")

Aside from the icon, there will be a tooltip explaining why the answer was automatically selected:

*Selected based on tag(s) X associated to archetype/application Y*

The user will be able to change the selection, but the icon will remain on the previously autoselected answer.


If the application has tags that would qualify for more than one answer in a question, no answer would be selected, but the icon would be displayed for each potential autoselected answer, with different tooltips explaining the origin of each answer.

#### Changes in the Review process

##### Related Use Cases

- [ASM002](#ARM002)
- [ASP003](#ASM003)

##### Involved Personas

- [Architect](#architect)

##### Description

The review flow will stay mostly the same, but now architects should be able to review both archetypes from the *Archetypes* view or individual applications from the *Application inventory* view. As with assessments, applications associated to an archetype that has been reviewed will inherit the review, and it will be possible to override it with a dedicated review as well.

The main change in the *Review* view will be related to the way risks are displayed. The *Assessment summary* section will be rearranged to accommodate *Rationale* and *Mitigation strategies* data as expandable rows:

![Review](images/review-risks.png?raw=true "Review")

Only answers that contain *Rationale* and *Mitigation strategies* data will be displayed as expandable. Risk will be displayed with the same icons used in the questionnaire views instead of the labels that were used before. Filter criteria should now include Category (text), Question (text), Answer (text) and Risk (High, Medium, Low, Unknown).

#### Changes in the Application Inventory view

##### Related Use Cases

- [ARM004](#ARM004)
- [ASM006](#ASM006)

##### Involved Personas

- [Architect](#architect)
- [Migrator](#migrator)

##### Description

###### Main view

The *Assess* button in the top menu from the Assessment tab will be removed, and the *Assess* action will be made available as an option in the kebab menu for each row. This will leave the kebab menu with the following options:

- **Assess**: Highlight in yellow if the application has any inherited assessments coming from associated archetypes, white background otherwise.
- **Review**:  Highlight in yellow if the application has any inherited reviews coming from associated archetypes, white background otherwise.
- **Discard assessment**: Only available if the application has been assessed individually. This will revert back to the inherited assessment(s) if the application is associated with any archetype(s).
- **Discard review**: Only available if the application has been reviewed individually. This will revert back to the inherited review(s) if the application is associated with any archetype(s).
- **View assessment**: Only available when the application or any of its associated archetypes have been assessed.
- **Manage dependencies**: Always white background.
- **Delete**: Always highlighted in red.

The options *Copy assessment* and *Copy assessment and review* available in previous releases will be removed from the kebab menu, as these actions are no longer available.

###### Side drawer

The application profile side drawer will include the list of associated archetypes in the *Archetypes* field. If no archetypes are associated to the application, the field will display "None".

![Side drawer](images/drawer-multiple-reviews.png?raw=true "Side drawer")

Aside from that, since applications can be associated with multiple archetypes, and those archetypes can potentially have a review for each of them, the *Proposed action*, *Effort estimate*, *Business Criticality* and *Work priority* fields will display the values for each archetype that has been reviewed, adding the name of the corresponding archetype in parenthesis. If the application is only associated with one archetype or if it has been reviewed directly, the name of the archetype won't be rendered alongside the value for each of these fields.

To calculate the aggregated risk level for an application, the following formula, inherited from the previous Pathfinder module, should be applied:

- Any Red means overall risk level is Red.
- Yellow above 30% means overall risk level is Yellow.
- Unknown above 20% means overall risk level is Red.
- Any other percentages mean overall risk level is Green.

**This rule applies both for questionnaires inside an assessment and multiple assessments indirectly associated to an application**. Making these thresholds configurable could be explored in future releases.

**Two new sources will be available in the Tags tab**: *Assessment* and *Archetype*, which will include transitive tags coming from associated archetypes.

> **Note**:  Tags in the **Assessment** and **Archetype** sources of an application coming from an associated archetype are transitive. If the application stops being associated to the archetype or the archetype is deleted, the tags will be removed from the application.


###### View Assessment

The *View assessment* view will allow users to browse the answers provided on the different questionnaires that belong to the assessments(s) associated with an application directly or transitively through archetype(s). Clicking on the *View assessment* option in the kebab menu for a given application will navigate away from the inventory view to another view in which the user will see the list of questionnaires available for the assessment for a given archetype in case there are many.

![Assessment View](images/assessment-view-assessment-questionnaires.png?raw=true "Assessment View")

The *Archetype* dropdown at the top will display the list of archetypes associated to the current application. If the application is not associated with any archetype (the application has been assessed individually), this dropdown won't be rendered.

Clicking on the *View* button for any questionnaire will navigate to a different view that renders the questionnaire, highlighting the answers that were provided for each question:

![Assessment View](images/assessment-view-assessment-view.png?raw=true "Assessment View")

Highlighted answers will also include their associated risk level, which won't be displayed for the other choices that hadn't been selected for each question.

The user will be able to filter questions using a search box. Each vertical tab corresponding with categories will highlight the number of matches for the query:

![Assessment View](images/assessment-view-assessment-view-filter.png?raw=true "Assessment View")

#### Changes in the Application Reports view

##### Related Use Cases

- [ARM004](#ARM004)
- [ASM006](#ASM006)

##### Involved Personas

- [Architect](#architect)
- [Migrator](#migrator)

##### Description

Some changes will be required in the *Current landscape* and *Identified risks* from the *Reports* view. Firstly, in order to accommodate data from multiple questionnaires, the *Current landscape* section will include a dropdown to select the questionnaire for the data to be displayed, with "All questionnaires" selected by default.

![Reports](images/reports-overview.png?raw=true "Reports")

To calculate the aggregated risk level for an application, the following formula, inherited from the previous Pathfinder module, should be applied:

- Any Red means overall risk level is Red.
- Yellow above 30% means overall risk level is Yellow.
- Unknown above 20% means overall risk level is Red.
- Any other percentages mean overall risk level is Green.

**This rule applies both for questionnaires inside an assessment and multiple assessments indirectly associated to an application**. Making these thresholds configurable could be explored in future releases.

If the user selects a concrete questionnaire, the *risk_messages* data will be displayed below each risk level, and data on the doughnut charts will be updated to reflect the specific values for the selected questionnaire:

![Reports](images/reports-questionnaire-selected.png?raw=true "Reports")


The *Identified risks* section will be rearranged to accommodate *Rationale* and *Mitigation strategies* data as expandable rows:

![Reports](images/reports-risks.png?raw=true "Reports")

Only answers that contain *Rationale* and *Mitigation strategies* data will be displayed as expandable. Risk will be displayed with the same icons used in the questionnaire views instead of the labels that were used before. Filter criteria should now include Application (text, multiselect), Category (text), Question (text), Answer (text) and Risk (High, Medium, Low, Unknown).


The number of applications displayed in the *Applications* row on the table will be static text and not a link, which could be explored in future releases.

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
