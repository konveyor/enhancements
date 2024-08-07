---
title: Gradle credentials
authors:
  - "@jmle"
reviewers:
  - "@pranavgaikwad"
  - "@rromannissen"
  - "@shawn-hurley"
approvers:
  - TBD
creation-date: 2024-08-07
last-updated: 2024-08-07
status: implementable
---

# Gradle credentials
This enhancement presents the possibility of adding support for specifying credentials for Gradle applications
to access private repositories.


## Release Signoff Checklist
- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary
This enhancement discusses the possibility of the user being able to specify credentials for the analysis of a
Gradle Java application, so that it can be built against private repositories.

## Motivation
This enhancement targets the analysis of Java applications which use Gradle and require credentials for being
built properly. This might be necessary due to the fact that some applications, especially corporate ones,
may have dependencies that are privately held in a private repository.

In order for Konveyor to properly analyze this sort of applications, the Java language server normally will try
to internally build the project. Also, Konveyor performs an extraction of the dependencies of the application
to show it to the final user, and to have the dependency information available to match rules against their
presence/absence.

### Goals
1. To make the end user able to upload their necessary credentials to their hub instance.
2. To enable the Java language server to properly build the application internally and parse it to do searches on it.
3. To enable Konveyor to show the full list of dependencies of the application, including the private ones.

## Proposal
The proposal will mainly include adding a new type of credential and its proper handling by the hub.

### User Stories [optional]

#### Story 1
As a user, I want to be able to upload my Gradle credentials, so that I can properly analyze my application.

#### Story 2
As a user, I want to be able to see my newly uploaded credential in my list of credentials, so that I can be aware
of the credentials I currently have.

#### Story 3
As a user, I want to be able to specify which credential file applies to the application I am analyzing, so that
I can have multiple credentials files for different applications.


### Implementation Details/Notes/Constraints [optional]
1. A new type of credential, named **"Gradle properties file"**, will be added to the "New credential" dialog form
that appears by clicking on the "Create new" button of the "Credentials" page under the "Administration" section.
2. Upon upload, the credential will be added to the list of credentials.
3. For analysis, the `gradle.properties` file will be added to the project folder.
4. If a `gradle.properties` file already exists, it can be placed under `$HOME/.gradle/`.


### Security, Risks, and Mitigations
- If the credentials is placed under `$HOME/.gradle/`, the analyzer must ensure that the already existing directory is
correctly handled.


## Design Details

### Test Plan
To test this, the `gradle` branch of the [tackle-testapp-public](https://github.com/konveyor/tackle-testapp-public/tree/gradle) can be used, which contains a dependency hosted in a private repository.
- The analysis should succeed.
- The full list of dependencies should appear.

This can be compared with the analysis of the same application under its main branch, which uses Maven. They must be similar.


### Upgrade / Downgrade Strategy
N/A

## Implementation History
N/A

## Drawbacks
N/A

## Alternatives
There are no alternatives, apart from the user directly placing its credentials under the `gradle.build` file, which
of course is not recommended.

## Infrastructure Needed [optional]
N/A
