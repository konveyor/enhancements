---
title: cloud-foundry-application-discovery
authors:
  - "@jordigilh"
  - "@gciavarrini"
reviewers:
  - "@dymurray"
  - "@jortel"
  - "@eemcmullan"
  - "@JonahSussman"
  - "@rromannissen"
  - "@savitharaghunathan"
approvers:
  - "@dymurray"
  - "@jortel"
  - "@eemcmullan"
  - "@JonahSussman"
  - "@rromannissen"
  - "@savitharaghunathan"
creation-date: 2025-01-17
last-updated: 2025-01-17
status: provisional
see-also:
  - https://github.com/konveyor/enhancements/pull/210
replaces:
superseded-by:
---

# Cloud Foundry Application Discovery to canonical form

This is the title of the enhancement. Keep it simple and descriptive. A good
title can help communicate what the enhancement is and should be considered as
part of any review.

The YAML `title` should be lowercased and spaces/punctuation should be
replaced with `-`.

To get started with this template:
1. **Pick a domain.** Find the appropriate domain to discuss your enhancement.
1. **Make a copy of this template.** Copy this template into the directory for
   the domain.
1. **Fill out the "overview" sections.** This includes the Summary and
   Motivation sections. These should be easy and explain why the community
   should desire this enhancement.
1. **Create a PR.** Assign it to folks with expertise in that domain to help
   sponsor the process.
1. **Merge at each milestone.** Merge when the design is able to transition to a
   new status (provisional, implementable, implemented, etc.). View anything
   marked as `provisional` as an idea worth exploring in the future, but not
   accepted as ready to execute. Anything marked as `implementable` should
   clearly communicate how an enhancement is coded up and delivered. If an
   enhancement describes a new deployment topology or platform, include a
   logical description for the deployment, and how it handles the unique aspects
   of the platform. Aim for single topic PRs to keep discussions focused. If you
   disagree with what is already in a document, open a new PR with suggested
   changes.

The `Metadata` section above is intended to support the creation of tooling
around the enhancement process.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]

This is where to call out areas of the design that require closure before deciding
to implement the design.  For instance,
 > 1. This requires exposing previously private resources which contain sensitive
  information.  Can we do this?

## Summary

Cloud Foundry (CF) is a platform-as-a-service solution that simplifies application deployment by abstracting infrastructure concerns. Applications on CF are deployed using manifests, which are YAML files specifying application properties, resource allocations, environment variables, and service bindings. These manifests provide a structured and declarative way to define the runtime and platform configurations required for an application.

MTA is an IBM research project with the goal to provide tools to migrate applications to other platforms, particularly Kubernetes. One of the use cases of MTA is to migrate Cloud Foundry applications to Kubernetes, but users have been struggling with the template language used to capture the Kubernetes resources, as it is not well known and requires an additional effort to master, compared to other templating languages.

## Motivation

The challenge brought by the templating language severely impacts the usability and acceptance of the tool. Thus the existence of this enhancement to provide a similar tool to MTA, extensible, and that improves on the templating engine, so that it offers a pluggable design that can be used to implement well known templating engines.

### Goals

* Identify and understand Cloud Foundry Application manifests (v3) fields.
* Extract and process Cloud Foundry application manifest into a new canonical form, capturing the intent of the original field value and
  with the foresight of the future application of the given field in a Kubernetes platform.
* Provide documentation for developers to understand the relationship between the original manifest and resulting canonical manifest.

### Non-Goals

* Transformation provider for Kubernetes using Helm Charts.

## Proposal

To migrate applications from Cloud Foundry to Kubernetes, it is essential to translate these manifests into an intermediate format, or canonical form, that captures the intent and configuration of the CF manifest. This intermediate manifest serves as a bridge, retaining critical deployment configurations while adapting them to Kubernetes-native practices. The format needs to be designed as platform-agnostic and compatible with multiple templating engines like Helm or Ansible, enabling flexibility in how the deployment configurations are generated and managed.

The following table depicts the relationship between the Cloud Foundry Application manifest fields and the proposed location in the canonical form manifest.

### Application specification {#application-specification}

