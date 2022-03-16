---
title: crane-ui-plugin-auth
authors:
  - "@djzager"
reviewers:
  - "@alaypatel107"
  - "@mturley"
  - "@shawn-hurley"
approvers:
  - "@mturley"
  - "@shawn-hurley"
creation-date: 2022-01-18
last-updated: 2022-01-18
status: implementable
see-also:
  - "/enhancements/crane-2.0/tekton-ui-integration/README.md" 
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Crane UI Plugin Auth

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

When migrating workloads and state between clusters, we need a mechanism for
managing access to these clusters. As was detailed in 
[Tekton UI Integration](/enhancements/crane-2.0/tekton-ui-integration/README.md),
the UI creates Secrets for the source and destination cluster to be used by the
generated Pipelines (and PipelineRuns). This proposal introduces a simple
service the Crane UI Plugin reaches out to in order to generate a Kubernetes
Secret to hold the user's credentials for accessing the destination cluster.

## Motivation

We want communications from our migration pipelines to source and destination
clusters to be **as** the user who submitted the pipeline. Additionally, we want
to be mindful of common security concerns with respect to authentication,
authorization, and credentials storage.

### Goals

* Allow the user to execute migration tasks with their level of permission. 
* Prevent the user from having to provide login credentials in a UI that already
    required the user to login.
* Getting the informed consent of the user before we take their credentials and
    shove it in a Secret on their behalf.
* Prevent "stale" permissions. For example, if we map the user's current
    permissions onto a Service Account + Role + Role Binding, then we have the
    concern that the user no longer has these permissions at the time the
    PipelineRun is executed (privilege escalation).

### Non-Goals

* To introduce a mechanism through which users can escalate their
    privileges in the cluster (privilege escalation).

## Proposal

At the time of this enhancement, the UI is asking the user for their token and
storing that token in a secret. When this enhancement is implemented, the UI
will make a `POST` request to the specified service to create a Secret with a
specified Name & Namespace. The designated service will retrieve the
authentication headers and use these to create a secret for use in the
Pipelines.

1. Introduce a new service (tentatively the `crane-secret-service`).
1. Update operator to deploy this service.
1. Update crane-ui-plugin's ConsolePlugin definition to include this service as
   an authenticated proxy.
1. Prompt the user in the user that, continuing will result in their credentials
   being stored as a secret. This is our mechanism for getting informed consent.
1. When the user decides to continue, the UI reaches out to the
   `crane-secret-service` to create a Secret with a given name and namespace.
   The service will combine the authorization headers with the name and
   namespace parameters to create a Secret with the credentials needed to access
   the cluster.
1. The UI can then use the Secret in the remainder of the flow as it did
   previously when the user was being asked for their token.

### Security, Risks, and Mitigations

* Risk: anyone with permission to read Secrets in the namespace will have access
    to the submitting user's credentials.
   * Mitigation: We will specify an OwnerReference on the created Secret pointing
      to the Pipeline created by the UI so that when the Pipeline is cleaned up,
      so is the user's credentials.
     * Additionally, this is the expected behavior in Kubernetes' clusters.
         User's given access to read Secrets should be trustworthy.

<!---
## Design Details

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to...
  -  keep previous behavior?
  - make use of the enhancement?
-->

## Drawbacks

There is certainly a need in the Tekton community, and in the larger Kuberentes
community, for a mechanism that allows workloads to be run on behalf of the user
and for API requests made by the workload to be made **AS** the workload
submitter. When a more generic solution exists that solves our problem exists,
we should consider pursuing it.

## Alternatives

For ease of reference the alternatives are simply stated here:

1. Ask the user for their token and store it in a secret.
1. Send the user to the token exchange to retrieve a token and store that in a
   secret.
1. Introduce new CRD surface + controller + and WebHook.
1. Force the Pipelines to run as "admin" by binding a Service Account to the
   "admin" RoleBinding.
1. Map the user's permissions (via a `SelfSubjectRulesReview`) onto a Role and
   bind that to a generated Service Account.
