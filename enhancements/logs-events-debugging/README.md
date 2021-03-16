---
title: logs-events-debugging
authors:
  - "@djwhatle"
reviewers:
  - "@jortel"
  - "@shawnhurley"
  - "@alaypatel07"
approvers:
  - "@jortel"
  - "@shawnhurley"
  - "@alaypatel07"
creation-date: 2021-03-08
last-updated: 2021-03-08
status: provisional
see-also:
  - "/enhancements/debug/README.md"  
  - "/enhancements/dynamic-pipeline/README.md"  
  - "/enhancements/progress-reporting/README.md"  
replaces:
  - n/a
superseded-by:
  - n/a
---

# MTC Logging and Events improvements for Debugging (2.0)

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

This is where to call out areas of the design that require closure before deciding
to implement the design.  For instance, 
 > 1. Should we move away from having a pkg level logger object to enable concurrent reconciles without disturbing the per-reconcile logged UID?
 > 2. Should we carry a patch in the Velero codebase so that it would log a migration name or UID  we would pass in through an annotation? 

## Summary

At the time of this writing, MTC doesn't consistently give _enough_ __understandable feedback__ for its users (cluster admins) who:

 - May have a __migration stuck waiting__ on something, but it's __unclear what is being waited on__
 - Are accustomed to using __logs and events__ for __granular feedback__ on controller activities
 - Would like to __understand what the controller is thinking__ at any given point in the migration
 - Would like to __understand all changes being made to their clusters__, e.g. any API request modifying etcd store
 - Expect to be able to easily filter for __all logs for a particular migration__ across all system components 
 - Need __context__ with all log messages so that they are able to locate whatever that log message is related to across __clusters__, __namespaces__, and __kinds__.
 - Can't be expected to interpret or fix their own problem with a __stack trace__, can likely only __send it to us__
 - May want to bump verbosity to get __lower-level log messages__ e.g. from controller-runtime if they are seeing network-level issues etc.
 - Don't necessarily know __how to recover__ from a problem with the system __even if__ they discovered what the problem was
 - Don't know the set of CLI commands required to __traverse our hierarchical API__ relationships looking for feedback on migration progress
 - Don't know __which Pod or CR to look at__  for an error message for the problem they are encountering
 - Don't know __which cluster to investigate on__, since migration operations span up to 3 (host, src, target) clusters per migration

## Motivation

Due to the large _matrix of cluster configurations, network conditions, and cluster activity_, it's likely that a user will need to troubleshoot some problems during a migration.

We should work to __enhance discoverability__ by giving our users feedback in the places they would naturally expect to find it. When a problem occurs, we should provide clues on where to look next. 

Cluster admins should be capable to __navigate our system with OpenShift knowledge they already have__.

### Goals

How will we know that this has succeeded?

- When a problem occurs, and the user looks at (logs, conditions, events, UI), they should naturally be able to discover enough context to dig further into the problem.
- It will be clear at all times what MTC (especially mig-controller) is "thinking" and  doing, and what is preventing further progress.
- The user has a view into the API of the system at that exposes its natural structure, and quickly pinpoints where current activity is happening and where warnings/errors exist.
- Additional logs and events reveal the internal decision making processes of mig-controller and other componenets (velero plugins, maybe others).


### Non-Goals

- If feasible, avoid complete re-architecture of how we pass errors around in mig-controller.
- Do not provide logs or events on things that will never be useful in a troubleshooting situation

## Proposal

### Logging 

- For our controllers that carry out migration tasks (migmigration, dvm, dim, dism) we should log
  - At the beginning of reconcile
  - When we make important decisions during the reconcile
  - When we create/update/delete resources on the user cluster during reconcile
  - When we exit the reconcile without proceeding (being sure to provide a reason why we can't proceed).

- Logging should make use of structured K/V pairs where possible instead of Sprintf. Keys need to be explicit. Don't use keys like `ns` or `name` if these will introduce ambiguity into the log message. Keep in mind that K/V pairs have probably already been passed to the logger so you may be overwriting existing K/Vs if you don't select a unique key name.

- For CRs that only reconcile occasionally such as MigCluster, MigPlan, MigStorage: we should log all conditions on each reconcile so user knows the state of these resources.
- For CRs that reconcile constantly while a migration is running such as MigMigration, DVM, DIM, DISM, DVMP: we should only log critical conditions, since logging all conditions would reduce the signal to noise ratio in the logs.

- Whenever logging about a reason why the controller can't proceed, be sure to include the `type`, `ns/name` and `cluster` of the resource blocking progress so that the user can go investigate without having to intimately know our API hierarchy.

### Risks and Mitigations

Risks 
 - Wrong amount of logging
   - Not enough logging: this will result in difficult or impossible to troubleshoot situations without attaching a debugger
   - Too much logging: this will result in low signal-to-noise ratio in logs, making troubleshooting more difficult
 - Logging sensitive information

Mitigations:
 - Developers should take care when writing a log message to think about how the user could use the information being logged.
 - We should not be logging _contents_ of secrets, ever, if we can help it.

## Implementation History

 - Large chunk of logs added throughout controller: https://github.com/konveyor/mig-controller/pull/992

## Drawbacks

Additional logs will make the codebase a bit less clean, but I think this is worthwhile for the enhnaced debugging experience that will result. 
