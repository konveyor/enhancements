---
title: plugin-discovery-download
authors:
  - "@shawn-hurley"
reviewers:
  - "@jmontleon"
  - "@alaypatel07"
  - "@sseago"
approvers:
  - "@sseago"
creation-date: 2021-07-28
status: implementable
---

# Plugin Discovery and Download

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

## Summary

As users create and develop plugins for transformations they want to be able to share them with other users. Users also want to be able to discover and download plugins. These are going to be very important features, as Red Hat and other vendors are going to want to distribute plugins that are important to them. We will want to start with a single place, but plan on expandablity for the future.

## Motivation

Providing a set of plugins that a user can choose to use will greatly increase the usability of the crane CLI transformation. The entire system is built for extensiblity and none of the logic (with exception of k8s) is included by default. This is to make this vendor agnostic and focused on upstream k8s.

### Goals

1. Users can find and use a set of plugins for what they need.
2. Developers can publish plugins, to a common repository to get data.
3. Plugins can be delivered downstream in a different way than upstream.
4. Plugins will be vetted to make sure that there is some quality control. We will need to have some CI in place to do this efficently.

### Non-Goals

1. Upgrade and downgrade of plugins 

## Proposal

### User Stories

#### Story 1

As a user of crane, I would like to easily find and use plugins for specific purposes.

#### Story 2

As a developer of a crane plugin, I would like to be able to publish a plugin for users to find them.

### Implementation Details/Notes/Constraints

I believe that we should follow a pattern like [krew index](https://github.com/kubernetes-sigs/krew-index). I believe that we do not have to start with the full type. For now we will have a smaller subset of fields. We will make sure of github releases and create a new folder called crane-index in the crane repo. This will be baked into the crane CLI for start. We will in a follow on enhancement allow for folks to provide their own indexes.

### Risks and Mitigations

Upgrading a plugin in this workflow would require an update in the CLI binary. For now this is acceptable, but we will come back and change this in the future.

## Design Details

### Test Plan

We will want to have a set of resources, that when the plugin is downloaded, and run against, produces a provided output. This simple flow will allow us to write a github action that will pull the provided plugin, an verify that it works. 

### Upgrade / Downgrade Strategy

We are explicitly ignoring this becuase it is all versioned by the CLI. We will need to figure out this process in a follow on when we more fully flesh out the plugin publishing.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

Try to to use some sort of container. This might be possible but would add un nessary permisions.

## Infrastructure Needed

We will need to create a github action to run the tests and to only run on this new folder.