1. Allow the user to specify a Service Account to use in the migration.
1. Create a separate namespace where the user can run the migration, call it
   migration namespace for clarity. The idea is to create resources and secrets 
   in the migration namespace so it is secure from other users in the user namespace. 
   However, this does not solve the original problem because the template for
   creating the namespace is specified by the cluster-admin at the cluster level,
   more details
   [here](https://docs.openshift.com/container-platform/4.10/applications/projects/configuring-project-creation.html#modifying-template-for-new-projects_configuring-project-creation). 
   Because it is a cluster-wide configuration, the same user groups that have read
   access in the namespace being migrated will continue to have the read access in
   the migration namespace. Hence this is not any better than migrating in the existing 
   namespace.

### Ask User for Their Token

This is the approach that was chosen for the original
[Tekton UI Integration Enhancement](https://github.com/konveyor/enhancements/pull/59/files).
Since the Crane UI is already asking users for a URL and token for the source
cluster, we simply added an additional form field to retrieve the user's token
for accessing the destination cluster (ie. where the UI is running). That token
would be stored in a secret -- same as the source cluster credentials -- to be
referenced in the PipelineRun created by the UI.

We rejected this because of the horrible user experience,
entering credentials in an application for which you have already been
authenticated.

### Use the Token Exchange

Multiple alternatives involved the use of a token exchange that would require
the Crane UI user to be directed away from the wizard to the token exchange
where a new token (presumably with lesser privileges) would be returned, stored
in a secret, and leveraged in PipelineRuns.

This alternative also suffered from the horrible user experience of forcing an
authenticated user to go get us a token with the slight bonus of potentially
providing a mechanism to change expiry and namespace scoping to the permissions.
We weren't able to determine that retrieving a net new token with less
privileges was even possible.

This alternative was rejected because of the user experience issues and
increased complexity over simply asking for the user's token.

### New Migration Controller + CRD + WebHooks

For this alternative, we would need to implement a whole new controller + CRD
surface; for this example we will call it a `MigRun`. The UI would then submit
`MigRun`s, the `MigRun` would be annotated with the submitting user's name, and
WebHooks would be used to prevent manipulation of the submitted `MigRun`. The
`MigRun` controller would be responsible for creating a Service Account in the
user's namespace with a RoleBinding giving it permission to `impersonate` the
user in that namespace. The Crane Runner ClusterTasks would be updated to
support running `--as $USER` to perform all work as the user who completed the
UI wizard.

Although a significant increase in complexity versus some of the other options,
this alternative solved many of our auth concerns. A significant driver of this
option was the ability to prevent us from storing the user's credentials in a
Secret. When it was determined

### Generate SA and add to "admin" RoleBinding

This alternative depended on how, when a user creates a namespace, they
are automatically made "admin" of the namespace. Since normal Kubernetes users
cannot be specified on Pod definitions (or Tekton resources) for the purposes of
auth, we would create a new Service Account and add it as a subject to the
"admin" RoleBinding. We would then take that Service Account and use it in the
PipelineRuns generated by the UI.

This alternative was rejected because we determined it was inappropriate to
assume that an application owner would be a namespace admin. If that is true
then an application owner may be put in a position where they attempt to perfom
a migration (or application import) but fail because they are not able to create
the RoleBinding but would have succeeded with properly scoped permissions.

### Generate SA + Role + RoleBinding

This alternative leveraged Kubernetes' `SelfSubjectRulesReview` that:

> enumerates the set of actions the current user can perform within a namespace

with the idea that if we knew what the user had permission to do, we could
simply create a Role with those permissions and bind that to a Service Account
we generate for the Pipeline Run. However, it was determined that this would be
an abuse of the `SelfSubjectRulesReview`s stated purpose:

> SelfSubjectRulesReview should be used by UIs to show/hide actions, or to quickly let an end user reason about their permissions. 

**Links**

* https://kubernetes.io/docs/reference/access-authn-authz/authorization/
* https://docs.openshift.com/container-platform/4.9/rest_api/authorization_apis/selfsubjectrulesreview-authorization-k8s-io-v1.html

### Allow the User to Specify SA to Use

This alternative deferred the problem onto the user to create a properly
permissioned Service Account **or** submit a request to their cluster
administrator to do the same. Here we would simply have the user select, in the
Crane UI, what Service Account to use and it would be referenced in the
generated PipelineRun as appropriate.

We rejected this alternative because it required either a savvy Kubernetes user
OR prior effort on the part of the cluster administrator to create Service
Accounts suitable for use in migration tasks. Neither of these were acceptable
limitations on what we wanted to deliver.
