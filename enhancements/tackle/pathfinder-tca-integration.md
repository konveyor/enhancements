---
title: app-inv-pathfinder-tca-integration
authors:
  - "@rofrano"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2022-03-01
last-updated: 2022-03-15
status: provisional
see-also:
  - ""  
replaces:
  - ""
superseded-by:
  - ""
---

# App Inventory / Pathfinder / Tackle Container Advisor Integration

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

None  

## Summary

[Tackle Application Inventory](https://github.com/konveyor/tackle-application-inventory) is a tool for cataloging and tagging applications that are candidates for modernization / migration. [Tackle Pathfinder](https://github.com/konveyor/tackle-pathfinder) is a tool for determining what path to take to modernize workloads on Kubernetes. [Tackle Container Advisor](https://github.com/konveyor/tackle-container-advisor) (TCA) is a tool to help standardize technology descriptions and determine if a workload can be run in a container, and if so, what container should be used.

This enhancement proposes to have Tackle Application Inventory and Tackle Pathfinder integrate with Tackle Container Advisor for two purposes:

1. The technology standardization of TCA can be used to take application descriptions and create consistent tags in the Application Inventory

2. The output from TCA can be additional input to the report that Pathfinder generates to determine the disposition of a workload in the cloud by identifying if it is suitable for containerization and if so what container images to choose..

## Motivation

Creating tags and tagging applications can be time consuming and error prone. When tagging a few application it's not a lot of work but when tagging 1000's of applications for an enterprise migration to Kubernetes it can be a daunting task. Many times application descriptions are vague and use inaccurate terms like "rhel, db2, java ee, tomcat". TCA has the ability to turn that string into standardized terms like "RedHat Enterprise Linux, IBM DB2, Java J2EE, Apache Tomcat". These more standardized terms can be used to create tags and then tag applications consistently in the Application Inventory.

A prerequisite to deploying applications on Kubernetes is knowing if the application can be containerized. This is critical input to the Pathfinder report yet it is missing. TCA can provide an evaluation to determine if the technologies used by an application can be run in a container and which container to suggest be used when deploying the application. This will help the users of Pathfinder determine which workload disposition (i.e., retire, retain, rehost, replatform, refactor, rewrite) to use.

### Goals

- Enhance the capabilities of Tackle Application Inventory with the standardization technology from Tackle Container Advisor for the purpose of generating tags and tagging applications consistently.
- Enhance the reporting capabilities of Tackle Pathfinder with the container advisory technology from Tackle Container Advisor to recommend which applications can be containerized and which container images to use.

### Non-Goals

- None yet

## Proposal

This is where we get down to the nitty gritty of what the proposal actually is.

### Personas / Actors

#### Migration Architect

The Migration Architect is responsible for determining teh disposition of each application with regard to whether it should be retired, retained, rehosted, replatformed, refactored, rewritten.

### User Stories

#### Story 1

**As a** Migration Architect  
**I need** to be able to easily tag applications with their technologies  
**So that** I can categorize applications for migration based on the technologies that they use  

#### Story 2

**As a** Migration Architect  
**I need** to be able to understand which applications can be containerized  
**So that** I can determine the feasibility of deploying them onto Kubernetes

### Implementation Details/Notes/Constraints [optional]

#### Tackle Pathfinder Implementation

The proposed solution for Tackle Pathfinder integration is for TCA to create a Command Line Interface (CLI) that could be invoked as an Add-On to Pathfinder following it's add-on architecture. TCA is currently a Python microservice that runs as a service.

#### Tackle Application Inventory Implementation

It is not clear how TCA would integrate with Tackle Application Inventory. I'm not sure if the Application Inventory has similar extensibility as Pathfinder. We will need to do more work in figuring this out.

### Security, Risks, and Mitigations

**Carefully think through the security implications for this change**

Tackle Container Advisor is a stateless microservice. It retains none of the input data that is sent and records none of the output data that is returned.

As such it has minimal security implications.

## Design Details

### Test Plan

**Note:** *Section not required until targeted at a release.*

None yet.

### Upgrade / Downgrade Strategy

None yet.

## Implementation History

None yet.

## Drawbacks

None yet.

## Alternatives

None yet.

## Infrastructure Needed [optional]
