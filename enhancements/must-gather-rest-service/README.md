---
title: must-gather-rest-service
authors:
  - "@aufi" / maufart
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2021-07-16
last-updated: 2021-07-16
status: implementable
see-also:
  - "/enhancements/forklift-2.2/must-gather-from-ui/README.md"
  - "https://github.com/aufi/must-gather-rest-wrapper"
---

# Must-gather REST Service

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

The must-gather REST service should provide a backend for must-gather execution from UI.

There is an OpenShift ```must-gather``` command, which gathers information about OpenShift cluster. In order to make it faster and easier for users, it is should be possible run it from UI.

This enhancement introduces a must-gather service which provides HTTP API to trigger, get status and download result must-gather archive.

## Motivation

The ```oc adm must-gather``` CLI command gathers information like logs, CRs, etc. from the OpenShift cluster.
In order to make the must-gather execution easier for users, it would be great to be able trigger it from UI and that needs a backend service which will execute the must-gather command on behalf of user's CLI.

It is expected to be used e.g. by customers for providing details on their support tickets easier, attaching details to upstream issues or by QE to easily collect information on a test suite failure.

### Goals

Provide a service which will do a backend for must-gather from UI.

The service should run must-gather for the user and serve result archive to be downloaded.

The service should be configurable to be shared across projects, even from different UIs.

### Non-Goals

This enhancement doesn't cover UI work. That should be implemented directly in UI of projects which will use it.

## Proposal

A new service service should be implemented. The service will provide HTTP API to trigger must-gather, get its status and retrieve archive with results. All must-gather command line parameters should be supported.

The must-gather execution itself will run within the must-gather service system (container). The wrapper service should be deployed to the OpenShift cluster by an operator.

### API

Start must-gather execution

```
$ curl -X POST -H "Content-Type: application/json" -d '{"image": "quay.io/maufart/forklift-must-gather", "timeout": "15m"}' http://localhost:8080/must-gather
```

Get must-gather execution (status field values: new, inprogress, completed, error)

```
$ curl  http://localhost:8080/must-gather/15
```

Download must-gather archive (available only if must-gather execution status == "completed")

```
$ curl http://localhost:8080/must-gather/15/data -o must-gather-archive.tar.gz
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
  "status": "completed",
  "image": "quay.io/maufart/forklift-must-gather",
  "image-stream": "",
  "node-name": "",
  "command": "",
  "source-dir": "",
  "timeout": "30m",
  "server": "",
  "exec-output": "[must-gather      ] OUT Using must-gather plug-in image: quay.io/maufart/forklift-must-gather\n[must-gather      ]..."
}
```

Initial implementation: https://github.com/aufi/must-gather-rest-wrapper

Demo video: https://www.youtube.com/watch?v=axwzNMYlxsQ

### Risks and Mitigations

The must-gather service needs OpenShift cluster admin account token to be able to execute must-gather command. The token should be passed by Operator, in similar way as for other migration-toolkit components.

HTTP API should use the same authentication as UI where is should be executed from. Must-gather could provide sensitive information about the cluster so needs to be handled in same way as admin UI is.

## Design Details

### Test Plan

The code has its test suite, automatically triggered on repo commit/PR open.

E2E test should be done as test of the UI implementation of must-gather from UI feature.

## Implementation History

2021-07-15 Initial implementation of a REST wrapper tool, that can run must-gather triggered via HTTP and provide result archive, was completed and is waiting for reviews - https://github.com/aufi/must-gather-rest-wrapper/issues/1

## Drawbacks

There is existing feature in OpenShift command line.

## Alternatives

User could switch to command line and do the same with existing set of features. Also other way of getting debug information from OpenShift than must-gather could be implemented.
