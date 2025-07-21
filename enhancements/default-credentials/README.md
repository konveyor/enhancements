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

These test cases cover specific interaction flows for setting, changing, and using default credentials within the
application analysis workflow.

| #  | Test Title                                                                                                    | Steps to Reproduce                                                                                                                                                                                                                                                                                                                                                                                                     | Expected Result                                                                                                                                                                                                                                                                                |
|----|:--------------------------------------------------------------------------------------------------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1  | Verify the default credential indicator (star) on creation for **source control** credentials.                | 1. Navigate to the Credentials section. <br> 2. Create a new source control credential. <br> 3. During creation, check the `Default credential?` field. <br> 4. Fill in a valid username and password. <br>5. Save the credential.                                                                                                                                                                                     | The new credential row in the list should immediately display a **star icon** to signify it is the default.                                                                                                                                                                                    |
| 2  | Verify the default credential indicator (star) on creation for source **maven settings** credentials.         | 1. Navigate to the Credentials section. <br> 2. Create a new maven settings credential. <br> 3. During creation, check the `Default credential?` field. <br> 4. upload a valid maven settings xml file. <br> 5. Save the credential.                                                                                                                                                                                   | The new credential row in the list should immediately display a **star icon** to signify it is the default.                                                                                                                                                                                    |
| 3  | Analyze succeeds an application using default credentials from **_source control_** and **_maven settings_**. | 1. Create source control and maven settings credentials <br> 2. Set them as the **default**. <br> 3. Create a new application. <br> 4. Launch an analysis for the application **without** manually selecting any credentials.                                                                                                                                                                                          | The analysis should complete successfully.                                                                                                                                                                                                                                                     |
| 4  | Analysis should fail when no default credentials are available.                                               | 1. Ensure no default credentials are set. <br> 2. Create a new application. <br> 3. Launch an analysis for the application **without** manually selecting credentials.                                                                                                                                                                                                                                                 | The analysis should fail.                                                                                                                                                                                                                                                                      |
| 5  | Analyze with **Default Source Control** and **Manual Maven Settings** Credentials.                            | 1. Create and set **source control** credential type as default. <br> 2. Create a maven settings credential but do **not** set it as default. <br> 3. Create a new application. <br> 4. Manually select the maven settings credential. <br> 5. launch an analysis.                                                                                                                                                     | The analysis should complete successfully.                                                                                                                                                                                                                                                     |
| 6  | Analyze with **Default Maven Settings** and **Manual Source Control** Credentials.                            | 1. Create and set **Maven settings** credential type as default. <br> 2. Create a source control credential but do **not** set it as default. <br> 3. Create a new application. <br> 4. Manually select the source control credential. <br> 5. launch an analysis.                                                                                                                                                     | The analysis should complete successfully.                                                                                                                                                                                                                                                     |
| 7  | Re-Analyze an application after **removing** of default credentials.                                          | 1. Create source control and maven settings credentials <br> 2. Set them as the **default** <br> 3. Run analysis and wait for it to complete <br> 4. in the credentials page, make sure no default credentials is set for each type <br> 5. Run analysis on the same application                                                                                                                                       | The analysis should fail.                                                                                                                                                                                                                                                                      |
| 8  | Re-analyze an application after **setting** default credentials.                                              | 1. Create source control and maven credentials, but do not set them as default. <br> 2. Run analysis on an application, and wait for it to fail. <br> 3. On the credentials page, set the existing credentials as the default for each type. <br> 4. Run the analysis again on the same application without manually selecting credentials.                                                                            | The analysis should complete successfully.                                                                                                                                                                                                                                                     |
| 9  | Verify the behavior when changing the default credential for **source control**.                              | 1. Ensure a default source credential already exists (and has a star indicator). <br> 2. Begin creating a second source credential. <br> 3. Check the `Default credential?` field. <br> 4. Save the new credential.                                                                                                                                                                                                    | 1. A confirmation message (e.g., "Default credential will change") should appear on the creation popup **before saving**. <br> 2. After saving, the star icon should be **removed from the first** source control credential row and **appear on the new selected** source control credential. |
| 10 | Verify the behavior when changing the default credential for **Maven settings**.                              | 1. Ensure a default Maven settings credential already exists (and has a star indicator). <br> 2. Begin creating a second Maven settings credential. <br> 3. Check the `Default credential?` field. <br> 4. Save the new credential.                                                                                                                                                                                    | 1. A confirmation message (e.g., "Default credential will change") should appear on the creation popup **before saving**. <br> 2. After saving, the star icon should be **removed from the first** maven settings credential row and **appear on the new selected** maven settings credential. |
| 11 | Verify multiple, different types of credentials can be default simultaneously.                                | 1. Create a set of **Source Credentials** and check the `Default credential?` field. <br> 2. Create a set of **Maven Credentials** and check the `Default credential?` field.                                                                                                                                                                                                                                          | Both credential rows should display a star icon in the list, as each one is the default for its specific type.                                                                                                                                                                                 |
| 12 | Set the first **source control** default via Actions menu.                                                    | 1. Create a **source control** credential, but do **not** set it as the default during creation.<br>2. Find the new credential's row and click its "Actions" button.<br>3. Select the "Set as Default" option from the menu.                                                                                                                                                                                           | The star indicator should **appear** on the credential row.                                                                                                                                                                                                                                    |
| 13 | Set the first **maven settings** default via Actions menu.                                                    | 1. Create a **maven settings** credential, but do **not** set it as the default during creation.<br>2. Find the new credential's row and click its "Actions" button.<br>3. Select the "Set as Default" option from the menu.                                                                                                                                                                                           | The star indicator should **appear** on the credential row.                                                                                                                                                                                                                                    |
| 14 | Remove default status from **source control** via Actions menu.                                               | 1. Ensure a **source control** credential is set as the default and has a star indicator.<br>2. Find the default credential's row and click its "Actions" button.<br>3. Select the "Remove Default" option from the menu.                                                                                                                                                                                              | The star indicator should be **removed** from the credential row.                                                                                                                                                                                                                              |
| 15 | Remove default status from **maven settings** via Actions menu.                                               | 1. Ensure a **maven settings** credential is set as the default and has a star indicator.<br>2. Find the default credential's row and click its "Actions" button.<br>3. Select the "Remove Default" option from the menu.                                                                                                                                                                                              | The star indicator should be **removed** from the credential row.                                                                                                                                                                                                                              |
| 16 | Change default **source control** credential via Actions menu.                                                | 1. Create two separate **source control** credentials. ( e.g. `SC-1` and `SC-2`).<br>2. Set `SC-1` as the default credential upon creation.<br>3. Verify that the row for `SC-1` has the star indicator.<br>4. On the credentials list, find the row for `SC-2` (the non-default one).<br>5. Click the "Actions" button for `SC-2` to open the options menu.<br>6. Select the "Set Default" option.                    | The star indicator should be **removed** from the `SC-1` credential row and should now **appear** on the `SC-2` credential row.                                                                                                                                                                |
| 17 | Change default **maven settings** credential via Actions menu.                                                | 1. Create two separate **maven settings** credentials. ( e.g. `MVN-1` and `MVN-2`).<br>2. Set `MVN-1` as the default credential upon creation.<br>3. Verify that the row for `MVN-1` has the star indicator.<br>4. On the credentials list, find the row for `MVN-2` (the non-default one).<br>5. Click the "Actions" button for `MVN-2` to open the options menu.<br>6. Select the "Set Default" option.              | The star indicator should be **removed** from the `MVN-1` credential row and should now **appear** on the `MVN-2` credential row.                                                                                                                                                              |
| 18 | Analysis Should fail after **deleting** the default credentials.                                              | 1. Create the necessary source control and maven settings credentials and set them as **default**.<br>2. Launch an analysis on an application and wait for it to complete successfully.<br>3. Navigate to the Credentials page.<br>4. For each default credential, use the Actions menu to **Delete** it.<br>5. Verify the credentials are no longer in the list.<br>6. Launch a new analysis on the same application. | The analysis should fail.                                                                                                                                                                                                                                                                      |

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
