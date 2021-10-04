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

## Open Questions [optional]

This is where to call out areas of the design that require closure before deciding
to implement the design.  For instance, 
 > 1. This requires exposing previously private resources which contain sensitive
  information.  Can we do this? 

## Summary

In MTC 1.7, we envision to implement first class UX around _Storage Conversion_ where the users will use MTC to change the underlying storage of their PVCs. _Storage Conversion_ is already possible in MTC 1.6, but it doesn't have a specific guided experience. The users need to use _State Migration_ in a certain way to achieve storage conversion. Currently, the PV selection and the discovery part in State Migration takes place in the existing _MigPlan_ API. There are no changes made in the API itself for state migrations. Unlike usual migrations, both _state migration_ and the _storage conversion_ are not designed for migrating entire namespaces. The user experience of current _MigPlan_ is suitable for migrations of entire namespaces. There are problems when it comes to using the same API for _State Migrations_ or _Storage Conversion_. With minor changes to API, we can build first class experience around _Storage Conversion_. Similar changes will also improve UX of _State Migrations_. This enhancement mainly highlights user experience issues around _Storage Conversion_ and proposes changes in _MigPlan_ API to consolidate the experience of different types on migrations in one place.

## Motivation

_Storage Conversion_ will allow users to convert underlying storage of PVCs. The internal mechanism will be handled through existing _State Migration_. With MTC 1.6, storage conversion is already possible. However, there is no obvious way for users to know how to achieve it without supporting documentation. There are details that users may not necessarily know (such as selecting the same clusters, not mapping namespaces, mapping PVCs to different names, etc). We need to provide a full end-to-end experience for storage conversion. Also, there are two different options the users can convert storage of their PVCs using _State Migration_. We want to give them only one option. For that, it is necessary to make some changes in the _MigPlan_ API. It will facilitate providing better warning messaging and validations based on type of migration being performed. Similar changes will also improve UX of _State Migrations_. 

### Goals

> Propose solution to implement first class user experience around storage conversion

> Propose approach to consolidate different types of migrations through Migration Plan

> Explain how proposes changes will improve existing state migration experience

### Non-Goals

> Propose controller changes for storage conversion

## Proposal

* _Storage conversion_ is already possible in MTC 1.6 but it doesn't have a guided experience. The users need to set the right namespace and PV mappings in the Migration Plan, set the source & destination clusters to the same cluster, map storage classes and then perform a _State Migration_ to achieve storage conversion. For end users, this is hard to figure out without reading detailed supporting documentation. If we could mark a _MigPlan_ to be exclusively used for storage conversion, we can automatically choose the source, destination _MigCluster_ resources. We can also choose the namespace mappings correctly without the user having to provide it. Additionally, we can implement specific validations based on the type of the _MigPlan_. This wouldn't be possible if we don't know which type of migration the current plan will be used for.

* If the users perform a _State Migration_ by selecting a subset of PVCs in the plan, those settings are persisted on the plan itself. However, that _MigPlan_ is still available for normal migrations. This presents with some edge cases that are hard to guard against. For instance, if a PVC is skipped from a state migration, it remains skipped during a final migration as well. But if a workload is using that PVC, it would fail to come up in the destination cluster. It would be hard for a user to reach the root cause of the issue if they do not know the details. On the other hand, if we could use the _MigPlan_ exclusively for _State Migration_ we can disable its use for normal migrations and vice versa. Based on the `IncludedResources` API field, we can generate helpful warnings on _MigPlan_ beforehand.

* Based on the type of the migration, we can also enable/disable indirect migration and hooks options. Also, for _State Migration_ we can provide users with further options for configuring permission/SCC information which wouldn't be available for normal migrations. The details of permission handling will be covered in a different enhancement.

_MigPlan_ API will mainly introduce two new fields:

```yaml
// MigPlanSpec defines the desired state of MigPlan
type MigPlanSpec struct {
	[...]
	
	// Specifies whether migration plan will be used for state migration
	IsStateMigration  bool `json:"isStateMigration,omitempty"`
	
	// Specifies whether migration plan will be used for storage conversion
	IsStorageConversion  bool `json:"isStorageConversion,omitempty"`
}
```

These fields will tie every plan to a specific type of migration. The plan cannot be used for a different type of migration once that field is set. The field can only be changed if there are no migrations assigned with it. 

### User Stories [optional]

#### Story 1

As a user, I would like to use MTC to configure Migration Plan easily so that I can perform storage class conversion. 

### Implementation Details/Notes/Constraints [optional]

In progress...

### Risks and Mitigations

In progress...

## Design Details

### Test Plan

In progress...

### Upgrade / Downgrade Strategy

Both the fields are optional and will be defaulted to `false`. 

Existing MigPlans from MTC 1.6 and below will not have this field. Therefore, any such MigPlans can only be used for normal migrations in MTC 1.7 and above.

Any plan created in MTC 1.6 can have a _State Migration_ associated with it. The presence of state migration can be identified by checking the PVC mappings, MigCluster references, selection of PVCs, or the presence of `IncludedResources` field on the plan spec. In such cases, a new _Critical_ condition will be added on the migration plan to notify that the same plan cannot be used for migrations in MTC 1.7. Additionally, the message will also recommend a _Rollback_ operation before deleting the Migration Plan. These plans cannot be used in MTC 1.7 until a _Rollback_ is performed and the correct Spec fields are set indicating what kind of migration the plan will be used for in future.

For downgrades from 1.7 to lower versions, the new fields will be removed. The _State Migration_ will be available through the API as before. And hence, for storage conversions, the users will go back to using state migrations.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.
