---
title: must-gather-from-ui
authors:
  - "@aufi" / maufart
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2021-07-16
last-updated: 2021-07-28
status: implementable
see-also:
  - "https://bugzilla.redhat.com/show_bug.cgi?id=1944402"
  - "https://github.com/aufi/must-gather-rest-wrapper/issues/1"
---

# Forklift must-gather from UI

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

There is an OpenShift ```must-gather``` command, which gathers information about OpenShift cluster. In order to make it faster and easier for users, it is should be possible run it from UI.

Forklift UI should be able to trigger must-gather on selected objects (plan, VMs) and provide result archive.
The UI needs a backend service for the must-gather execution via API, so it needs to be implemented too.

## Motivation

When a VM migration fails, Forklift user needs quick access to migration debugging informations and other details provided by must-gather feature. It should help to identify where is a problem and to make a successful migration once the problem is resolved.

There is ```oc adm must-gather``` CLI command gathers information like logs, CRs, etc. from the OpenShift cluster.

To make it easy for user, it should be supported to trigger must-gather directly from Forklift UI and don't have to switch to CLI.

It is expected to be used e.g. by customers for providing details on their support tickets easier, attaching details to upstream issues or by QE to easily collect information on a test suite failure.

### Goals

User should be able to execute must-gather from Forklift UI and get archive with results.

### Non-Goals

Provide other custom debugging information than must-gather archives created by existing must-gather images used from CLI.

## Proposal

The Forklift UI should allow user to execute OpenShift must-gather and provide the result archive to user. To provide backend for the UI a new service service should be implemented. The service will provide HTTP API to trigger must-gather, get its status and retrieve archive with results. All must-gather command line parameters should be supported.

The Forklift must-gather should utilize targeted-gathering feature for:
- Plan
- VM


A Command parameter of must-gather to specify targets for gathering (e.g. ```PLAN=plan1 /usr/bin/targeted``` or ```VM=vm-name-from-plan NS=ns-name /usr/bin/targeted```) will be prepared on UI side into a string passed to must-gather-service API call.

The must-gather execution itself will run within the must-gather service system (container). Temporary storage for archives will be on the container filesystem. The wrapper service should be deployed to the OpenShift cluster by an operator.

### Data model

```json:something``` means exposed in the JSON API response

```form:something``` means to be accepted as a parameter from the API request

```
type Gathering struct {
	ID          uint      `gorm:"primarykey" json:"id"`
	CreatedAt   time.Time `json:"created-at"`
	UpdatedAt   time.Time `json:"updated-at"`
	CustomName  string    `gorm:"index" form:"custom-name" json:"custom-name"`
	Status      string    `json:"status"` // Expected values: new, inprogress, completed, error
	Image       string    `form:"image" json:"image"`
	ImageStream string    `form:"image-stream" json:"image-stream"`
	NodeName    string    `form:"node-name" json:"node-name"`
	Command     string    `form:"command" json:"command"`
	SourceDir   string    `form:"source-dir" json:"source-dir"`
	Timeout     string    `form:"timeout" json:"timeout"`
	Server      string    `form:"server" json:"server"`
	ArchivePath string    `json:"-"` // Not exposed via JSON API
	ArchiveSize uint      `json:"archive-size"`
	ArchiveName string    `json:"archive-name"`
	ExecOutput  string    `json:"exec-output"` // Fields without form:"<name>" cannot be set via API by bind
}
```

### API

Start must-gather execution

```
$ curl -X POST -H "Content-Type: application/json" -d '{"image": "quay.io/maufart/forklift-must-gather", "timeout": "15m"}' http://localhost:8080/must-gather
```

Get must-gather execution (status field values: new, inprogress, completed, error). Get needs ID or a custom-name provided in must-gather trigger call.

```
$ curl  http://localhost:8080/must-gather/15
```

Download must-gather archive (available only if must-gather execution status == "completed")

```
$ curl -OJ http://localhost:8080/must-gather/15/data
```

List all must-gather executions

```
$ curl  http://localhost:8080/must-gather
```

Example of must-gather JSON object returned by API
```
{
  "id": 15,
  "created-at": "2021-06-30T15:19:17.514594773+02:00",
  "updated-at": "2021-06-30T15:23:05.415732774+02:00",
  "custom-name": "",
  "status": "completed",
  "image": "quay.io/konveyor/forklift-must-gather",
  "image-stream": "",
  "node-name": "",
  "command": "",
  "source-dir": "",
  "timeout": "30m",
  "server": "",
  "archive-size": 95334,
  "archive-name": "must-gather.tar.gz",
  "exec-output": "[must-gather      ] OUT Using must-gather plug-in image: quay.io/maufart/forklift-must-gather\n[must-gather      ]..."
}
```

Initial implementation: https://github.com/aufi/must-gather-rest-wrapper

Demo video: https://youtu.be/b5YTxyp6WyY

### Risks and Mitigations

#### UX/UI

The UI needs to be intuitive for user, will be discussed with UX team.

#### Security

Must-gather itself needs ```cluster-admin``` role. It could provide sensitive information about the cluster so needs to be authorized by OpenShift RBAC.

- HTTP API will require ```oauth``` token and verify if the token provides ```cluster-admin``` access to the cluster
- backend service needs a service account token/kubeconfig that will be ona PV mounted by Operator (in similar way as for other migration-toolkit components)

The must-gather wrapper service is an administration tool serving users with oauth token for ```cluster-admin``` role, so it does not accept requests from anonymous or unknown clients, so e.g. accepting arbitrary ```image``` as API parameter seems reasonable.

#### Resources consumption and limits

Compute power - the must-gather wrapper service is not a bottle-neck regarding to CPU. The gathering itself runs in a pod started from the must-gather image separate namespace in OpenShift outside the pod where the wrapper runs, so there doesn't seem to be need for amount of must-gather executions running in paralel.

Disk space - result archives are temporary stored directly on the filesystem of the pod where must-gather wrapper service is running. The archive size is unknown before the must-gather archive is downloaded to the must-gather wrapper pod. Reasonable maximum disk limit for the must-gather wraper service pod seems to me to be 10GB or more, however I do believe that the limit should be just space that OpenShift node could provide to the pod with must-gather, no must-gather wrapper service limits.

## Design Details

### Test Plan

The code has its test suite, automatically triggered on repo commit/PR open.

E2E test should have following scenario:

- After a VM migration from VMware or RHV to OpenShift Virtualization, click on button to trigger must-gather and wait until an archive is downloaded. It should not take more than 5 minutes. The archive should contain same information as an archive originated in OpenShift CLI must-gather execution.

## Implementation History

2021-07-26 Updated backend must-gather-rest-wrapper API to provide archive name, size and progressing exec-output. Added custom-name which could be used for query must-gather executions by UI without need for remembering execution ID.

2021-07-15 Initial implementation of a REST wrapper tool, that can run must-gather triggered via HTTP and provide result archive, was completed and is waiting for reviews - https://github.com/aufi/must-gather-rest-wrapper/issues/1

## Drawbacks

There is existing feature in OpenShift command line, so user could use CLI.

## Alternatives

User could switch to command line and do the same with existing set of features. Also other way of getting debug information from OpenShift than must-gather could be implemented.

In some other projects (Crane) there is quite detailed way to get details on Migration directly in UI.
