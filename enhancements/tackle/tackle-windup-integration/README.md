---
title: tackle-windup-integration
authors:
  - "rromannissen"
reviewers:
  - "jortel"
approvers:
  - "jortel"
creation-date: 2021-11-19
last-updated: 2022-06-14
status: implemented
see-also:
  -   
replaces:
  -
superseded-by:
  -
---

# Tackle Analysis: Tackle Hub integration with Windup


## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [X] Design details are appropriately documented from clear requirements
- [X] Test plan is defined
- [X] User-facing documentation is created

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

> **Answer**: Tackle 2.0 will use the Windup CLI wrapper approach, leveraging the new addon architecture. Once the Tackle Analysis API is mature enough, two flavors of this integration will be made available for users to decide the most suitable approach considering their potential resources constraints.


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

It is important to note that in most cases, analyzing the dependencies an application might have is more important than the application source itself, as these dependencies often include corporate frameworks that are common to many applications and can be used to classify them in application types. Almost in all scenarios the key to large scale migrations resides in migrating the corporate frameworks first during the pilot phase to have a common ground to industrialize the process for the rest of the application portfolio. Because of this, it is essential to include the analysis of dependencies even when dealing with just source code instead of binaries that might have these dependencies embedded.

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

#### Architect

A technical lead for the migration project that can create and
modify applications and information related to it. The Architects don’t need to
have access to sensitive information, but can consume it.

#### Migrator

A developer that should be allowed to run analysis,
but not to assess, create or modify applications in the portfolio.


### User Stories

#### Analysis Configuration

##### AC001

*As an Architect/Migrator I want to be able to define the source for application analysis:
source code or binary.*

##### AC002

*As an Architect I want to be able to configure if Maven dependencies are analyzed as binaries
 when using source code as source for application analysis.*

##### AC003

*As an Architect/Migrator I want to be able to use Git, SVN and Maven repositories as input
for an analysis.*

##### AC004

*As an Architect/Migrator I want to be able to upload one binary from my workstation
as input for the analysis of a single application.*

**Acceptance criteria**