| Name | Mapped  | Canonical Form | Comments |
| ----- | :---: | ----- | ----- |
| **name** | Y | Metadata.Name | Name is derived from the application’s Name field, which is stored in the metadata of the discovery manifest, following Kubernetes structured resources format.
See [metadata specification](#metadata-specification). |
| **buildpacks** | *N* |  | These fields in CF specify how to build your application (e.g., "nodejs\_buildpack", "java\_buildpack"). The canonical form should focus on "what" to deploy, not "how" to build it |
| **docker** | Y | Process.Image | The value of the docker image pullspec is captured for each \`Process\` in the Image field. See [process specification](#process-level-configuration). |
| **env** | Y | Env |  |
| **no-route** | Y | Routes | Processes will have no route information in the canonical form manifest. See [process specification](#process-specification). |
| **processes** | Y | Processes | See [process specification](#process-specification) |
| **random-route** | Y | Routes | See [route specification](#route-level-configuration). |
| **default-route** | Y |  | The default behavior for CF Applications is to define a route based on the name of the application and its domain ([https://docs.cloudfoundry.org/devguide/deploy-apps/deploy-app.html\#default-push](https://docs.cloudfoundry.org/devguide/deploy-apps/deploy-app.html#default-push)).
Example:
https://docs.cloudfoundry.org/devguide/multiple-processes.html |
| **routes** | Y | Routes | See [route specification](#route-level-configuration). |
| **services** | Y | Services | See [service specification](#service-specification). |
| **sidecars** | Y | Sidecars | See [sidecar specification](#sidecar-specification). |
| **metadata** | Y | Metadata | See [metadata specification](#metadata-specification). |
| **buildpack** | N |  | Already deprecated in CF. See buildpacks. |
| **timeout** | Y | StartupTimeout |  |
| **instances** | Y | Replicas |  |
| **stack** | Y | Stack |  |

### Sidecar specification {#sidecar-specification}

| Name | Mapped  | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **name** | *Y* | Name | Name of the sidecar |
| **process\_types** | Y | ProcessTypes | Slice of process types associated to this sidecar |
| **command** | Y | Command | Command to run this sidecar |
| **Memory** | Y | Memory | (Optional) The amount of memory allocated for the sidecar. |

### Service specification {#service-specification}

Maps to Spec.Services in the canonical form. Only \`name\` and \`parameters\` fields are captured since \`binding\_name\` is only used when defining a service and not applicable to a CF application.

| Name | Mapped  | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **name** | *Y* | Name | Name of the service required by the application |
| **parameters** | Y | Parameters | Parameters to be used when connecting to the service. |

### Metadata specification {#metadata-specification}

| Name | Mapped  | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **Application.name** | *Y* | Name | Name is derived from the application’s Name field, which is stored in the metadata of the discovery manifest, following Kubernetes structured resources format. |
| **labels** | *Y* | Labels |  |
| **annotations** | Y | Annotations |  |

### Process specification {#process-specification}

| Name | Mapped | Canonical Form | Comments |
| ----- | :---: | ----- | ----- |
| **type** | Y | Type | Only web or worker types are supported. |
| **Application.docker** |  | Image | Pull specification of the container image. This field is derived from the docker’s field in the application spec. See [application specification](https://docs.google.com/document/d/1zYBWSe6WYQzv6eLozi6jBL-TYVpOKktuzv2no92tG30/edit?pli=1&tab=t.0#heading=h.5vxln8xxilyw). |
| **command** | Y | Command | The command used to start the process. |
| **disk\_quota** | Y | DiskQuota | Example: 1G unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case
Note: In CF, limit for all instances of the **web** process; |
| **memory** | Y | Memory | The value at the application level defines the default memory requirements for all processes in the application, when not specified by the process itself. The discovery process will consolidate the amount of memory specific to each process based on the information either in the application or the process fields. Example: 128MB unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case Note: In CF, limit for all instances of the **web** process; |
| **health-check-http-endpoint** | Y  | HealthCheck | health-check fields are captured in a Probe structure, common with the readiness-heath-check |
| **health-check-invocation-timeout** |  |  |  |
| **health-check-interval** |  |  |  |
| **health-check-type** | N |  | Type of health check to perform; `none` is deprecated and an alias to `process` |
| **readiness-check-http-endpoint** | Y | ReadinessCheck | readiness-health-check fields are captured in a Probe structure, common with the heath-check |
| **readiness-check-invocation-timeout** |  |  |  |
| **readiness-check-interval** |  |  |  |
| **readiness-health-check-type** | N |  | Type of health check to perform; `none` is deprecated and an alias to `process` |
| **instances** | Y | Replicas | This field determines how many instances of the process will run in the application.  |
| **log-rate-limit-per-second** | Y | LogRateLimit | The log rate limit for all the instances of the process; unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case, or \-1 or 0 |

## HealthCheck specification

| Name | Mapped  | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **health-check-http-endpoint** | *Y* | Endpoint |  |
| **health-check-invocation-timeout** | Y | Timeout |  |
| **health-check-interval** | Y | Interval |  |
| **health-check-type** | N |  | Type of health check to perform; `none` is deprecated and an alias to `process` |

## ReadinessCheck specification

| Name | Mapped  | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **readiness-health-check-http-endpoint** | *Y* | Endpoint |  |
| **readiness-health-check-invocation-timeout** | Y | Timeout |  |
| **readiness-health-check-interval** | Y | Interval |  |
| **readiness-health-check-type** | N |  | Type of health check to perform; `none` is deprecated and an alias to `process` |

## Route-level specification

Captures the name of the route that will be shown as hostname.

This can be either the application name (by default CF will try to create a route with the app name as the hostname when \`no-route\` is not set to true and the process has no route defined.

If the application has globally defined routes, those processes of web type (worker type are not meant to have any port opened) will inherit the routes defined in this field. Overriden when a Process has defined their own routes.

| Name | Mapped  | Canonical Form | Description |
| ----- | :---: | ----- | ----- |
| **hostname** | Y | FQDN  | `FQDN as defined in the field value.` |
|  |  | Port | If the hostname contains a port (TCP protocol type), it is captured in the Port field. |
| **protocol** | *Y* | Protocol | It can be HTTP, HTTPS or TCP. |


### User Stories [optional]

### Implementation Details/Notes/Constraints [optional]

What are the caveats to the implementation? What are some important details that
didn't come across above. Go in to as much detail as necessary here. This might
be a good place to talk about core concepts and how they relate.

### Security, Risks, and Mitigations

**Carefully think through the security implications for this change**

What are the risks of this proposal and how do we mitigate. Think broadly. How
will this impact the broader OKD ecosystem? Does this work in a managed services
environment that has many tenants?

How will security be reviewed and by whom? How will UX be reviewed and by whom?

Consider including folks that also work outside your immediate sub-project.

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

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to...
  -  keep previous behavior?
  - make use of the enhancement?

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
