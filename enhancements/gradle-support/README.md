---
title: Gradle support
authors:
  - "@jmle"
reviewers:
  - "@pranavgaikwad"
  - "@rromannissen"
  - "@shawn-hurley"
approvers:
  - "@pranavgaikwad"
  - "@rromannissen"
  - "@shawn-hurley"
creation-date: 2024-05-10
last-updated: 2024-05-10
status: implementable
---

# Gradle support

This enhancement presents the possibility of adding support for Gradle in the Konveyor Java provider.


## Summary

While not as widely used as Maven, many Java projects use [Gradle](https://gradle.org/) as their build tool. It is therefore desirable for Konveyor to be able to analyze these projects, and for this ability to be out-of-the-box and transparent for the user.

## Motivation

As Konveyor's user base increases, the need to support Gradle has become evident. For instance, Gradle has been traditionally used for Android projects. Leaving Gradle out of scope for Konveyor would mean leaving a potentially big chunk of users unable to use the tool.

### Goals

- The ability to analyze the code of Java Gradle projects
- The ability to identify the dependencies of Java Gradle projects
- The ability to analyze the dependencies of Java Gradle projects
- The ability to be as backwards compatible as possible in relation to Gradle versions

### Non-Goals

- Support for Groovy, Kotlin, or other programming languages usually associated with Gradle

## Proposal

The two main areas concerning support for Gradle are the analysis of the source code, done by Konveyor's rules engine through the usage of a language server, and the identification of the dependencies of the project (also required for certain features of the rules engine).

### User Stories

- As a user, I want to be able to analyze the source code of my Gradle Java project.
- As a user, I want to be able to identify the dependencies used by my Gradle Java project, both direct and transitive.
- As a user, I want the ability to analyze the source code of the dependencies of my Gradle Java project.
- As a user, I want to be able to analyze a project that uses an old Gradle version
- As a user, I want to be able to run Konveyor for my Gradle project within a constrained environment

### Implementation Details/Notes/Constraints

#### Java language server (JDTLS) and Gradle
The Java language server that Konveyor uses (the [Java Development Tool's Language server](https://github.com/eclipse-jdtls/eclipse.jdt.ls)) supports Gradle out-of-the-box through the use of [Buildship](https://github.com/eclipse/buildship), its Gradle integration. This means that the LS is automatically capable of loading the project and analyzing its files, just like it would with a Maven project.

#### Dependency identification
Gradle has [a built-in task](https://docs.gradle.org/current/userguide/viewing_debugging_dependencies.html) which enables the user to get a tree with all the dependencies, both direct and transitive, of the project. This tree can contain multiple levels of nesting, representing the depth of transitivity of each dependency.

#### Sources download
One of the features of the Konveyor Java Provider is the ability to download the source code of the dependencies of the project. These sources can be provided to the Language Server for analysis of their corresponding dependencies. Later on, the user can see the relevant snippets of code where a particular issue happened.

When analyzing Maven projects, this is done through an existing plugin. A similar plugin does not exist for Gradle, so a task or plugin will have to be created for this. This plugin or task will have to be as backwards compatible as possible.

#### Gradle wrapper
Unlike Maven, which can be considered quite stable, Gradle often releases new versions and updates. It is not mandatory for a Gradle project to specify in its configuration or settings which Gradle version it needs.

On the other hand, it is a widely used convention to include a [Gradle wrapper](https://docs.gradle.org/current/userguide/gradle_wrapper.html) in every Gradle project, which is able to download the required Gradle version and execute it automatically. Is this why this enhancement **will require the presence of the Gradle wrapper in every Gradle project being analyzed**.

#### Backwards compatibility
As mentioned previously, Gradle often releases new versions. This increases the difficulty of having backwards [compatibility](https://docs.gradle.org/current/userguide/compatibility.html) with older versions of Gradle.
Information about backwards compatibility will have to be added to the docs, potentially in the form of a table.

Since Konveyor is a Migration & Modernization framework, we should make the enhancement as *backwards compatible as possible*. Projects that use older versions of Gradle are expected.

### Security, Risks, and Mitigations

#### Constrained development environments
Some users might work within environments constrained in different forms, usually for security reasons. While the usage of Konveyor within such environments doesn't imply a security risk, the enhancement should provide the possibility of adapting to them. The hub will probably have to adapt to working in these conditions with Gradle.

## Design Details

### Test Plan

- e2e test to be added to the [tests repo](https://github.com/konveyor/go-konveyor-tests).

## Implementation History
TBD
