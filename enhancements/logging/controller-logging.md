---
title: Improve mig-controller logging
authors:
  - "@alaypatel07"
reviewers:
  - "@djwhatle"
  - "@jortel"
  - "@eriknelson"
approvers:
  - "@djwhatle"
  - "@jortel"
  - "@eriknelson"
creation-date: 2020-09-10
last-updated: 2020-09-10
status: implementable
see-also:
replaces:
superseded-by:
---

# Cluster Application Migration (CAM) Controllers logging design

This enhancement proposes logging improvements to controller keeping the needs of developer in mind. 

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created and linked

## Open Questions [optional]


## Summary

The current controllers are implemented with `minimal logging` philosophy, i.e. only errors 
which are not expected are logged, and logged only once. While this is good in keep the logs
brief, _*it misses*_ a lot of information that can be helpful  to an engineer while debugging.
After implementing this enhancement, a developer will be able to browse 
through the logs and create a timeline of actions that mig-controller tried to perform. In a 
large scale migration this will be an essential pillar for the dev team to rely on. 


## Motivation

To enable developers and users dig through the logs and have enough information to understand
what actions were performed by mig-controller against the source and destination clusters.

### Goals

- Improving debugging experience by adding more logging information to mig-controller
- Alleviate pains and improve turnaround time in chasing bugs

### Non-Goals

- Aggregate logs from other pods in MTC architecture for better visibility

## Proposal

The current logging philosophy is a drift from most controller in 
kubernetes[[1]](https://github.com/kubernetes/kubernetes/blob/0fd10997dfee0f9ec2c413004bf93d310d76c84b/pkg/controller/deployment/deployment_controller.go#L581)
 [[2]](https://github.com/kubernetes/kubernetes/blob/0fd10997dfee0f9ec2c413004bf93d310d76c84b/pkg/controller/deployment/deployment_controller.go#L572)
and OpenShift
[[3]](https://github.com/openshift/cluster-etcd-operator/blob/0806334d716f294ebd22b0cf38be4ebce1e64b30/pkg/operator/clustermembercontroller/clustermembercontroller.go#L92) 
[[4]](https://github.com/openshift/cluster-etcd-operator/blob/0806334d716f294ebd22b0cf38be4ebce1e64b30/pkg/operator/clustermembercontroller/clustermembercontroller.go#L170). 

All our controller as essentially driving towards the desired state. Because
of this, most of the time a controller checks the state, there is no action required
(steady state). When there is no action required, the controller should still log 
important info but at level v(4) or more verbose. When controller does have to take
an "ensure" action, it should log at v(2). By default, the pod starts with  v(2), but
can be started with v(4) to help with debugging.  

Any warning you provide should be important enough to also emit an event.

### User Stories [optional]

- With this feature, a CAM user will be able to trace all the actions taken
  by the controller in the controller pod logs. Important information would also
  be available via events.

- CAM support and engineers will have faster turn around time for bugs in MTC
  controller    

### Implementation Details/Notes/Constraints [optional]

### Risks and Mitigations

NA

## Design Details

1. Use the `log.V(x)` to get the verbosity of the log.
1. Replace all `log.Trace(err)` to include some context of:
    1. what the controller was trying to do when error occured
    1. the resource and NamespacedName on which error occured (if applicable)
    1. if this is source cluster or destination (if applicable)
   Example: `log.Trace(fmt.Errorf("unable to update foo: fooNS/fooName in %s cluster", clusterURL))`
1. Go through all ensure actions and log after making create/update/delete call 
   to kube apiserver. If the action is important enough, generate an event
1. For any status condition change, generate a kube event,
1. For all logs that are `warning`, generate a kube event

### Test Plan

**Note:** *Section not required until targeted at a release.*

TBD

### Upgrade / Downgrade Strategy

This change will not pose an problems in upgrading/downgrading

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

[ ] update logs in migmigration-controller
[ ] update logs in migplan-controller
[ ] update logs in migstorage-controller
[ ] update logs in migcluster-controller
[ ] update logs in mighook-controller
[ ] update logs in mig-analytic-controller
[ ] update logs in migmigration

## Drawbacks

This will make logs little more verbose and devs will have to find effective tools 
to filter based on regex

## Alternatives

None

## Infrastructure Needed [optional]

NA