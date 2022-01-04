---
title: tackle-git-maven-integration
authors:
  - "rromannissen"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2021-11-20
last-updated: 2022-01-50
status: provisional
see-also:
  -   
replaces:
  -
superseded-by:
  -
---

# Tackle integration with Git, SVN and Maven repositories


## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions




## Summary

Application source code and binaries are typically stored and managed using corporate repositories. Most projects from the Tackle toolkit require access to source code to perform their function, and manually downloading source and/or binaries doesn't work well at scale. This enhancement proposes the integration of Tackle with the most widespread source and binaries repositories to automate the task of gathering source and binaries for them to be fed to the Tackle tools that might require them. Since credentials are required to access these repositories in most cases, a big part of this enhancement is dedicated to safely store and manage them.


## Motivation

Gathering source code and binaries is almost always a difficult task when working on large scale migration projects with customers. In most cases, the customer will provide the migration team with just a spreadsheet with application information, including URLs for source repositories or artifact repositories for each application or component.

Having application source and/or binaries when the assessment phase starts is key to have an analysis that provides a holistic view of the application portfolio and hence for the success of the assessment itself. It is true that the migration team could create scripts to consume that information and automate the download of source code and/or artifacts themselves, but the process would be time consuming and would require some automation skills that not AppDev Architects/Consultants have.



### Goals

- Enhance the Application entity to store information about source and binary repositories.
- Provide a way to store and manage credentials safely.
- Add an administration view that can accommodate application wide configurations and sensitive data management.

### Non-Goals

- Provide clone/checkout/download logic for addons to get access to the source or binaries. Each addon will be responsible of gathering source or binaries by itself using the credentials provided by Tackle Hub.

## Proposal

### Personas / Actors


#### Administrator

The administrator for the tool that has access to some application-wide configuration parameters that other users can consume but not change or browse. Example: Git credentials, Maven settings.xml files.

#### Architect

A technical lead for the migration project that can create and modify applications and information related to it. The Architects don’t need to have access to sensitive information, but can consume it. Example: Associate an existing credential to access the repository of a given application.

#### Migrator

A developer that should be allowed to run assessments and analysis, but not to create or modify applications in the portfolio.


#### Organization Administrator

A member of the operations or security team of the organization undergoing the adoption/migration initiative. The Organization Administrator manages the credential when there are security constraints that would prevent the Administrator from managing that information (For example when the Administrator belongs to the adoption team and that has been outsourced to a third party company).



### User Stories

#### Global Configuration

##### GLC001

*As an Administrator I want to be able to manage the global configuration for the tool through an administration panel.*

#### Credentials Management

##### CM001

*As an Administrator I want to be able to manage credentials so that they can be consumed by other regular users without compromising the security of the accessed systems.*

##### CM002

*As an Administrator I want to be able to manage Git User/Password credentials.*

##### CM003

*As an Administrator I want to be able to manage SSH Keys for Git access.*


##### CM004

*As an Administrator I want to be able to manage Subversion User/Password credentials.*


##### CM005

*As an Administrator I want to be able to manage settings.xml files so that they can be used to retrieve Maven artifacts.*


##### CM006

*As an Architect I want to be able to assign existing credentials to applications on a per application or bulk basis.*



#### Git Configuration


##### GC001

*As an Architect I want to be able to define per application or at bulk whether new changes in a repository are acquired via a pull or by performing a full clone on each analysis.*

##### GC002

*As an Administrator I want to be able to manage global Git configuration.*

##### GC003

*As an Administrator I want to be able to define the default behavior for whether new changes in a repository are acquired via a pull or by performing a full clone on each analysis.*

##### GC004

*As an Administrator I want to be able to configure if insecure Git repositories can be consumed.*

##### GC005

*As an Architect I want to be able to associate a Git repository to an application.*



#### Subversion Configuration

##### SC001

*As an Architect I want to be able to define per application or at bulk whether new changes in a repository are acquired via an update or by performing a full checkout on each analysis.*

##### SC002

*As an Administrator I want to be able to manage Global Subversion configuration.*

##### SC003

*As an Administrator I want to be able to define the default behavior for whether new changes in a repository are acquired via an update or by performing a full checkout on each analysis.*

##### SC004

*As an Administrator I want to be able to configure if insecure Subversion repositories can be consumed.*

##### SC005

*As an Architect I want to be able to associate a Subversion repository to an application.*

#### Maven Configuration

##### MC001

*As an Architect I want to be able to define per application or at bulk if an update should be forced for artifacts and dependencies on each analysis.*

##### MC002

*As an Administrator I want to be able to manage global Maven configuration.*

##### MC003

*As an Administrator I want to be able to configure the default behavior for forced updates for artifacts and dependencies on each analysis.*

##### MC004

*As an Administrator I want to be able to configure if insecure Artifact Repositories can be consumed.*

##### MC005

*As an Administrator I want to be able to know the size of the local artifact repository.*

##### MC006

*As an Administrator I want to be able to purge the local artifact repository.*

##### MC007

*As an Architect I want to be able to associate an artifact with an application.*

### Functional Specification

#### Administrator Perspective

##### Related Use Cases

