---
title: migration-waves
authors:
  - "rromannissen"
reviewers:
  - "@jortel"
  - "@mansam"
  - "@jwmatthews"
  - "@ibolton336"
approvers:
  - "jortel"
creation-date: 2023-02-02
last-updated: 2023-02-02
status: provisional
see-also:
  -   
replaces:
  -
superseded-by:
  -
---

# Custom Migration Targets


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

- Something similar to this user experience would be very helpful in the IDE plugins. How could something like this be achieved?

## Summary

Large scale modernization and migration projects are often led by a group of architects working together as a steering team and a series of third party factories that execute the transformation at scale. In order to reduce costs, it's not unusual for these third party factories to provide developers with a very focused set of skills, making it difficult and expensive for them to learn new tools and technologies, especially if they are complex. One of the key concerns of the Konveyor project is to provide differential value on each stage of the modernization process, and that includes making sure we lower the barrier for these developers to take advantage of the capabilities of the Konveyor toolkit when the transformation is executed at scale.

Through interaction with different focus user groups in the field, it has been discovered that configuring custom rules on each analysis run can be cumbersome and doesn't scale well when dealing with third party factories, as it requires them to understand some concepts from the Konveyor domain that could be abstracted away. The purpose of this enhancement is to provide an abstraction layer for these users, hiding away the complexities of custom rules configuration by exposing them as custom migration targets managed by advanced users.


## Motivation

The enhancement provides the following advantages:

- Developers don't necessarily need to understand how custom rules work in order to take advantage of them.
- Architects can maintain custom rules in their own, and make sure third party developers use them in a straightforward way.
- Analysis can be configured faster when dealing with a series of well known custom technologies in an organization that are widespread across the whole application portfolio.

### Goals

- Lower the barrier for the usage of custom rules at scale.
- Abstract unskilled users from the complexities of custom rules development and configuration.
- Simplify analysis configuration and execution.


### Non-Goals

- Replace the usage of custom rules in analysis.
- Modify the mechanics of how custom rules are developed or configured in the Windup CLI.

## Proposal

### Personas / Actors

#### Administrator

The administrator for the tool that has access to some application-wide configuration parameters that other users can consume but not change or browse.

#### Architect

A technical lead for the migration project that can create and modify applications and information related to them.

#### Migrator

A developer that should be allowed to run analysis, but not to assess, create or modify applications in the portfolio.

### User Stories


#### Custom Migration Targets Management

##### CMTM001

*As an Administrator I want to be able to manage custom migration targets*

##### CMTM002

*As an Administrator I want to be able to upload custom rules files and assign them to a custom migration target*

##### CMTM003

*As an Administrator I want to be able to assign rules from a repository to a custom migration target*

##### CMTM004

*As an Administrator I want to be able to assign an icon to custom migration targets*

##### CMTM005

*As an Administrator I want to be able to determine the order in which custom migration targets are displayed in the analysis configuration wizard*

#### Analysis Configuration Wizard

##### ACW001

*As an Architect or a Migrator, I want to be able to select custom migration targets when configuring an analysis in the analysis configuration wizard*


##### ACW002

*As an Architect or a Migrator, I want to be able to distinguish between custom migration targets and the default migration targets that come out of the box with Konveyor*


##### ACW003

*As an Architect or a Migrator, I want to be able to consume custom rules from a git repository*


### Functional Specification

#### Custom Migration Targets view

##### Related Use Cases

