---
title: migplan-ux-improvements
authors:
  - "@pranavgaikwad"
reviewers:
  - "@djwhatle"
  - "@sseago"
  - "@alaypatel07"
  - "@shawn-hurley"
approvers:
  - "@djwhatle"
  - "@sseago"
  - "@alaypatel07"
  - "@shawn-hurley"
creation-date: 2021-10-04
last-updated: 2021-10-04
status: implementable
see-also:
  - "../state-only-migrations/README.md" 
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# MigPlan UX Improvements

This enhancement proposes API changes to build a full end-to-end UX around _Storage Conversion_. The document explains how proposed changes consolidate different types of migrations in one place and provide a better overall experience for all three types of migrations currently available in MTC.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [ ] User-facing documentation is created

## Summary

In MTC 1.7, we envision to implement first class UX around _Storage Conversion_ where the users will use MTC to change the underlying storage of their PVCs. _Storage Conversion_ is already possible in MTC 1.6, but it doesn't have a specific guided experience. The users need to use _State Migration_ in a certain way to achieve storage conversion. Currently, the PV selection and the discovery part in State Migration takes place in the existing _MigPlan_ API. There are no changes made in the API itself for state migrations. Unlike usual migrations, both _state migration_ and the _storage conversion_ are not designed for migrating entire namespaces. The user experience of current _MigPlan_ is suitable for migrations of entire namespaces. There are problems when it comes to using the same API for _State Migrations_ or _Storage Conversion_. With minor changes to API, we can build first class experience around _Storage Conversion_. Similar changes will also improve UX of _State Migrations_. 

This enhancement mainly highlights user experience issues around _Storage Conversion_ and proposes two possible solutions to consolidate the experience of different types on migrations in one place. Finally, it discusses pros and cons of both approaches before coming to a conclusion of which one best fits our needs. 

## Motivation

_Storage Conversion_ will allow users to convert underlying storage of PVCs. The internal mechanism will be handled through existing _State Migration_. With MTC 1.6, storage conversion is already possible. However, there is no obvious way for users to know how to achieve it without supporting documentation. There are details that users may not necessarily know (such as selecting the same clusters, not mapping namespaces, mapping PVCs to different names, etc). We need to provide a full end-to-end experience for storage conversion. Also, there are two different options the users can convert storage of their PVCs using _State Migration_. We want to give them only one option. For that, it is necessary to make some changes in our API that will assist the users to make the right decisions. It will facilitate providing better warning messaging and validations based on type of migration being performed. Similar changes will also improve UX of _State Migrations_.

### Goals

> Propose solution to implement first class user experience around storage conversion

> Propose approach to consolidate different types of migrations through Migration Plan

> Explain how proposed changes will improve existing state migration experience

### Non-Goals

> Propose controller changes for storage conversion

## Proposal
### Approach 1: Introducing MigPlan API field to specify type of migration

* _Storage conversion_ is already possible in MTC 1.6 but it doesn't have a guided experience. The users need to set the right namespace and PV mappings in the Migration Plan, set the source & destination clusters to the same cluster, map storage classes and then perform a _State Migration_ to achieve storage conversion. For end users, this is hard to figure out without reading detailed supporting documentation. If we could mark a _MigPlan_ to be exclusively used for storage conversion, we can automatically choose the source, destination _MigCluster_ resources. We can also choose the namespace mappings correctly without the user having to provide it. Additionally, we can implement specific validations based on the type of the _MigPlan_. This wouldn't be possible if we don't know which type of migration the current plan will be used for.

* If the users perform a _State Migration_ by selecting a subset of PVCs in the plan, those settings are persisted on the plan itself. However, that _MigPlan_ is still available for normal migrations. This presents with some edge cases that are hard to guard against. For instance, if a PVC is skipped from a state migration, it remains skipped during a final migration as well. But if a workload is using that PVC, it would fail to come up in the destination cluster. It would be hard for a user to reach the root cause of the issue if they do not know the details. On the other hand, if we could use the _MigPlan_ exclusively for _State Migration_ we can disable its use for normal migrations and vice versa. Based on the `IncludedResources` API field, we can generate helpful warnings on _MigPlan_ beforehand.

* Based on the type of the migration, we can also enable/disable indirect migration and hooks options. Also, for _State Migration_ we can provide users with further options for configuring permission/SCC information which wouldn't be available for normal migrations. The details of permission handling will be covered in a different enhancement.

_MigPlan_ API will introduce a new field to indicate the type of the plan:

```yaml
// MigPlanSpec defines the desired state of MigPlan
type MigPlanSpec struct {
	[...]
	
	// Specifies type of the migration plan
	PlanType  *PlanType `json:"planType,omitempty"`
}

type PlanType string
```

These fields will tie every plan to a specific type of migration. The plan cannot be used for a different type of migration once that field is set. The field can only be changed if there are no migrations assigned with it.

### Approach 2: Introducing MigMigration API field to specify type of Migration

This approach relies on changing the _MigMigration_ API instead of _MigPlan_ API. Existing API contains two different fields to specify whether its a Stage migration or a Rollback:

