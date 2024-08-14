---
title: neat-enhancement-idea
authors:
  - "@shawn-hurley"
reviewers:
  - "@aufi"
  - "@mansam"
  - "@ibolton336"
  - "@djzager"
approvers:
  - "@jortel"
  - "@eemcmullan"
  - "@sjd78"
  - "@rromannissen"
creation-date: 2024-08-13
last-updated: 2024-08-13
status: provisional
---

# dismiss issues that no longer of impact 

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]


## Summary

Analysis results can often lead to issues being found that the architect either deems as not urgent or unnecessary to change. 
This can be because there is a non-code fix (for example: using a PV for file access) or it is not a warning of a future deprecation. 
In both of these cases, the architect would not want a migration factory team to be focused on making these changes for the applications that have been analyzed. 
The architect may want this to work across the portfolio meaning for a given application (source repository, binary) when running with kantra or in the hub, the user would want the issues that have been dismissed to persist. 
The user will also want to be able to see these issues, even after they are ignored. 

## Motivation

An architect may consider that certain incidents are not on the migration path, and because of this, they will not want to see them from the incidents/issues that are shown. For a given application, they may have a solution in place that is not fixed in code or they could consider the issue not on the critical path. This is about helping expose only the most important issues first.

### Goals

1. Allow users to filter incidents that they do not believe are important for their migration.
2. Persist this filter across runs of analysis in both kantra and the hub.
3. Allow users to see the dismissed analysis report and via the UI (both tackle-ui and static-report)

### Non-Goals

1. Auto-detect what should be filtered out.
2. Hub to Kantra communication

## Proposal

### User Stories [optional]

#### Story 1

As an architect, I would like to dismiss incidents in the tackle-ui, and they are removed from the main view.

#### Story 2

As an architect, I would want the dismissed incidents to persist over subsequent runs of the analysis.

#### Story 3

As an architect, I would like to be able to export the dismissed rules to be used with a local run of kantra.

#### Story 4

As an architect, I would like to see the dismissed issues for an analysis. 

#### Story 5

As an architect, I would like to export a set of dismissed issues from a static report to be used in subsequent analysis runs of kantra.

#### Story 6

As an architect, I would like to import a list of dismissed issues to the hub for subsequent analysis runs.
### Implementation Details/Notes/Constraints [optional]

This will be implemented across the portfolio, and the implementation will need to be consistent across these repositories. Because of this, everything is already using the analysis results as a shared source of truth for a given analysis, so we should implement the underlying dismissal in this repository.

#### Analyzer-LSP

In the analyzer, we will implement a new input format, that identifies dismissed issues by a combonation of:
rule-id, file uri and line number. Today, this combination is already unique for an incident in the analysis run, so it is a good combonation that is already tested to represent a single issue. The command should be able to take more than one dismissed incident files(DIF from here on out).

The analysis engine, will create a new output list, `dismissed`, that will contain for each combination found above, the same incident as if it were a violation/incident. The output will be the exact same as if it was in the violations.

#### Kantra & Static Report

For kantra we will need to do two things. The first thing, is enhance the static reports to be able to dismiss incidents as well allow a user to download the DIF to the file system for future use. 
The second thing we will need to do, is have a consistant way to map applications to the DIF. This file will need to be able to be shared between the hub and kantra.

##### Labeling Applications for Shared Reconigtion

For java applications the most obvious answer is the maven identifier for `<group>.<artifact>.<version>` and potentially ignoring the version in this case. This will not work for other applications or platforms. In the hub we have an identifier for an Application (the Application ID) that would consistant for a single hub instance but that would mean nothing to the kantra user, who only has a filepath to either source code or a binary. Mandating a connection the hub in this case would be disaster as primary use case of kantra is to test out analysis before deploying a hub. 

Given all of these constraints specified above, I do not believe that we can do anthing automatically, we will need user input.

##### Kantra specific application and dismisssed incidents label

One goal should be that a user, once they make these dismissals, they will not need to do this on each subsequent run. To achieve this, we should have a new subcommand that will manage the DIF for a given application. 

```kantra dismissed-incidents

Application Path        Dismissed File
abs-path-to-file        <path to DIF>
abs-path-to-file        <path to DIF>
abs-path-to-file        <path to DIF>
```

On the run of an analysis, kantra will look at the abs path to input and determine if there is a DIF(s). We will then  use those files during the analysis of the applications. 

Kantra will also allow a one time use of the DIF(s), using a new flag on analyze:

```
kantra analyze -i <path> -t <targets> -o <out> --overwrite --dismissed-incidents <path> --dismissed-incidents <path>
```

Note that this WILL NOT tie the DIF's to the application input path. 

##### Static Report

The static report will be enhanced in three ways:
1. It will now understand the new output of `dismissed` from the analysis results. This make them viewable in the UI, under a new tab or navigation window. 
2. It will allow NEW incidents to be dissmissed. This will remove them from the main incidents view, and move them to the dimissed view. 
3. The dismissed view, will allow the user to download the DIF. 

#### Hub and Tackle UI

The Hub and Tackle UI will be enhanced in a couple of ways to achieve a similar UX like the static reports and kantra.

##### Hub

The hub will now allow a DIF to be attached to application. The analyzer-addon will be responsible for pulling this file when running anlysis. This will follow a similar pattern to other application information that is passed to the addon, and the pulling of the file will follow a similar pattern to pulling the rulesets. (@jortel can you verify that this makes sense or how you would do it?).

The Hub will also have a to dynamically create and append this DIF as the UI dismisses incidents. To achive this, I propose a new field on incidents, called dismissed that is a boolean. This new incident update API will allow for the UI to tell the hub when an incident has been dismissed. When this happens the hub will need to:
1. Move the incident to dismissed
2. Add the incident to the either already existing DIF or create one and add it.

For the Hub it may make sense that the DIF is actually a new table in the database, and it is generated on retrieval. I think this is an open question to be discussed with the Hub team.

##### UI

The UI will now have a new button to add to a given incident page. This button will allow the user to dismiss an incident. 
The UI will also need to add a new page for a given application, that will allow you to see the "dismissed" incidents for that application. 
The UI will also need to have the ability to download the DIF. This will allow for the user to give to someone running kantra locally.

### Security, Risks, and Mitigations

All of the API's and feature related to dismissal in the Hub/UI case must be only for the architect/admin user. The migration user should not be able to dismiss incidents. 

The migration user should be able to see the dimissed incidents as well as download the DIF.

## Design Details

### Test Plan

Testing will be focused on e2e flows. We will create a new e2e that will:
1. Create an intial DIF file that is known to filter out some incidents for an application. I think first pass should be tackle-testapp filtering out local-file incident.
2. We will run analysis via the hub and verify that the incidents are dismissed.
3. We will then run kantra with the DIF, and verify the same.

We will also create another test case that will:
1. Create an application and run analysis in the hub.
2. Dismiss an incident and download the file
3. Use the DIF in an kantra command and make sure that the incident is dismissed.

### Upgrade / Downgrade Strategy

As this will be a new feature, the upgrade strategy will be handled with a combination of sql migrations and new container images.

The downgrade will follow a similar approach, the database will need to be rolled back, we will lose all the data about dimissed incidents. Kantra and analyzer-lsp will be unable to handle DIF files and the arguements will error.


