---
title: neat-enhancement-idea
authors:
  - "jortel@redhat.com"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2024-04-25
last-updated: 2024-04-25
status: provisional|implementable|implemented|deferred|rejected|withdrawn|replaced
see-also:
replaces:
superseded-by:
---

# Support Multiple Languages


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

We are anticipating changes in the analyzer-lsp base image. We will split up providers into their individual images. In the enhancement, we mainly want to discuss: 
- How exactly we will run the addon Pod with user selected language providers? An important problem here is sharing the source location. 
- How will the UI filter targets/rulesets appropriate for provider(s) that will be selected.
- How could someone use their own provider image? This is not just overriding an existing provider image, but can we make it so that users can also test with new language providers that are not yet in the UI? 
- How can we be sure that provider containers be terminated when the addon has completed?
- How will credentials be injected into provider containers? Example: maven settings file.
- In future, we might need to run .NET on a windows host remotely.

## Motivation

Applications may be implemented by languages other than Java.

### Goals

- Determine which language providers to execute based on existing information about the application.
- User can run analysis without knowing the languages.
- Targets (rules) may be filtered by language.
- Targets (rules) may be pre-filtered by the UI when language is known.
- Language defaults to Java (until automated discovery feature added).

### Non-Goals

## Proposal

#### Filtering/Selection ####
Targets will be filtered and Providers will be selected by `Language` tag(s).  This will likely depend on the Application Discovery feature.

#### Addon Extensions ####

 An Addon Extension is primarily an optional container definition along with criteria defined by a new CR. Providers will be defined to the hub as an Addon Extension.  The first supported selection criteria will be by application tag matching.

#### Task Pod ####
The task pod will be configured with a _main_ container for the addon and a _sidecar_ container for each of the providers.  All containers will mount an `EmptyDir` volume into /shared for sharing files such as credentials and source code trees.

#### Custom Providers ####
Users may define new providers for testing using the UI by added a new Extension CR that is mapped to a new Language tag.  Then, add/replace the language tag so that extension (provider) is selected. 

Testing with the API is simpler.  The user would add the Extension, and submit an analysis tag using the API that references the extension explicitly.

### User Stories

### Implementation Details/Notes/Constraints

Constraints:
- Must support running the analyzer (addon) and providers in:
  - same container
  - separate containers
- Must support _remote_ providers running outside the cluster.
- Must ensure all containers are terminated when analysis has completed.
- Must provide the user with provider output/logs.
- Must provide credentials to through provider settings. Example: maven settings.
- Must support TCP port assignment which is injected to the provider settings.
- Must support provider specific settings.

Details: [Hub Issue](https://github.com/konveyor/tackle2-hub/issues/599) for hub/addon details.

### Security, Risks, and Mitigations

## Design Details

See: [Hub Issue](https://github.com/konveyor/tackle2-hub/issues/599) for hub/addon details.

### Test Plan

### Upgrade / Downgrade Strategy

## Implementation History

## Alternatives

## Infrastructure Needed [optional]

