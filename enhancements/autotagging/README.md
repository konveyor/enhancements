---
title: autotagging
authors:
  - "rromannissen"
reviewers:
  - "@jortel"
  - "@mansam"
  - "@jwmatthews"
  - "@ibolton336"
approvers:
  - "jortel"
creation-date: 2023-02-06
last-updated: 2023-03-09
status: provisional
see-also:
  -   
replaces:
  -
superseded-by:
  -
---

# Automated Tagging of Applications


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

- TBD

## Summary

One of the key assets in Konveyor is its tagging model that allows classifying applications in multiple dimensions. However, the task of tagging applications remains manual, which might not scale well when dealing with a large portfolio. The current enhancement aims at providing a way for addons to automatically tag applications based on the information they are able to surface. The starting point would be to leverage the technology stack information that Windup is able to collect during an analysis and translate that into tags that get automatically added to an application.


## Motivation

The enhancement provides the following advantages:

- Metadata about applications can be enriched in an automated fashion.
- Users will be able to surface information about the portfolio at scale.


### Goals

- Simplify the tagging process in Konveyor.
- Obtain verified data coming from application source code and binaries.
- Streamline tag representation in the Application Inventory UI.
- Enhance the Windup addon to export the technology reports from the analysis output as tags.
- Define standards for other addons to provide automated tagging.


### Non-Goals

- Create a canonical tag repository shared across different Konveyor projects.

## Proposal

### Personas / Actors


#### Architect

A technical lead for the migration project that can create and modify applications and information related to them.

#### Migrator

A developer that should be allowed to run analysis, but not to assess, create or modify applications in the portfolio.

### User Stories


##### AT001

*As an Architect or Migrator I want to be able distinguish the source of a set of tags*


##### AT002

*As an Architect or Migrator I want to be able to obtain tags out of an application analysis*


##### AT003

*As an Architect or Migrator I want to be able to configure if the automated tagging should happen during an analysis*



### Functional Specification

#### Tags visualization in the Application Inventory

##### Related Use Cases

- [AT001](#AT001)


##### Involved Personas

- [Architect](#architect)
- [Migrator](#migrator)


##### Description

The application profile for each application in the inventory has being redesigned, and now relies on a side drawer with multiple tabs to display all the information about a given application in an organized way:

![Application Profile Side Drawer](images/new-application-drawer.png?raw=true "Application Profile Side Drawer")

The *Tags* tab contains all the tags associated to the selected application, organized by the different sources they might be coming from and adding the Tag Type, now renamed to *Category*, for each tag. The user will be able to filter tags by their source and category.

![Application Profile Side Drawer](images/new-application-drawer-tags.png?raw=true "Application Profile Side Drawer")

A third tab will be made available to store information about reports associated to the application:

![Application Profile Side Drawer](images/new-application-drawer-reports.png?raw=true "Application Profile Side Drawer")

This will include the possibility of downloading HTML or CSV reports if the corresponding option is enabled in the General configuration menu from the Administration perspective as explained in [the RFE that requested this behavior](https://github.com/konveyor/enhancements/issues/91).

#### Changes in the Analysis Configuration Wizard

##### Related Use Cases

- [AT002](#AT002)
- [AT003](#AT003)


##### Involved Personas

- [Architect](#architect)
- [Migrator](#migrator)


##### Description

The analysis configuration wizard should include an additional option to allow users to configure whether they want the analysis to automatically tag the included applications or not. This will be displayed as an additional switch in the advanced options view from the wizard and enabled by default:

![Analysis Configuration Wizard](images/analysis-wizard.png?raw=true "Analysis Configuration Wizard")

### Implementation Details/Notes/Constraints

- The Windup CLI should be able to produce JSON output including the set of technology tags that were found during the analysis. Windup tags are classified in three levels, with only the last one being relevant. For example, JPA Entities are classified under Store/Java EE/Persistence, with only "Persistence" being the relevant category to be used as tag type/category in the Hub.
- The list of technologies and their corresponding categories that Windup can currently identify will be used to seed the tag types/categories and tags to be shipped with Konveyor out of the box.
- Custom rules can identify additional tags, so the Windup addon should be able to create tags if they are not already present in the system.
- The source for tags coming from a Windup analysis will be "Analysis".
- If an analysis is run on an application that was analyzed previously, the list of tags associated to the "Analysis" source will be recreated unless the user disables the automated tagging in the advanced options screen from the analysis wizard.
- When running an analysis in the "Source + Dependencies" mode, the Windup CLI produces output for two differentiated applications, one being related to the source code of the application itself and the other to its dependencies. All technologies from both applications should be created as tags for the application that was analyzed. In other words, technologies coming from both the application source code and its dependencies should be taken into account and tagged accordingly.



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
