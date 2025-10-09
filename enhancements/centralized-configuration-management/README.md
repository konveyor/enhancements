---
title: centralized-configuration-management
authors:
  - "rromannissen"
reviewers:
  - "@dymurray"
  - "@jwmatthews"
  - "@fabianvf"
  - "@jortel"
  - "@eemcmullan"
  - "@shawn-hurley"
  - "@djzager"
  - "@ibolton"
  - "@sjd78"
approvers:
  - "@dymurray"
  - "@jwmatthews"
  - "@fabianvf"
  - "@jortel"
  - "@eemcmullan"  
creation-date: 2025-10-09
last-updated: 2025-10-09
status: provisional
see-also:
  -    
replaces:
  -
superseded-by:
  -
---

# Centralized Configuration Management


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

- Since architects are expected to be in charge of managing *Analysis Profiles*, shouldn’t they be in charge of managing Migration Paths as well? Should the Migration Paths management view be moved from the Administration Perspective into the Migration Perspective?

## Summary

This enhancement aims at enabling organizations to enforce standards across the different components of the Konveyor architecture (Hub, CLI and IDE plugin) by centralizing configuration management in the Hub. From an organizational standpoint, this enhancement proposes enabling Konveyor to provide a **Platform Engineering approach to Migration and Modernization**, based on the following principles:
- Enable organizations to enforce standard practices and configurations to be easily followed by all developers involved in the migration initiative.
- Harmonize concepts and abstractions across all components.
- Enable connectivity between the local components (IDE Plugins, CLI) and the central Hub:
  - Share application data across components:
    - Discovery derived (Archetype, Target Platforms…).
    - Analysis results.
  - Synchronize configuration defined by architects in the Hub:
    - Analysis configuration.
    - LLM configuration.
  - Abstract developers from managing the lifecycle of custom rules in their local development environments.


## Motivation

With the inclusion of the IDE plugin, the detachment in the current component architecture has become apparent with some abstractions:
- Custom migration paths are aimed at simplifying the user interaction with rules, but are only present at the Hub.
- *Analysis Profiles* brilliantly simplify analysis configuration management, but are only present in the IDE plugin.
- There’s no way to enforce corporate standards for users of the IDE plugin and CLI. Architects invest a lot of time in modeling Tags, Archetypes, Target Platforms and Custom Migration Paths in the Hub to get the best representation of their organization and their intended migration plan, but those concepts don’t exist on the local/client side components.


### Goals

- Enable Architects to manage organization wide configuration in the Hub.
- Bring the *Analysis Profiles* abstraction to the Hub.
- Remove the need for developers to configure LLMs locally in their plugin.
- Streamline and simplify the distribution and usage of custom rules in the corporate environment.
- Allow developers to stay focused on migration (analysis and fixing of issues) without having to actively take care of staying aligned with corporate standards.
- Enable organizations to lock the Konveyor IDE plugin to only work with the configuration served by the Hub to enforce alignment with corporate practices and remove non meaningful options for the migration initiative.

### Non-Goals

- Establishing auditing mechanisms in the Hub about IDE or CLI usage.


## Proposal

### Personas / Actors

#### Administrator

The administrator for the tool that has access to some application-wide configuration parameters that other users can consume but not change or browse.

#### Architect

A technical lead for the migration project that can create and modify applications and information related to them.

#### Migrator

Developers in charge of migrating specific applications. They perform changes to the source code in their IDE environments. They can only consume configuration managed by Administrators and Architects to run the resolution flows for issues found by analysis.


### User Stories

TBD


### Functional Specification

TBD

### Implementation Details/Notes/Constraints

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