- [GLC001](#GLC001)
- [CM001](#CM001)
- [GC002](#GC002)
- [SC002](#SC002)
- [MC002](#MC002)

##### Involved Personas

- [Administrator](#administrator)

##### Description

An Administrator Perspective is proposed as the way for the Administrator to access the administration panel for the tool. The user experience here should be consistent with the Administrator Perspective available in OCP 4.x, using the perspective switcher component:

<p align="middle">
  <img src="images/admin-selector.png" width="400" />
  <img src="images/dev-selector.png" width="400" />
</p>


For consistency sake with the user experience in OCP, the names “Administrator” and “Developer” will be kept for the two different perspectives available for the moment.

The Control Menu on the left will display the following options under the Administrator Perspective:
- Credentials
- Repositories
  - Git
  - Subversion
  - Maven
- Proxy

For the Developer perspective, the same options from previous releases remain:
- Application Inventory
- Reports
- Controls

#### Credentials Management

##### Related Use Cases

- [CM001](#CM001)
- [CM002](#CM002)
- [CM003](#CM003)
- [CM004](#CM004)
- [CM005](#CM005)

##### Involved Personas

- [Administrator](#administrator)


##### Description

###### Main View

The main Credentials Management view can be reached by clicking on the “Credentials” option from the Control Menu from the Administrator Perspective. This will display a view with a table with the list of available credentials, including the following fields:

- Name
- Description
- Type
- Created by

Each row will have an edit and a delete button. On the header of that table, a filter will allow the user to filter the table based on any of the previous fields as criteria. On its right, a “Create new” button will allow the user to create credentials.

![Credentials Main View](images/credentials-main.png?raw=true "Credentials Main View")

###### Create Credentials

By clicking on the “Create new” button, the system will open a modal window with the following fields:
- Name: String
- Description: String (Optional)
- Type: Dropdown, single selection. Values:
  - Source Control
  - Maven Settings File

![Create credentials](images/credentials-create-main.png?raw=true "Create credentials")

If the user selects “Source Control” in the Type field, the following fields will load on the modal window dynamically:
- User credentials: Dropdown, single selection. Values:
  - Username/Password
  - SCM Private Key/Passphrase

![Create credentials](images/credentials-create-sc.png?raw=true "Create credentials")


If the user selects "Username/Password" in the "User credentials" dropdown, the following fields will load dinamically:

- Username: String
- Password: String (Hidden)

![Create credentials](images/credentials-create-sc-up.png?raw=true "Create credentials")


If the user selects "SCM Private Key/Passphrase" in the "User credentials" dropdown, the following fields will load dinamically:

- SCM Private Key: String.
- Private Key Passphrase: String (Hidden)

![Create credentials](images/credentials-create-sc-spkp.png?raw=true "Create credentials")

The SCM Private Key field should allow the user to paste the Key content directly or drag and drop the private key file. If the user goes for the second approach, the file will be uploaded to the system and displayed on the field afterwards.

In case the user selects "Maven Settings File" in the "Type" dropdown, the following fields will load on the modal window dynamically:

- Settings File: String

![Create credentials](images/credentials-create-maven.png?raw=true "Create credentials")

As in the SCM Private Key field, the user should be able to paste the Key content directly or drag and drop the settings.xml file. Again, if the user goes for the second approach, the file will be uploaded to the system and displayed on the field afterwards.

There will be two buttons at the bottom of the modal window:
- **Save**: Validates the input and creates a new credential. SCM keys and Maven settings.xml files must be parsed and checked for validity. If the validation fails, an error message displaying “not a valid key/XML file” should be displayed.
- **Cancel**: Exists the modal window. If any data has been entered, the system will ask for confirmation.

###### Edit Credentials

The edit button on a credential from the list available on the main view will open a modal window similar to the one in the Create Credentials view. The fields displayed on the window will depend on the credential type and user credentials provided on creation. For example, for Maven credentials, the following fields will be displayed:

![Edit credentials](images/credentials-edit-maven.png?raw=true "Edit credentials")

The user won't be able to see the value of the Settings File field, with “Encrypted” displayed in the field value instead.

For source credentials using Username/Password authentication, the Password will appear hidden:

![Edit credentials](images/credentials-edit-sc-up.png?raw=true "Edit credentials")

Finally, for source credentials using SCM Private Key/Passphrase, the approach will be similar to what we have seen for Maven credentials:

![Edit credentials](images/credentials-edit-sc-spkp.png?raw=true "Edit credentials")

The SCM Private Key field will display "Encypted" as well, with the Private Key Passphrase field displaying a hidden value.

###### Delete Credentials

The delete button on a credential from the list available on the main view will delete a set of credentials after user confirmation. If the credentials have been associated with any application, the system will ask for additional confirmation with the following message:

![Delete credentials](images/credentials-delete.png?raw=true "Delete credentials")


### Implementation Details/Notes/Constraints

- Git uses both HTTPS and SSH as communication protocols, thus allowing authentication via an user/password pair and SSH keys respectively.
- Subversion mostly uses the SVN protocol which is secured using a user/password pair. Some repositories allow SVN over SSH protocol, so using a SSH key for authentication is also possible but less common.
- Accessing corporate artifact repositories requires a certain configuration on the Maven installation to use for that. This is achieved by including certain configurations on the local settings.xml file the installation uses, such as adding servers with user credentials, mirrors and repositories. Both Sonatype Nexus and JFrog Artifactory refer to this method as the way to consume them as central corporate repositories.
- Downloading source repositories doesn’t include the dependencies for a given application, as these are managed by Maven/Gradle and are downloaded as binaries at build time.


### Security, Risks, and Mitigations

- All credentials must be encrypted on the target datastore.
- Stored credentials can't be retrieved by any user once they have been created.

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
