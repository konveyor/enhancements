---
title: kantra-analyze-multiple-inputs
authors:
  - "@aufi" (maufart)
reviewers:
  - "@dymurray"
  - "@brunoborges"
  - "@pranavgaikwad"
  - "@rromannissen"
  - "@shawn-hurley"
approvers:
  - TBD
creation-date: 2024-08-13
last-updated: 2024-08-14
status: implementable
see-also:
  - "https://github.com/konveyor/enhancements/issues/194"  
---

# Kantra analyze multiple inputs

This enhancement introduces feature of kantra to analyze multiple input applications with a single kantra command execution with results (issues&incidents) combined into a single static-report.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

- `--input-bin-dir` support and its relation to multiple `--input` args (or exclusivity of usage)

## Summary

Konveyor consists of the main cloud-native installation driven by _Hub_ that provides advanced features of application analysis execution including analysis task scheduling, parallel tasks executions. And more lighweight command line tool called _kantra_, that can analyze just a single input application on its command execution.

Users community requested add support for multiple inputs to kantra, that would match to previous _mta-cli_ tool multiple inputs feature. In order to accomplish that, kantra will be updated to accept multiple ```--input``` args and handle analysis for all those inputs.

## Motivation

This enhancement should close a feature gap to "original" mta-cli multiple input option and increase usability of kantra tool.

### Goals

Analyze multiple applications with a single kantra command execution.

### Non-Goals

Implement an analysis task scheduler within the kantra or parallel application analysis execution similar to Hub features.

## Proposal

Extend kantra to accept multiple ```--input``` args, execute all needed steps to analyze those input applications and provide static-report with results for all provided inputs.

### User Stories

As a user, I want to be able execute kantra analyze command with multiple inputs and get static-report with all provided inputs.

Example:
```
$ kantra analyze --input ApplicationA --input ApplicationB --input ApplicationC --output=<path/to/output/applicationsABC>
```

### Implementation Details/Notes/Constraints

### Security, Risks, and Mitigations

There is a possibility to overload local machine (CPU or memory) that executes the analysis. That could be caused by size of the application(s) itself or chosen rules. User need to keep in mind, that kantra analyze execution might consume significant resources on their machine. 

## Design Details

It would be nice to generate static-report after each input application analysis completion to have some static-report available before finishing all inputs or for a case that analysis crash or an unexpected termination.

### Limitations

All analysis options are the same for all input applications and there is no plan to implement different source/targets/... for different input applications.

### Test Plan

Run kantra analyze with multiple (different) input applications and check output for:
- static-report contains
- - all input applications
- - issues and incidents for each input application
- - issues and incidents for all input applications together
- output.yaml and optionally dependencies.yaml files for input applications look sane
- analysis.log file content to look sane

## Drawbacks

This feature overlaps with Konveyor Hub features that is preffered way to run Konveyor and brings a logic to kantra, that doesn't fit naturally to the tool itself (from engineering team point of view).

## Alternatives

### kantra```--batch```

There is already an option to analyze multiple input applications with `--batch` option and get static-report that contains issues&incidents for all applications. However, that requires call `kantra analyze` for each input application.

On the other hand, this solution has a minimal impact on kantra codebase complexity and maintenance.

Example:
```
kantra analyze --bulk --input=ApplicationA --output=<path/to/output/applicationsABC>
kantra analyze --bulk --input=ApplicationB --output=<path/to/output/applicationsABC>
kantra analyze --bulk --input=ApplicationC --output=<path/to/output/applicationsABC>
```
