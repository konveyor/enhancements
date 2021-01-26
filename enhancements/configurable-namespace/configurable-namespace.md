---
title: controller-and-operand-namespace-seperation
authors:
 - "@shawn-hurley"
reviewers:
  - "@alaypatel07"
  - "@djwhatle"
approvers:
  - "@jortel"
creation-date: 2021-01-25
last-updated: 2021-01-25
status: implementable

---
# Controller and Operand Namespace Seperation

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

1. Should OLM provide the functionality of creating a both the namespace that the operator is installed in as well as the namespace that the operator acts on.

## Summary

The proposal is for creating two namespaces, one that the operator and mig-controller's are created in and then another namespace that the UI/Users will use to create and manage the Mig{Plan/Migration/Cluster etc} resources. This will be achieved by the MigOperator having namespace create permissions, and a new field on the MigrationController CR that will tell the migration-controller where to watch for resources. Migration controller will  have to take a configurable value for the namespace(s) it is watching.

## Motivation

As we look forward to some features that we may want to implement in the future, it is becomes apparent that the namespace that the operator and controller are deployed to should be different. One of the issues is that we are not currently using finalizers on resources, even when we need them to stick around. The DVM resource for instance can be deleted, throwing the mig migration into an state where it is waiting for a resource that has been deleted. What we would want is a finalizer to prevent the deletion so that the mig controller can gracefully handle this. The primary reason for not using finalizers, is because removing the namespace that has Mig* resources is not possible when the controller and those resources are in the same namespace. In addition to allowing for the finalizers to be used by the controller, it also will start to enable Migration controller watching a configurable namespace. This will help in our future plans of non admin work.

This pattern is used by many controllers in openshift already, most notably the core operators for api-server, DNS, and networking.

### Goals

List the specific goals of the proposal. How will we know that this has succeeded?

1. mig controller can use finalizers to prevent the deletion of resources, and gurantee saftey without the user being unable to delete the namespace that contains the resources they create.
1. mig controller cr can take a configurable namespace, that the migration controller will use to watch for MigMigration resources.

### Non-Goals

1. Allow for the mig controller to watch a subset of namespaces
1. Allow for the mig controller to safetly watch the entire cluster and tenant the operations
1. Determining all thge saftefy requirements for finalizers and implementation. 

## Proposal

1. Adding a ENV variable to the mig controller to take the namespace to watch. In the main file will set the manager.Options.Namesapce field with the contents of the env var

```golang
...
// Create a new Cmd to provide shared dependencies and start components
log.Info("setting up manager")
namespace := os.Getenv("CONTROLLER_WATCH_NAMESPACE")
mgr, err := manager.New(cfg, manager.Options{Namespace:namespace})
```

1. MigrationController will have a new field `controller_watch_namespace` which be defaulted to `openshift-migration` to keep backwards compatability.
1. The migration UI will have to be deployed in the same namespace as the migration controller and have the "CONTROLLER_WATCH_NAMESPACE" added to it's permissions.
1. If there is an active MigMigration, we should protect the MigPlan with a finalizer.
1. Must Gather must be updated to handle the configurablity of the controller to find the Mig* and Direct* resources.


### User Stories

Detail the things that people will be able to do if this is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.

#### Story 1
As a cluster admin I would like to configure the location of the openshift-migration resources to be created/edited and watched.

#### Story 2
As a cluster admin I would like the resources that should not be deletable be protected with a finalizer so they can be removed saftly.

### Implementation Details/Notes/Constraints

The use of finalizers is to make sure that the controller is alerted to the deletion action and can take the correct actions. In the case of a Plan deletion, this means the clean up of the MigMigration resources and when that is deleted any resources on the source or destination cluster. 
The namespace seperation is also a step in the direction for non-admin because today the controller only watches a single namespace name and this moves us into the direction that this can be configurable or there could be multiple controllers watching multiple namespaces. This is not something to be solved in this proposal but does start us down the direction that this is possible. I also think it is imprtant that the operator and the controller are in the same namespace.

### Risks and Mitigations

1. We need to have documentation and information about setting the namespace to one that exists or that regular users have access to.
2. We need to make sure that the UI can only create the resources that we think it should be created and should re-audit the permissions that it has.

## Design Details

### Upgrade / Downgrade Strategy

* Because the default is the same, the upgrade strategy should be just the default is used.
* Downgrading an operator with OLM after this change will add complexity, if this is a requirement then I think we may have other issues w/ the velero CRD's as well as the OLM process for downgrading.

## Drawbacks

1. More logic making the controller and must gather harder to implement

## Alternatives

1. Use Finalizers in the same namespace as the controller. Because deletion of the namespace does not have ordering, the namespace is left in deleting state until the finalizers are cleared---
