---
title: UI task log attachment list and download
authors:
  - "@sjd78"
reviewers:
  - "TBD"
approvers:
  - "TBD"
creation-date: 2024-05-15
last-updated: 2024-05-16
status: implementable
see-also:
  - "[tackle2-ui PR1072](https://github.com/konveyor/tackle2-ui/pull/1072)"
  - "[tackle2-ui PR1629](https://github.com/konveyor/tackle2-ui/pull/1629)"
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Enhance UI task log viewer to list and view attached files


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created


## Open Questions [optional]

The exact look and function of the details view still need to be finalized, but the base
direction and wireframes/mockups are complete.


## Summary

As described in [Issue159](https://github.com/konveyor/enhancements/issues/159), the main
Analysis Details modal displays a task's details, including its activity log.  Every task
has a list of attached/uploaded files.  These files can be anything but are typically command
output.  They are associated with a particular activity log line.  Currently, these extra
command output files can be embedded in the activity log portion appearing below their tagged
log line.  This can lead to a much longer activity log.


## Motivation

Allowing direct access to view and download files attached to an analysis task will
make it more convenient for a user to view and, if necessary, share the raw analysis logs.

### Goals

  - The goal of this enhancement is to provide individual access to log files attached to
    an application's current analysis task.

### Non-Goals

  - Displaying each log file type as anything other than json, yaml, or plain text is
    not necessary.


## Proposal

Since the files are explicitly listed, are available individually, and may be of a large size,
allowing them to be individually viewed is possible.  This enhancement is upgrade the UI
to:
  1. Show a list of the task's attached files
  2. Allow view and download of the files individually


### User Stories

Currently, the analysis details are be viewed on the UI in a modal window with an embedded
json/yaml viewer.  This view is available from the Application inventory table by either:
  - opening the kebab menu of an application that has an analysis task at least started,
    and selecting the "Analysis details" menu item
  - opening the details side drawer, selecting the Reports tab, and clicking the
    "View analysis details" link

The enhancement will retain these existing paths but change the resulting interaction.


#### Story 1

When any of the analysis details links are selected for a given application, the user
will be pushed to a new page.  Initially, the page will display the same detail log
as the current modal window.  Putting the display on a normal page will allow for better
use of screen space when viewing large files.

The page will have appropriate bread crumbs to indicate the path taken to view the
logs, including the application being viewed.  Only the most current task for an application
needs to be displayed, so the task information isn't necessary on the breadcrumb trail.

From the application table kebab, clicking Analysis here:
![Application Table](images/application-inventory-kebab.png?raw=true "Application Table")

Opens the analysis details / task log on a new page:
![On Page View](images/analysis-details-on-page.png?raw=true "On Page View")

The select list on the top left will allow viewing of:
  - The base task log (which should always be selected by default and be the first item in the list)
  - Each of the attached files

Expanded list to look like:
![On Page Select List](images/analysis-details-select-expanded.png?raw=true "On Page Select List")

Selecting an attached file:
![On Page Selected Attached File](images/analysis-details-log.png?raw=true "On Page Selected Attached File")

### Implementation Details/Notes/Constraints

Attachments listed in a task's yaml can currently be downloaded via the `files/` endpoint.

For example, given an analysis details yaml that looks like this:
```yaml
id: 1
createTime: 2024-04-29T17:28:19.699101894Z
name: Foxtrot Tango Whiskey.6.windup
addon: analyzer
data:
    mode:
        artifact: ""
        binary: false
        withDeps: false
    rules:
        labels:
            excluded: []
            included:
                - konveyor.io/target=cloud-readiness
                - konveyor.io/target=linux
                - konveyor.io/target=eap8
        path: ""
        tags:
            excluded: []
    scope:
        packages:
            excluded: []
            included: []
        withKnownLibs: false
    sources: []
    tagger:
        enabled: true
    targets: []
    verbosity: 0
application:
    id: 6
    name: Foxtrot Tango Whiskey
state: Succeeded
image: quay.io/konveyor/tackle2-addon-analyzer:latest
started: 2024-04-29T17:28:20.603742223Z
terminated: 2024-04-29T18:33:35.221855509Z
activity:
    - Fetching applications.
    ...snip...
    - Done.
attached:
    - id: 334
      name: ssh-agent.output
      activity: 2
    - id: 335
      name: git.output
      activity: 9
    - id: 336
      name: git.output
      activity: 11
    - id: 337
      name: windup-shim.output
      activity: 21
    - id: 338
      name: konveyor-analyzer.output
      activity: 52
    - id: 339
      name: konveyor-analyzer-dep.output
      activity: 54
```

The attached files can be fetched from an auth disabled UI server accessible on `127.0.0.1:8080` like this:
```
> curl "http://127.0.0.1:8080/hub/files/334"
SSH_AUTH_SOCK=/tmp/agent.1; export SSH_AUTH_SOCK;
SSH_AGENT_PID=14; export SSH_AGENT_PID;
echo Agent pid 14;
```


## Design Details

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).


## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.


## Drawbacks

N/A


## Alternatives

N/A

