---
title: cam-non-admin-user-types
authors:
  - TBD
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2020-06-22
last-updated: 2020-06-22
status: deferred
see-also:
replaces:
superseded-by:
---

# Cluster Application Migration (CAM) Non-Admin User Types

This enhancement defines and describes the types of user for non-admin 
workflow. Terms defined in the enhancement will form the basis of the
user-facing documentation. 

## Open Questions [optional]

 > 1. What shall we name the type of user who needs to have "use" verb
on migration-controllers in remote clusters?

## Summary

CAM will have multiple types of users once non-admin is implemented. With non-admin
we have made a choice to make the UI multi-tenant in order for the oauthclient
secrets to not be copied into non-admin user namespaces. Because of this the 
UI needs to query the discovery service the namespace that users have access to.
With this there are 4 major buckets of users in CAM:

1. Basic Tenant: A basic tenant user for a namespace is defined as the user who has
access to create, update, get and delete migplans, migtokens, migmigrations and 
migstorage. When the UI first loads the for non-admin user, it asks for the list of 
namespaces that the non-admin user is a basic tenant for. The user will select
and active namespace in which UI will dump all their mig objects duing a migration.

2. Mig Tenant Admin: A mig tenant admin user for a namespace is a basic tenant user
but also has access to create, update, get and delete migclusters and 
migrationcontrollers. This makes the user a tenant admin in the sense that this user
can enable other users to run migration by creating source and destination migclusters
for them to use. When the operator is installed through OLM, all users with either
admin, edit or view-crd cluster role will be mig tenant admin, which means all non-admin
users will be mig tenant admins.

3. cluster-admin: This is the cluster admin user which has */*/* on all group, 
namespaces and verbs, i.e. full access to the cluster.

4. _____________: This is the user in the cluster for which mig-tokens are created
and the "use" verb SAR checks are performed for migrationcontroller/
migrations-controllers resource in migcluster for which the mig-token is created. 
This user is being granted "use" by the cluster-admin so that CAM can use elevated
privileges on behalf of the user to migrate this user's workload.

## Motivation

To define terms for types of user to make it explicit in design discussions and 
set basis for user-facing documentation.