- The dialog to upload appears if the persona selects the `Upload a local binary` in the `Source for analysis` drop-down list
- The dialog to upload must state the maximum allowed size in MB
- The dialog to upload must give an error message if the binary exceeds the maximum size
- The dialog to upload must distill down the set of extensions to the one allowed (jar, war, ear)
- The dialog to upload must have a progress bar to let the user how the uploading is progressing
- The dialog to upload must have the cancel icon to stop an upload
- The dialog to upload must have the delete icon for deleting an already uploaded binary
- The upload of binary is a mandatory step for each analysis execution. They will not be persisted within the Application Inventory (in iteration #1) for re-use in subsequent analyses.

##### AC005

*As an Architect/Migrator I want to be able to select the migration target for
my application.*

**Set transformation path acceptance criteria:**

- The primary list of targets will be presented as a series of buttons (panels) that the user can interact with (EAP, Containerization, Jakarta EE, Linux,OpenJDK, Camel, Quarkus, EAP, Spring Boot on RHR).
- The EAP target (version 7) will be selected by default with version 6 as the only other selectable value.
- Targets can be selected/deselected by clicking on the appropriate button.
- The target button border will show which targets are selected (i.e. which buttons have been pressed).
- The icon for each target must be intuitive to understand (the ideal being that for targets that represent upstream projects the official project icon is used) and legal for use within an upstream project.
- This list of icons will not include any downstream product icons.
- An analysis can not be invoked unless at least one target is selected. However the user is free to select target(s) via this screen and/or the advanced options.

**Target selection via advanced options acceptance criteria:**
- The Target field will be pre-populated with the target(s) selected via the ‘Set transformation path’.
- The user can enter an additional target(s) from the drop down list of shipped (predefined) targets (a consolidated list of all of the targets used within the shipped rulesets).
- The dropdown list will be presented in alphabetical order.
- An analysis can not be invoked unless at least one target is selected.

##### AC006

*As an Architect/Migrator I want to be able to specify the packages to be analyzed.
If no packages are specified, then all packages identified as 'application packages' will be analyzed unless the analysis of known Open Source libraries is requested*

**Acceptance criteria:**

- The user can navigate to the next or previous step in the analysis configuration without entering any package details.
- The user can manually add packages.
- The user can delete packages.
- The package/subpackage names must only contain alphanumeric characters and the period character.
- There will be no validation of overlap, so for example the user could legitimately enter mypackage.persistence mypackage.persistence.jdbc

##### AC007

*As an Architect/Migrator I want to be able to select the migration source for
my application.*

**Acceptance criteria:**

- The source field is optional.
- The user can enter source(s) from the drop down list of shipped (predefined) sources  (a consolidated list of all of the sources used within the shipped rulesets).
- The dropdown list will be presented in alphabetical order.

##### AC008

*As an Architect/Migrator I want to be able to
see the summary of the analysis configuration before triggering the analysis execution*

**Acceptance criteria:**

- It must be possible to see all of the application(s) selected for analysis.
- It must be possible to see all of the analysis configuration parameters (target(s), source(s), etc.)
- It must be possible to invoke the analysis, cancel or return to editing the analysis configuration.


##### AC009

*As an Architect/Migrator I want to be able to provide custom rules for the analysis.*

#### AC010
*As an Architect/Migrator I want to be able to select an application(s) for analysis.*

**Acceptance criteria:**

- It must be possible to select an application for analysis.
- It will not be possible to select an application that has an analysis status of In progress.


#### AC011
*As an Architect/Migrator I want to be able to specify which rule tags to be excluded in an analysis*

**Acceptance criteria:**

- It must be possible to select 0, 1 or many rules tags from the drop down list.
- The shipped collection of rules tags will be presented in alphabetical order.
- Validation will ensure that any excluded tags are not specified in the included list (above).


#### Analysis Output

##### AO001

*As a Architect/Migrator I want to get access to the result of the analysis (analysis report)
of each individual application.*

**Acceptance criteria:**

- For any application within the Application Inventory with an analysis status of Completed it must be possible to invoke an action to access the analysis static reports.
- The action is unavailable if the analysis status is not Completed.
- The reports will be opened in a separate tab of the browser.
- If a new analysis is invoked for a particular application then the static reports for an earlier analysis execution will no longer be accessible.  

##### AO002

*As a Architect/Migrator I want to be able to track the specific version of the source code
that has been analyzed.*

##### AO003

*As a Architect/Migrator I want to see metadata about the history of the analyses at a
summary level in terms of number of issues, story points, analysis configuration
(source, target and options), for each analysis execution*

**Acceptance criteria:**

- For each analysis execution that completes successfully a collection of data will be returned by the analysis process that will be persisted and exposed to the user via the Application Inventory. That data includes the analysis Id, analysis configuration parameters (target(s), source(s), etc.),  total number of story points and number of incidents by category (migration mandatory, migration optional, etc.).
- It must be possible to view the analysis history data for any application that has been analyzed. The data will be presented in Analysis Id sequence descending (so most recent to earliest sequence).
- The action to view the analysis history will only be available for applications that have at least 1 completed analysis.
Analyses in progress, or failed executions will not be included in the analysis history.

#### Analysis Execution

##### AE001
*As an Architect/Migrator I want to be aware of the status of the most recent analyses execution for each application.*

**Acceptance criteria:**

- The analysis status for each application is visible in the Application Inventory.
- From the Application Inventory the user can see the status of the last analysis invoked for each application (the relationship between Application to Analysis is
1 : 0,1).
- An analysis can not be triggered if the analysis status is In progress.
- An analysis that is In progress can be cancelled.
- The cancel action is not available for Analyses that have a status of Completed.
- For an In progress analysis the user can find out the number of analysis steps completed and the total number of steps.
- To be very clear the visibility of the data will not be limited to the analyses that have been invoked by a particular user. Rather ALL analyses invoked by ALL users.


### Functional Specification

#### Updates in the Inventory view

##### Related Use Cases

- [AC010](#AC010)
- [AE001](#AE001)

##### Involved Personas

- [Architect](#architect)
- [Migrator](#migrator)


##### Description

The new analysis feature requires a reorganization of the main inventory view, as there would be to many significant fields to be displayed on each application row with the current layout. The proposed solution is based in arranging fields and actions relative to analysis and assessment in different tabs. The application list displayed on each of the tabs will be the same, based on the selected criteria on the filtering system, but the information displayed in each row, the application details and the available actions on the kebab menu on each application row will change depending on the selected tab. The same will happen with the list of actions available on the buttons on top of the main table. Some row fields will remain fixed for all tabs. These fields are the following:

- **Name**: Application name
- **Description**: Application description
- **Business service**: Business service associated with the application.
- **Tags**: Number of tags assigned to the application.

Also, the expandable section for the application details will always display the following fields:

- **Tags**: List of tags assigned to the application.
- **Comments**: Comments about the application.

The rest of available fields for each application row will vary depending on the selected tab. For example, the assessment tab will contain the fields and actions currently available in the main inventory view from Tackle 1.x:

![Inventory Main View - Assessment Tab](images/inventory-tab-assessment.png?raw=true "Inventory Main View - Assessment Tab")

The list of fields for each application row in the Assessment tab are the following:

- **Assessment**: Status of the assessment associated with the application. Values:
  - Completed
  - In-progress
  - Not started
- **Review**: Status of the review associated with the application. Values:
  - Completed
  - In-progress
  - Not started

The list of assessment related fields in the application detail expandable section are the following:

- Proposed action
- Effort estimate
- Business criticality
- Work priority
- Risk
- Review comments

The rendering and types for each field values will remain the same as they are now.

Regarding the actions available in the kebab menu for each application row, the list would be the following:

![Inventory Main View - Assessment Tab](images/inventory-tab-assessment-kebab.png?raw=true "Inventory Main View - Assessment Tab")

This list of actions matches what is currently available in Tackle 1.x.

The available actions on the top menu for the Assessment tab are the following:

- **Assess**: Starts the assessment flow. It will only be enabled when a single application is selected, and disabled in any other case.
- **Review**: Navigates to the review view. It will only be enabled when an application that has its assessment status as "Completed" is selected.

The Analysis tab will include a different set of fields and actions:

![Inventory Main View - Analysis Tab](images/inventory-tab-analysis.png?raw=true "Inventory Main View - Analysis Tab")

The application row will include the following additional fields:

- **Analysis**: Status of the latest analysis associated with the application. Values:
  - Not started
  - Scheduled
  - In-progress
  - Canceled
  - Failed
  - Completed

The list of analysis related fields in the application detail expandable section are the following:

- Credentials
- Analysis

The analysis field will display an icon with a link to the HTML reports when the analysis status is "Completed". Otherwise, it will display "Not available".

Regarding the actions available in the kebab menu for each application row, the list would be the following:

![Inventory Main View - Analysis Tab](images/inventory-tab-analysis-kebab.png?raw=true "Inventory Main View - Analysis Tab")

Manage dependencies and Delete will remain doing the same as in Tackle 1.x. "Cancel analysis" will cancel an analysis that hasn't finished yet, and should only be displayed for applications that have an Analysis with status "Scheduled" or "In progress".

The available actions on the top menu for the Analysis tab are the following:

- **Analyze**: Starts the analysis flow. Will be enabled when one or many applications that don't have their analysis status as "Scheduled" or "In progress" are selected.

###### Additional filters

A new filter should be added to allow users to filter the application list by their analysis source (Source code or Binary).

#### Bulk Analysis configuration


##### Related Use Cases

- [AC001](#AC001)
- [AC002](#AC002)
- [AC003](#AC003)
- [AC005](#AC005)
- [AC006](#AC006)
- [AC007](#AC007)
- [AC008](#AC008)
- [AC009](#AC009)
- [AC011](#AC011)


##### Involved Personas

- [Architect](#architect)
- [Migrator](#migrator)


##### Description

If the user selects one or many applications and clicks on "Analyze", the analysis configuration flow will start. This flow is mostly the same for the case of analyzing a single application or several of them. In this point, we will see the common points for both cases, to later on describe the particularities of the single application analysis flow.

![Analysis mode](images/bulk_analysis.png?raw=true "Analysis mode")

###### Analysis mode

The first step of the analysis configuration flow will be to define the analysis mode, which refers to the type of input that will be passed to Windup for the analysis:

![Analysis mode](images/ba-analysis-mode.png?raw=true "Analysis mode")

For multiple applications, the available modes are the following:

- **Binary**: Retrieves the application binary from a Maven repository and executes the analysis with Windup in binary mode.
- **Source code**: Retrieves the application source code from a source code repository (Git or Subversion) and executes the analysis with Windup in source mode.
- **Source code + dependencies**: Retrieves the application source code from a source code repository (Git or Subversion), uses the POM file to retrieve all dependencies from a Maven repository and analyzes both the application and the obtained binaries with Windup in source mode.

The user will be able to choose the analysis mode with a dropdown with single selection and Binary as the default choice:

![Analysis mode](images/ba-analysis-mode-options.png?raw=true "Analysis mode")

If any of the selected applications doesn't have the required repository and credentials defined for the mode activea warning will be displayed:

![Analysis mode](images/ba-analysis-mode-check-credentials.png?raw=true "Analysis mode")

For example, this warning will be displayed if any of the applications doesn't have a source code repository defined and "Source code" or "Source code + dependencies" is chosen as the analysis mode. **All of these applications that don't have the proper setup will be left outside the analysis selection for the rest of the flow**

###### Transformation target

Once the analysis mode is defined, the next step in the analysis configuration flow is to set the transformation targets to be used. This is screen is similar to the current user interface available in the web distribution of MTA.

![Transformation target](images/ba-transformation-path.png?raw=true "Transformation target")

Each transformation target will be presented with a box that includes an icon and some description. The available values are the following:

- **Application server migration to JBoss EAP 7**
  - Text: *Upgrade to the latest Release of JBoss EAP or migrate your applications to JBoss EAP from other Enterprise Application Server (e.g. Oracle WebLogic Server).*
  - Windup target value: **eap7**
- **Containerization**
  - Text: *A comprehensive set of cloud and container readiness rules to assess applications for suitability for deployment on Kubernetes.*
  - Windup target value: **cloud-readiness**
- **Linux**
  - Text: *Ensure there are no Microsoft Windows paths hard coded into your applications.*
  - Windup target value: **linux**
- **Oracle JDK to OpenJDK**
  - Text: *Rules to support the migration to OpenJDK from Oracle JDK.*
  - Windup target value: **openjdk**
- **OpenJDK 11**
  - Text: *Rules to support the migration to OpenJDK 11 from OpenJDK 8.*
  - Windup target value: **openjdk11**
- **Jakarta EE 9**
  - Text: *A collection of rules to support migrating applications from Java EE 8 to Jakarta EE 9. The rules cover project dependencies, package renaming, updating XML Schema namespaces, the renaming of application configuration properties and bootstrapping files.*
  - Windup target value: **jakarta-ee**
- **Quarkus**
  - Text: *Rules to support the migration of Spring Boot applications to Quarkus.*
  - Windup target value: **quarkus**
- **Spring Boot on Red Hat Runtimes**
  - Text: *A set of rules for assessing the compatibility of applications against the versions of Spring Boot libraries supported by Red Hat Runtimes.*
  - Windup target value: **rhr**
- **Open Liberty**
  - Text: *A comprehensive set of rules for migrating traditional WebSphere applications to Open Liberty.*
  - Windup target value: **openliberty**
- **Camel**
  - Text: *A comprehensive set of rules for migration from Apache Camel 2 to Apache Camel 3.*
  - Windup target value: **camel**

The "Windup target value" entry for each one of the transformation paths above will define the value that gets passes to the Windup CLI with the --target flag. Multiple boxes can be selected in this screen, and consequently, multiple --target flags can be passed to the Windup CLI.

###### Analysis Scope

The next step will determine the scope of dependencies to be included in the analysis. In this screen the user will be presented with a radio button to select the scope, and a switch to activate packages exclusion. If that switch is enabled, the user will be allowed to enter packages manually:

![Scope](images/ba-scope.png?raw=true "Scope")

A text field will be displayed for the user to enter the desired package to be excluded. Clicking on "Add" will validate the input (against pattern /^[a-z]+(\.[a-z0-9]+)*$/g) and add the package to the list below. Clicking on the remove icon for each row on the list will remove the package. The list will be passed to the Windup CLI using the --excludePackages flag. Each entry of the list will be added separated by a space after the flag (eg, --excludePackages PACKAGE_1 PACKAGE_2).

The radio button will have the following values:

- **Application and internal dependencies only**: This scope will stick to the packages and artifacts that Windup doesn't recognize as known Open Source libraries. This means the application packages and all corporate libraries that might be used as dependencies. This option won't pass any additional flags to the Windup CLI, as this corresponds with its default behavior.

- **Application and all dependencies, including known Open Source libraries**: This scope includes all the application related packages from the previous option and all known Open Source libraries. This option passes the --analyzeKnownLibraries flag to the Windup CLI.

- **Select the list of packages to be analyzed manually**: This option will allow the user to set the analysis scope by manually selecting the list of packages to be analyzed. Clicking on this option will display a nested section to add packages similar to the one that gets rendered when the "Exclude packages" switch is enabled. The user will then enter the list of packages to be analyzed, and only these packages will get added to the scope. This translates into using the --packages flag in the Windup CLI to specify the list of packages to be analyzed separated by a space (eg, --packages PACKAGE_1 PACKAGE_2). Also, as the user could potentially select packages from both corporate libraries and known Open Source libraries, the --analyzeKnownLibraries flag should be passed as well.


![Scope](images/ba-scope-manual.png?raw=true "Scope")


Application and internal dependencies only will be selected and Exclude packages disabled by default.

###### Custom rules

In the next step of the analysis configuration flow, the user will optionally add custom rules to be used by the tool. This screen consists of a table with a filter and an action button to upload rules files:


![Custom rules](images/ba-customrules.png?raw=true "Custom rules")

At first, the table will appear empty. Adding custom rules is not mandatory, so the "Next" button should be enabled even if no rules have been added. If the user clicks on the "Add rule" button, and upload dialog will open:

![Upload custom rules](images/ba-customrules-upload-empty.png?raw=true "Upload custom rules")


After selecting the files to upload or dragging them to the component, the upload will begin and a component will be displayed to keep track of the upload progress:

![Upload custom rules](images/ba-customrules-upload-uploading-collapsed.png?raw=true "Upload custom rules")

Clicking on the arrow will expand a section with the upload details, including the list of files and a progress bar for each one of them. While the upload is in progress, the "Add" button will be disabled:

![Upload custom rules](images/ba-customrules-upload-uploading-expanded.png?raw=true "Upload custom rules")

When the user uploads one or several files, validation should execute automatically once the file has been fully received. Validation will consist on the following:

- The file has a ".windup.xml" suffix. This is a requirement from the Windup CLI, so this validation is a way to make sure that any rules file that gets passed will not be ignored by the CLI.

- The file is validated against the following XML Schema Definition: https://windup.jboss.org/schema/jboss-ruleset/windup-jboss-ruleset.xsd

> **Note**: Tackle should be capable of running in disconnected Kubernetes instances, so the XSD file should be available locally for the validator, as downloading it might not be an option.

Once the validation is done, each file will have a green or red bar depending on the validation results for each one of them. The "Add" button will be enabled if at least one file is valid.

![Upload custom rules](images/ba-customrules-upload-uploading-success.png?raw=true "Upload custom rules")

If any of the files is not valid, the red bar and an error message will be displayed under the corresponding file:

![Upload custom rules](images/ba-customrules-upload-uploading-failure.png?raw=true "Upload custom rules")

If any of the files is not valid clicking on "Add" will result in only the valid files being added to the custom rules list:

![Custom rules](images/ba-customrules-listfiles.png?raw=true "Custom rules")

Once files are uploaded, they will be presented in a table with the following columns:

- **File name**: Name of the XML rules file that was uploaded.
- **Source/Target**: Source and Target for the ruleset defined in the XML rules file. These can be obtained parsing the id attribute from the ruleset.metadata.sourceTechnology and ruleset.metadata.targetTechnology tags. As this entries are optional, they could not appear in the XML file, so an empty value would be displayed then. For example, if the sourceTechnology tag is not defined and targetTechnology is defined with an id with value "cloud-readiness", the column would display "/cloud-readiness". If both tags are not defined under the metadata tag, the column would display "/".
- **Number of rules**: Number or rules that the rules file contains. It can be calculated as the number of ruleset.rules.rule tags present in the rules file.

Each row will display a trash can icon. Clicking on it will delete the file corresponding to its row.

###### Options

After optionally adding custom rules, the user can configure a series of advanced options for the analysis.

![Advanced Options](images/ba-advanced-options.png?raw=true "Advanced Options")

This screen presents the following fields:

- **Target(s)**: List of targets configured for the analysis. It should be prepopulated with the values that were established in the transformation target screen. **If the user uploaded any custom rules on the previous screen, targets from these rules should appear selected as well**. The field should be a dropdown with multiple selection. The selected values should be passed to the CLI with the --target flag separated by spaces. (eg. --target eap7 cloud-readiness) Values should include the following fixed list **plus any custom target that might have been derived from the custom rules in the previous screen**. For example, if any custom rule in the previous screen had a target named *custom-target*, it should be added as well to the following list of available options and appear as selected in the list of selected targets:
    - camel
	- cloud-readiness
	- drools
	- eap
	- eap6
	- eap7
	- eapxp
	- fsw
	- fuse
	- hibernate
	- hibernate-search
	- jakarta-ee
	- java-ee
	- jbpm
	- linux
	- openjdk
	- quarkus
	- quarkus1
	- resteasy
	- rhr
- **Source(s)**: List of sources configured for the analysis. This helps narrowing down the list of rules that should be executed during the analysis. The field should be a dropdown with multiple selection. It should be empty by default, but **if the user uploaded any custom rules on the previous screen, sources from these rules should appear selected as well** The selected values should be passed to the CLI with the --source flag separated by spaces. (eg. --source eap eap6). Values should include the following fixed list **plus any custom source that might have been derived from the custom rules in the previous screen**. For example, if any custom rule in the previous screen had a source named *custom-source*, it should be added as well to the following list of available options and appear as selected in the list of selected sources:
    - agroal
	- amazon
	- artemis
	- avro
	- camel
	- config
	- drools
	- eap
	- eap6
	- eap7
	- eapxp
	- elytron
	- glassfish
	- hibernate
	- hibernate-search
	- java
	- java-ee
	- javaee
	- jbpm
	- jdbc
	- jonas
	- jrun
	- jsonb
	- jsonp
	- kafka
	- keycloak
	- kubernetes
	- log4j
	- logging
	- narayana
	- openshift
	- oraclejdk
	- orion
	- quarkus1
	- resin
	- resteasy
	- rmi
	- rpc
	- seam
	- soa
	- soa-p
	- sonic
	- sonicesb
	- springboot
	- thorntail
	- weblogic
	- websphere
- **Excluded rule tags**: The list of rule tags that should be excluded from the analysis. A text field will be displayed for the user to enter the desired tag to be excluded. Clicking on "Add" will add the tag to the list below. The selected values should be passed to the CLI with the --excludeTags flag separated by spaces. (eg. --excludeTags corporate-config-01000 corporate-config-01023).

###### Review analysis details

At the end of the analysis configuration flow, the user will be presented with a review screen with all the options that have been configured for the analysis:


![Analysis Review](images/ba-review.png?raw=true "Analysis Review")

The review screen will be informative only, and no inputs will be requested to the user. The information to be displayed is the following:

- **Applications**: List of applications included in the analysis.
- **Target(s)**: Target migration paths defined for the analysis.
- **Source(s)**: Sources defined for the analysis.
- **Scope**: Scope defined for the analysis.
- **Included Packages**: List of packages defined for the analysis. This section will only be displayed if the "Select the list of packages to be analyzed manually" scope was defined.
- **Custom rules**: List of custom rules defined for the analysis. "None" if no custom rules were defined.
- **Advanced options**: List of Advanced options defined for the analysis. "None" if no advanced options were defined.


#### Single Application Analysis configuration


##### Related Use Cases

- [AC001](#AC001)
- [AC002](#AC002)
- [AC003](#AC003)
- [AC004](#AC004)
- [AC005](#AC005)
- [AC006](#AC006)
- [AC007](#AC007)
- [AC008](#AC008)
- [AC009](#AC009)
- [AC011](#AC011)


##### Involved Personas

- [Architect](#architect)
- [Migrator](#migrator)


##### Description

The analysis configuration flow for a single application will be almost the same as the bulk analysis configuration flow defined in the previous point. It will start when the user selects a single application and clicks on "Analyze" under the Analysis tab in the Inventory view:

![Single Application Analysis](images/single-analysis.png?raw=true "Single Application Analysis")

The only change that differentiates the Single Application analysis configuration flow is that the in the Analysis Mode screen there will be an additional option to upload a binary for the selected application, emulating the previous behavior of the Web Console flavor of MTA:

![Single Application Analysis](images/sa-analysis-mode-upload-binary.png?raw=true "Single Application Analysis")

If the "Upload a local binary" option is selected, a new field will appear below the "Source for analysis" field:

![Single Application Analysis](images/sa-analysis-mode-upload-binary-empty.png?raw=true "Single Application Analysis")

The "Next" button will be disabled until a file is uploaded using the new field. Once the user drags a file or chooses one from its filesystem using the component a progress bar will be displayed and the upload will begin:

![Single Application Analysis](images/sa-analysis-mode-upload-binary-uploading.png?raw=true "Single Application Analysis")

A validation will be fired to check if the file has a correct extension. Allowed extensions are: war, ear, jar, zip. If the file doesn't have an allowed extension, an error message will be displayed and the "Next" button will remain disabled:

![Single Application Analysis](images/sa-analysis-mode-upload-binary-error.png?raw=true "Single Application Analysis")

If the extension is correct, the "Next" button will become enabled and the user will be allowed to continue with the analysis configuration flow:

![Single Application Analysis](images/sa-analysis-mode-upload-binary-uploaded.png?raw=true "Single Application Analysis")

The rest of the analysis configuration flow will be exactly the same as described in the [Bulk Analysis configuration](bulk-analysis-configuration) specification.

### Implementation Details/Notes/Constraints

#### Maven and Windup CLI commands to be executed by the addon

[AC001](#AC001) and [AC002](#AC002) effectively define three analysis modes: Source, Source + Dependencies and Binary. Each one of these modes will require a different set of Maven and Windup commands. The following points will provide examples of how these commands could potentially look like.

>**Note** - The mta-cli command is likely to be replaced with a windup-cli command instead for the upstream Windup release. This is yet to be confirmed.

##### Source mode

Potentially the most straightforward mode of the three. Once the repository has been cloned, and provided the mta-cli command is available in $PATH and custom rules are available in the customrules directory in $HOME:

```shell
mta-cli --input $HOME/<repo_root>/<path> --userRulesDirectory $HOME/customrules/  --target <target1> --target <target2> ... --target <targetn> <other_flags> --batchMode --overwrite --sourceMode --output <bucket>
```

If no custom rules are provided in the request, the --userRulesDirectory flag should be skipped.

##### Source + Dependencies mode

The approach for this will be to download all project dependencies in a directory leveraging the Maven dependencies plugin and setting that directory as an additional input for the source analysis. First of all, provided the settings.xml file has been sent in the request and is stored in the $HOME directory:

```shell
mvn -f $HOME/<repo_root>/<path>/pom.xml -s $HOME/settings.xml dependency:copy-dependencies -DoutputDirectory=$HOME/dependencies
```

If no settings file was provided in the request, the "-s" part of the command should be skipped.

Once all dependencies have been downloaded, we will use the following command, provided the mta-cli command is available in $PATH and custom rules are available in the customrules directory in $HOME:

```shell
mta-cli --input $HOME/<repo_root>/<path> --input $HOME/dependencies/ --userRulesDirectory $HOME/customrules/  --target <target1> --target <target2> ... --target <targetn> <other_flags> --batchMode --overwrite --sourceMode --output <bucket>
```

If no custom rules are provided in the request, the --userRulesDirectory flag should be skipped.

##### Binary Mode

For the binary mode we will leverage the Maven dependencies plugin to retrieve the artifact to be analyzed using its GAV coordinates (potentially adding packaging as well for EAR and WAR artifacts). For that, provided the settings.xml file has been sent in the request and is stored in the $HOME directory:

```shell
mvn -s $HOME/settings.xml dependency:copy -Dmdep.useBaseVersion=true -DoutputDirectory=$HOME/binaries -Dartifact=<group>:<artifact>:<version>:<packaging>
```

If no settings file was provided in the request, the "-s" part of the command should be skipped. Also, if no packaging is provided in the request, the format for the -Dartifact flag should be \<group>:\<artifact>:\<version>.

Now, with the artifact available in $HOME/binaries, we can execute the analysis, provided the mta-cli command is available in $PATH and custom rules are available in the customrules directory in $HOME:

```shell
mta-cli --input $HOME/binaries/ --userRulesDirectory $HOME/customrules/  --target <target1> --target <target2> ... --target <targetn> <other_flags> --batchMode --overwrite --output <bucket>
```

If no custom rules are provided in the request, the --userRulesDirectory flag should be skipped.

##### Upload a local binary Mode

When dealing with the analysis of a single application, users can upload a binary directly from their workstations. For that case, considering the artifact has been made available in $HOME/binaries, we can execute the analysis, provided the mta-cli command is available in $PATH and custom rules are available in the customrules directory in $HOME:

```shell
mta-cli --input $HOME/binaries/ --userRulesDirectory $HOME/customrules/  --target <target1> --target <target2> ... --target <targetn> <other_flags> --batchMode --overwrite --output <bucket>
```

If no custom rules are provided in the request, the --userRulesDirectory flag should be skipped.

#### Purging the local Maven repository

Since no Maven builds are going to be executed by the addon, it should be safe to delete the local Maven repository located in *$HOME/.m2*. None of the analysis directly consume that directory, as dependencies are copied to the $HOME/dependencies on the Source + Dependencies mode for each analysis.  

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