```golang
type MigMigrationSpec struct {
	// Invokes the stage operation, when set to true the migration controller switches to stage itinerary. This is a required field.
	Stage bool `json:"stage"`

	// Invokes the rollback migration operation, when set to true the migration controller switches to rollback itinerary. This field needs to be set prior to creation of a MigMigration.
	Rollback bool `json:"rollback,omitempty"`
}
```

_Stage_ and _Rollback_ are two different types of migrations. There is a third type _Final_ which is an inferred type and it's triggered when _Stage_ is set to _False_. 


We will add _StateTransfer_ and _StorageConversion_ to the list of migrations. _StorageConversion_ is a special case of _StateTransfer_ in that the user selects the same cluster as a source and a destination and migrates PVs within the same namespace by changing the storage class. Therefore, this information can be automatically inferred by looking at _MigCluster_ refs and namespace mappings. As a result, we only want one additional field on the _MigMigration_ CR that will identify a state migration:

```golang
type MigMigrationSpec struct {
	State bool `json:"state,omitempty"`
}
```

We will add logic to validate co-existence of _Rollback_, _Stage_, and _State_ options. 

### Alternative: Change booleans to a typed field

Out of the above, field `Stage` is required while `Rollback` is optional. The problem with the two booleans is that there are no validations in place when both of these fields co-exist. If we were to follow the same approach to introduce one new field for _Storage Conversion_ and _State Transfer_, we would need additional logic to validate the co-existence of 3 fields instead of just two. We can use this as an opportunity to clean this up and introduce 1 typed field to indicate the type of migration:

```golang
type MigMigrationSpec struct {
	Type *MigrationType `json:"storageConversion,omitempty"`
}

type MigrationType string
```

MigrationType has four possible values: `Final`, `Stage`, `StateTransfer`, `StorageConversion`, `RollBack`.

This will present some problems with existing _MigMigrations_. Since migrations are supposed to reach to completion, these problems are relatively easier to solve. 

Existing CRs will not have the type field set, instead they will have the old booleans. In 1.7, we can deprecate the booleans and add controller logic to correctly set the Type field based on the old booleans. Then in later versions, we can remove the booleans altogether. Downgrades will be only possible from 1.7 to lower versions. 
#### Pros and Cons

> When users explicitely specify a type of Migration Plan, the controller can automatically change the validations based on the type and indicate users if they make mistakes specifying the fields. The validations will occur when creating a Migration Plan. For the latter approach, the same validations will happen at the time of creating a migration. The Migration Plan will still be created as usual.

> Changing _MigPlan_ API introduces more upgrade/downgrade issues than changing _MigMigration_ API. Adding a new field on _MigPlan_ to identify its type will make it harder to associate existing _MigPlans_ with their types as these fields never existed in old API. On the other hand, since _MigMigrations_ are supposed to reach a terminal state, the surface area of upgrade/downgrade issues is much smaller than that of the former approach. 

### User Stories [optional]

#### Story 1

As a user, I would like to use MTC to configure Migration Plan easily so that I can perform storage class conversion. 

## Design Details

### Test Plan

Test cases for Storage Conversion:

1. Portworx to OCS migration without attached workloads

2. Portworx to OCS migration with attached workloads:

  - we will test if the workloads are automatically updated to use the PVs

3. Test and confirm that Storage Class conversion is not possible in following conditions:

  - when source and destination clusters are different
  - when source namespace are mapped to different destination namespace
  - when indirect migrations are selected

General validations:

1. Co-existence of booleans should be validated

2. Appropriate conditions should be set on the _MigPlan_ CR based on associated migrations

  - durable conditions should stop users from switching to a migration type that will fail due to wrong selections
### Upgrade / Downgrade Strategy

#### MigPlan approach

The MigrationType field will be optional to allow Operator upgrade to 1.7 version. This will make sure that the existing MigPlans which dont have this field set will not block upgrade. For all Migration Plans that don't have this field set, the controller will default it to full namespace type migration and also add an condition notifying the user that the migration type is defaulted. All existing MigPlans will default to normal migration types.

Any plan created in MTC 1.6 can have a _State Migration_ associated with it. The presence of state migration can be identified by checking the PVC mappings, MigCluster references, selection of PVCs, or the presence of `IncludedResources` field on the plan spec. In such cases, a new _Critical_ condition will be added on the migration plan to notify that the same plan cannot be used for migrations in MTC 1.7. Additionally, the message will also recommend a _Rollback_ operation before deleting the Migration Plan. These plans cannot be used in MTC 1.7 until a _Rollback_ is performed and the correct Spec fields are set indicating what kind of migration the plan will be used for in future.

For downgrades from 1.7 to lower versions, the new fields will be removed. The _State Migration_ will be available through the API as before. And hence, for storage conversions, the users will go back to using state migrations.

#### MigMigration approach

The new boolean field will be optional. Any _MigMigration_ resources created in the previous versions will continue using the _Stage_ boolean. New _MigMigrations_ will use an additional _State_ migration field. Since _MigMigrations_ are supposed to reach to a completion, we dont have to worry about old completed migrations from previous versions. 

If there's any blocked (incomplete) migration during an upgrade, it will be defaulted to its type based on the value of `stage` boolean. However, this case should be rare. 