---
title: codify-and-apply-transformations 
authors:
  - "@shawn_hurley"
reviewers:
  - "@jmontleon"
  - "@alaypatel07"
approvers:
  - "@jmontleon"
  - "@alaypatel07"
creation-date: 2021-05-05
status: implementable
---

# Codify and Apply Transformations

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]


## Summary

Transformations are needed when moving kubernetes manifests between clusters,
between distributions or even to move particular objects to be reconciled by
new controllers. 
These transformations need to be sharable, testable, and idempotent.

## Motivation

The transformations are essential for users who want the choice of distribution, 
need to move applications between clusters, or want to move to a new deployment
 mechansim such as [ArgoCD](https://argoproj.github.io/argo-cd/).

### Goals

- Ability to run transformations on a set of kubernetes manifests
- Ability to codify the transformations and run them
- Ability to run transformations such that they can be applied at one time
- Transformations should be able to be determined if they conflict. 

### Non-Goals

- Any mechanism to ship/package or otherwise distribute transformations
- Transformations and application of transformations will not be run in a single command

## Proposal

### User Stories

#### Story 1

As a library user, I would like to create transformations that I define easily.


#### Story 2

As a library user, given a set of transformations, I would like to apply them 
to a yaml object and return a yaml object.

#### Story 3

As a library user, I would like to be able to create a plugin command easily
that conforms to the transform definition.



## Design Details

### Implementation

#### Packages

##### pkg/cmd

A library to implement a command that will conform to the defined
CLI interface for a plugin. 
The output will be a set of CLI flags and an options struct that can be used by
the users as well as used by further library functions and packages

##### pkg/transfrom

A library that will be responsible for helpers for creating and updating the
defined files for a transform directory.
This will work on file byte arrays to make all the implementation details of
the file, internal to the package.
We will add a versioning system for this file and internally to this package

##### pkg/apply

A package responsible for applying a transformation, defined as a byte array,
to create a new byte array that a user can use.
The user is responsible for what to do with the new byte array that will be
valid yaml.


### Test Plan

We will use a set of testdata files to verify that the transforms are created
correctly and that the application of the transformations is correct.
An e2e test will be created to use the cmd package and exercise the fully
defined interface that we expect to be present.