- [CMTM001](#CMTM001)
- [CMTM002](#CMTM002)
- [CMTM003](#CMTM003)
- [CMTM004](#CMTM004)
- [CMTM005](#CMTM005)


##### Involved Personas

- [Administrator](#administrator)


##### Description

###### Main view

A new top level option called "Custom migration targets" will be added in the left menu from the Administrator perspective. This will lead to a new view that will allow administrators to manage custom migration targets:

![Custom migration targets view](images/custom-migration-targets-screen.png?raw=true "Custom migration targets view")

Custom migration targets will be represented as cards alongside the default migration targets that ship out of the box with Konveyor. As they are created, custom migration target cards would appear at the bottom of the list, unless rearranged by the user. Users can drag and drop the cards to rearrange them:

![Custom migration targets view](images/custom-migration-targets-screen-drag.png?raw=true "Custom migration targets view")

Any changes to the layout will be reflected in the order in which cards are presented to Architects and Migrators in the analysis configuration wizard.

![Custom migration targets view](images/custom-migration-targets-screen-drop.png?raw=true "Custom migration targets view")

All cards, including the ones representing default migration targets, can be dragged and dropped to rearrange the layout, but **only custom migration target cards are editable**, with the available actions (edit and delete) available in the card's header as a kebab action menu.


###### Creating new custom migration targets

New custom migration targets can be created by clicking on the "Create new" button at the top menu. This will open a modal window with the following fields:

- **Name**: String
- **Description**: String (Optional)
- **Image**: File upload (Optional)
- **Custom rules**: Radio button. Options:
  - Upload Manually (default)
  - Upload from a repository


User will be able to upload JPEG and PNG images of up to 1Mb of size. Clicking in the "Upload" button will prompt for file selection. Once the file is selected it will be automatically resized to 80x80 pixels and its file name will appear in the File Upload component. Clicking on "Clear" will remove the image and its file name and restart the upload process.

When "Upload Manually" is selected in the Custom rules radio button, a section with a drag & drop file upload component will be displayed. **The behavior for this section should be exactly the same as the "Add rules" modal from the analysis configuration wizard**:

![New custom migration target modal](images/new-target-manual.png?raw=true "New custom migration target modal")

After selecting the files to upload or dragging them to the upload component, the upload will begin and a new expandable section will be displayed to keep track of the upload progress:


![New custom migration target modal](images/new-target-manual-uploading-collapsed.png?raw=true "New custom migration target modal")

Clicking on the arrow will expand a section with the upload details, including the list of files and a progress bar for each one of them. While the upload is in progress, the "Create" button will be disabled:

![New custom migration target modal](images/new-target-manual-uploading-details.png?raw=true "New custom migration target modal")

When the user uploads one or several files, validation should execute automatically once the file has been fully received. Validation will consist on the following:

- The file has a ".windup.xml" suffix. This is a requirement from the Windup CLI, so this validation is a way to make sure that any rules file that gets passed will not be ignored by the CLI.

- The file is validated against the following XML Schema Definition: https://windup.jboss.org/schema/jboss-ruleset/windup-jboss-ruleset.xsd

> **Note**: Konveyor should be capable of running in disconnected Kubernetes instances, so the XSD file should be available locally for the validator, as downloading it might not be an option.

Once the validation is done, each file will have a green or red bar depending on the validation results for each one of them. The "Add" button will be enabled if at least one file is valid.

![New custom migration target modal](images/new-target-manual-uploading-success.png?raw=true "New custom migration target modal")

If any of the files is not valid, the red bar and an error message will be displayed under the corresponding file. If any of the files is not valid clicking on "Create" will result in only the valid files being associated with the custom migration target.

When "Retrieve from a repository" is selected, a section with the following fields will be displayed:

- **Repository type**: Dropdown. Values:
  - Git
  - Subversion
- **Source repository**: String.
- **Branch**: String. (Optional)
- **Root path**: String. (Optional)
- **Associated credentials**: Dropdown. It will display a list with all Source credentials available on the system. (Optional)

![New custom migration target modal](images/new-target-repository.png?raw=true "New custom migration target modal")

Clicking on create will create the new custom migration target and add it at the end of the card list on the custom migration targets view. Additionally, a toast alert will appear to let the user know that the target was successfully created. The toast will contain a "Take me there" button inside of it. Clicking on that button will auto scroll the page to put the new card in view. This is especially beneficial when the list of cards is long.

![Custom migration targets view](images/new-target-success-toast.png?raw=true "Custom migration targets view")

###### Editing custom migration targets

Existing custom migration targets can be edited by clicking on the kebab at the top right corner of the corresponding card and selecting the option "Edit":

![Edit custom migration target modal](images/edit-target-kebab.png?raw=true "Edit custom migration target modal")


This will open a modal window displaying all the fields for the custom migration target. The associated image can be replaced by clicking "Clear" on the file upload component and uploading a new one. The value for the *Custom Rules* field can't be changed, and the option that wasn't selected when the user created the target will appear disabled. If the user selected the "Upload manually" option, the list of files will appear with a completion of 100%, allowing the user to delete any of them and upload additional ones with the same procedure:

![Edit custom migration target modal](images/edit-target-manual.png?raw=true "Edit custom migration target modal")

If the user selected the "Retrieve from a repository" option, the list of fields with their corresponding values will be displayed, allowing to edit all values:

![Edit custom migration target modal](images/edit-target-repository.png?raw=true "Edit custom migration target modal")

###### Deleting custom migration targets

Existing custom migration targets can be deleted by clicking on the kebab at the top right corner of the corresponding card and selecting the option "Delete".

#### Changes in the Analysis Configuration Wizard

##### Related Use Cases

- [ACW001](#ACW001)
- [ACW002](#ACW002)
- [ACW003](#ACW003)


##### Involved Personas

- [Administrator](#administrator)
- [Architect](#architect)
- [Migrator](#migrator)


##### Description

###### Custom migration targets in the set targets screen

All custom migration targets will appear in the "Set targets" screen from the analysis configuration wizard in the order that is determined in the custom migration targets view from the Administrator perspective. All custom migration targets will appear with a label "Custom" in the top right corner of their corresponding card to distinguish them from the default targets.

![Set targets screen](images/analysis-wizard-custom-target.png?raw=true "Set targets screen")

###### Allowing to retrieve custom rules from a repository

The custom rules configuration screen will be changed to accommodate the possibility of retrieving custom rules from a repository instead of manually uploading them. Both options will be mutually exclusive, and will be represented onscreen by two tabs, "Manual" and "Repository", with the first one being the default:

![Custom rules screen](images/custom-rules-default.png?raw=true "Custom rules screen")

The contents of the "Manual" tab will be exactly the same as [the previous custom rules configuration screen in the wizard](https://github.com/konveyor/enhancements/tree/master/enhancements/tackle/tackle-windup-integration#custom-rules):

![Custom rules screen](images/custom-rules-manual.png?raw=true "Custom rules screen")

If the user clicks on the "Repository" tab, a form with the following fields will be displayed:

- **Repository type**: Dropdown. Values:
  - Git
  - Subversion
- **Source repository**: String.
- **Branch**: String. (Optional)
- **Root path**: String. (Optional)
- **Associated credentials**: Dropdown. It will display a list with all Source credentials available on the system. (Optional)

![Custom rules screen](images/custom-rules-repository.png?raw=true "Custom rules screen")

Uploading a file on the "Manual" tab or entering data on any of the fields of the form in the "Repository" tab will automatically disable the other tab, that will remain disabled unless all data is cleared in the current tab.


### Implementation Details/Notes/Constraints

- No changes in the Windup CLI should be required to implement this enhancement.
- The retrieval of custom rules either manually uploaded by the user or from a repository should be managed by the Windup addon.
- There could be multiple custom targets selected, so the Windup addon should be capable of retrieving files from multiple sources, group them and pass them to the Windup CLI.
- Custom rules can be passed on the analysis wizard on top of the selection of custom migration targets. Those files should be passed to the Windup CLI as well.
- When retrieving files from a repository, the following needs to be taken into account:
  - The Windup CLI won't recurse into subdirectories, so only files in the root path will be taken into account.
  - The addon should validate that all files have a ".windup.xml" suffix. This is a requirement from the Windup CLI, so this validation is a way to make sure that any rules file that gets passed will not be ignored by the CLI. If a file doesn't have the suffix, it would be ignored right away by the Windup CLI, so the addon should take the responsibility of notifying the user about it. For each file that is not compliant with the file name format, the following line will be logged: 'File SFILENAME doesn't comply with the ".windup.xml" suffix and will be ignored during analysis'.




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
