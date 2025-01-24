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


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

Cloud Foundry (CF) is a platform-as-a-service solution that simplifies application
deployment by abstracting infrastructure concerns. Applications on CF are deployed
using manifests, which are YAML files specifying application properties, resource
allocations, environment variables, and service bindings. These manifests provide
a structured and declarative way to define the runtime and platform configurations
required for an application.

Move2Kube is an IBM research project with the goal to provide tools to migrate
applications to other platforms, particularly Kubernetes. One of the use cases
of Move2Kube is to migrate Cloud Foundry applications to Kubernetes, but users
have been struggling with the template language used to capture the Kubernetes
resources, as it is not well known and requires an additional effort to master,
 compared to other templating languages.

## Motivation

The challenge brought by the templating language severely impacts the usability
and acceptance of the tool. Thus the existence of this enhancement to provide a
similar tool to Konveyor, extensible, and that improves on the templating engine, so
that it offers a pluggable design that can be used to implement well known
templating engines.

### Goals

* Identify and understand Cloud Foundry Application manifests (v3) fields.
* Extract and process Cloud Foundry application manifest into a new canonical
  form, capturing the intent of the original field value and with the foresight
  of the future application of the given field in a Kubernetes platform.
* Provide documentation for developers to understand the relationship between
  the original manifest and resulting canonical manifest.

### Non-Goals

* Transformation provider for Kubernetes using Helm Charts.

## Proposal

To migrate applications from Cloud Foundry to Kubernetes, it is essential to
translate these manifests into an intermediate format, or canonical form, that
captures the intent and configuration of the CF manifest. This intermediate
manifest serves as a bridge, retaining critical deployment configurations while
adapting them to Kubernetes-native practices. The format needs to be designed
as platform-agnostic and compatible with multiple templating engines like Helm
or Ansible, enabling flexibility in how the deployment configurations are
generated and managed.

These structures are intended to abstract the CF application manifest format
so that changes to the CF manifest are contained.

### Cloud Foundry specification
This section outlines the Cloud Foundry (CF) schema fields as documented in the
[official CF documentation](https://v3-apidocs.cloudfoundry.org/version/3.163.0/#concepts).
It serves as a reference point for comparing and understanding the mappings
presented in the [Proposal Specification](#proposal-specification) section.

#### Space-level configuration

##### Definition

| Name | Type | Description |
| ----- | ----- | ----- |
| **applications** | array of [app configurations](#app-level-configuration) | Configurations for apps in the space |
| **version** | integer | The manifest schema version; currently the only valid version is `1`, defaults to `1` if not provided |

#### App-level configuration

This configuration is specified per application and applies to all of the application’s processes.

##### Definition

| Name | Type | Description |
| ----- | ----- | ----- |
| **name** | string | Name of the app |
| **buildpacks** | array of strings | a) An empty array, which will automatically select the appropriate default buildpack according to the coding language b) An array of one or more URLs pointing to buildpacks c) An array of one or more installed buildpack names Replaces the legacy `buildpack` field |
| **docker** | object | If present, the created app will have *Docker lifecycle type*[^1]; the value of this key is ignored by the API but may be used by clients to source the registry address of the image and credentials, if needed; the [generate manifest endpoint](https://v3-apidocs.cloudfoundry.org/version/3.163.0/#generate-a-manifest-for-an-app) will return the registry address of the image and username provided with this key |
| **env** | object | A key-value mapping of environment variables to be used for the app when running |
| **processes** | array of [process configurations](#process-level-configuration) | List of configurations for individual process types |
| **random-route** | boolean | Creates a random route for the app if `true`; if `routes` is specified, if the app already has routes, or if `no-route` is specified, this field is ignored regardless of its value |
| **no-route** | bool | If false, no route is created for this application, regardless of the configuration. Note that health checks will be impacted since CF [is not able to reach](https://lists.cloudfoundry.org/g/cf-dev/topic/app_attribute_no_route_true/6333713) to the app externally to check the heart beat. This will need to be addressed in the manifest template provided by the user. |
| **routes** | array of [route configurations](#route-level-configuration) | List declaring HTTP and TCP routes to be mapped to the app. |
| **services** | array of [service configurations](#service-level-configuration) | A list of service-instances to bind to the app |
| **sidecars** | array of [sidecar configurations](#sidecar-level-configuration) | A list of configurations for individual sidecars |
| **stack** | string | The root filesystem to use with the buildpack, for example `cflinuxfs4` |
| **metadata.labels** | array of k/v pairs | Labels applied to the app |
| **metadata.annotations** | array of k/v pairs | Annotations applied to the app |
| **timeout** | integer | Maximum time it can take an application to startup before CF considers it as failed. Measured in seconds |

[^1] This allows Cloud Foundry to run pre-built Docker images. When staging an
app with this lifecycle, the Docker registry is queried for metadata about the
image, such as ports and start command. When running an app with this lifecycle,
a container is created and the Docker image is executed inside of it.

#### [Process-level configuration](https://v3-apidocs.cloudfoundry.org/version/3.163.0/#processes)

This configuration is for the individual process. Each process is created if it
does not already exist. For backwards compatibility, the web process
configuration may be placed at the top level of the application configuration,
rather than listed under processes. However, if there is a process with `type: web`
listed under processes, this configuration will override any at the top level.

##### Definition

| Name | Type | Description |
| ----- | ----- | ----- |
| **type** | string | **(Required)** The identifier for the processes to be configured |
| **command** | string | The command used to start the process; this overrides start commands from [Procfiles](#procfiles) and buildpacks |
| **disk\_quota** | string | The disk limit for all instances of the web process; this attribute requires a unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case |
| **health-check-http-endpoint** | string | Endpoint called to determine if the app is healthy |
| **health-check-invocation-timeout** | integer | The timeout in seconds for individual health check requests for http and port health checks |
| **health-check-type** | string | Type of health check to perform; `none` is deprecated and an alias to `process` |
| **[readiness-health-check-http-endpoint](https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#readiness-health-check-http-ep)** | string | Endpoint called to determine if the app is ready to accept traffic.  |
| **[readiness-health-check-invocation-timeout](https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#readiness-health-check-invoc-time)** | integer | The timeout in seconds for individual health check requests for http and port health checks |
| **[readiness-health-check-type](https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#readiness-health-check-type)** | string | Type of check to perform; `none` is deprecated and an alias to `process` |
| **instances** | integer | The number of instances to run |
| **memory** | string | The memory limit for all instances of the web process; this attribute requires a unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case |
| **log-rate-limit-per-second** | string | The log rate limit for all the instances of the process; this attribute requires a unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case, or \-1 or 0 |

##### [Procfiles](https://v3-apidocs.cloudfoundry.org/version/3.163.0/#procfiles)

A Procfile enables you to declare required runtime processes, called process
types, for your app. Procfiles must be named `Procfile` exactly and placed
in the root directory of your application.

***Example***

```
web: bundle exec rackup config.ru -p $PORT
rake: bundle exec rake
worker: bundle exec rake workers:start
```

In a Procfile, you declare one process type per line and use the syntax
`PROCESS_TYPE: COMMAND`.

* `PROCESS_TYPE` defines the type of the process.
* `COMMAND` is the command line to launch the process.

###### Procfile use cases

Many buildpacks provide their own process types and commands by default; however,
there are special cases where specifying a custom `COMMAND` is necessary.
Commands can be overwritten by providing a Procfile with the same process type.

For example, a buildpack may provide a `worker` process type that runs the
`rake default:start` command. If a Procfile is provided that also contains a
`worker` process type, but a different command such as `rake custom:start`, the
`rake custom:start` command will be used.

Some buildpacks, such as Python, that work on a variety of frameworks, do not
attempt to provide a default start command. For these cases, a Procfile should
be used to specify any necessary commands for the app.

###### Web process

`web` is a [special process type](https://v3-apidocs.cloudfoundry.org/version/3.163.0/#web-process-type)
that is required for all applications. The `web` `PROCESS_TYPE` must be specified
by either the buildpack or the Procfile.

###### Specifying processes in manifest files

Custom process types can also be configured via a manifest file. Read more about
[manifests](https://v3-apidocs.cloudfoundry.org/version/3.163.0/#manifests).
It is not recommended to specify processes in both a manifest and a Procfile for
the same app.

#### [Route-level configuration](https://v3-apidocs.cloudfoundry.org/version/3.163.0/#routes)

This [configuration](https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#routes)
is for *creating* mappings between the app and a route. Each route is created if
it does not already exist. The protocol will be updated for any existing route
mapping.

Example:

```
---
  ...
  routes:
  - route: example.com
    protocol: http2
  - route: www.example.com/foo
  - route: tcp-example.com:1234
```

##### Definition

| Name | Type | Description |
| ----- | ----- | ----- |
| **route** | string | **(Required)** The route URI |
| **protocol** | string | (Optional) Protocol to use for this route. Valid protocols are `http1`, `http2`, and `tcp`. |

#### [Service-level configuration](https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#services-block)

This configuration is *creating* new service bindings between the app and a
service instance. The `services` field can take either an array of service
instance name strings or an array of the following service-level fields.

##### Definition

| Name | Type | Description |
| ----- | ----- | ----- |
| **name** | string | **(Required)** The name of the service instance to be bound to |
| **binding\_name** | string | The name of the service binding to be created |
| **parameters** | object | A map of arbitrary key/value pairs to send to the service broker during binding |

#### [Sidecar-level configuration](https://v3-apidocs.cloudfoundry.org/version/3.163.0/#sidecars)

This configuration is for the individual sidecar. Each sidecar is created if
it does not already exist.

##### Definition

| Name | Type | Description |
| ----- | ----- | ----- |
| **name** | string | **(Required)** The identifier for the sidecars to be configured |
| **command** | string | The command used to start the sidecar |
| **process\_types** | array of strings | List of processes to associate sidecar with |
| **memory** | integer | Memory in MB that the sidecar will be allocated |

### Proposal Specification

#### Space specification

| Name |  Canonical Form | Description |
| ----- | ----- | ----- |
| **applications** | Application | Direct mapping to a slice of canonical form manifests, each one representing the discovery results of a CF application. See [app-level specification](#application-specification) |
| **space** | Metadata.Space | See [metadata specification](#metadata-specification). This field is only populated at runtime. |
| **version** | Metadata.Version | The manifest schema version; currently the only valid version is 1, defaults to 1 if not provided. This field is only populated at runtime. |

#### Application specification

| Name | Canonical Form | Comments |
| ----- | ----- | ----- |
| **name** | Metadata.Name | Name is derived from the application’s Name field, which is stored in the metadata of the discovery manifest, following Kubernetes structured resources format.
See [metadata specification](#metadata-specification). |
| **buildpacks** | BuildPacks | This field in CF specify how to build your application (e.g., "nodejs\_buildpack", "java\_buildpack"). |
| **docker** | Process.Image | The value of the docker image pullspec is captured for each \`Process\` in the `Image` field. See [process specification](#process-specification). |
| **env** | Env | Direct mapping from the application's `Env` field |
| **no-route** | Routes | Processes will have no route information in the canonical form manifest. See [route specification](#route-specification). |
| **processes** | Processes | See [process specification](#process-specification) |
| **random-route** | Routes | See [route specification](#route-specification). |
| **routes** | Routes | See [route specification](#route-specification). |
| **services** | Services | See [service specification](#service-specification). |
| **sidecars** | Sidecars | See [sidecar specification](#sidecar-specification). |
| **metadata** | Metadata | See [metadata specification](#metadata-specification). |
| **timeout** | StartupTimeout | Maximum time allowed for an application to respond to readiness or health checks during startup.If the application does not respond within this time, the platform will mark the deployment as failed. |
| **instances** | Replicas | Number of CF application instances |
| **stack** | Stack | Stack is derived from the `stack` field in the application manifest. The value is captured for information purposes because it has no relevance in Kubernetes. |

```go
type Application struct {
  // Metadata captures the name, labels and annotations in the application.
  Metadata Metadata `json:",inline"`
  // Env captures the `env` field values in the CF application manifest.
  Env map[string]string `json:"env,omitempty"`
  // Routes represent the routes that are made available by the application.
  Routes []Route `json:"route,omitempty"`
  // Services captures the `services` field values in the CF application manifest.
  Services []Service `json:"service,omitempty"`
  // Processes captures the `processes` field values in the CF application manifest.
  Processes []Process `json:"process,omitempty"`
  // Sidecars captures the `sidecars` field values in the CF application manifest.
  Sidecars []Sidecar `json:"sidecar,omitempty"`
  // Stack represents the `stack` field in the application manifest.
  // The value is captured for information purposes because it has no relevance
  // in Kubernetes.
  Stack string `json:"stack,omitempty"`
  // StartupTimeout specifies the maximum time allowed for an application to
  // respond to readiness or health checks during startup.
  // If the application does not respond within this time, the platform will mark
  // the deployment as failed. The default value is 60 seconds.
  // https://github.com/cloudfoundry/docs-dev-guide/blob/96f19d9d67f52ac7418c147d5ddaa79c957eec34/deploy-apps/  large-app-deploy.html.md.erb#L35
  StartupTimeout uint `json:"startupTimeout,omitempty"`
  // BuildPacks capture the buildpacks defined in the CF application manifest.
  BuildPacks []string `json:"buildPacks,omitempty"`
}
```

### Sidecar specification

| Name | Canonical Form | Description |
| ----- | ----- | ----- |
| **name** | Name | Name of the sidecar |
| **process\_types** | ProcessTypes | ProcessTypes captures the different process types defined for the sidecar. Compared to a Process, which has only one type, sidecar processes can accumulate more than one type. See [processtype specification](#processtype-specification).|
| **command** | Command | Command to run this sidecar |
| **Memory** | Memory | (Optional) The amount of memory to allocate to the sidecar. |

```go
type Sidecar struct {
  // Name represents the name of the Sidecar
  Name string `json:"name"`
  // ProcessTypes captures the different process types defined for the sidecar.
  // Compared to a Process, which has only one type, sidecar processes can
  // accumulate more than one type.
  ProcessTypes []ProcessType `json:"processType"`
  // Command captures the command to run the sidecar
  Command []string `json:"command"`
  // Memory represents the amount of memory to allocate to the sidecar.
  // It's an optional field.
  Memory string `json:"memory,omitempty"`
}
```

### Service specification

Maps to Spec.Services in the canonical form. Only \`name\`, \`parameters\`, and `bindng\_name` CF
fields are captured.

| Name | Canonical Form | Description |
| ----- | ----- | ----- |
| **name** | Name | Name of the service required by the application |
| **parameters** | Parameters | key/value pairs for the application to use when connecting to the service. |
| **binding\_name** | BindingName | Name of the service to bind to. |

```go
type Service struct {
  // Name represents the name of the Cloud Foundry service required by the
  // application. This field represents the runtime name of the service, captured
  // from the 3 different cases where the service name can be listed.
  // For more information check https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#services-block
  Name string `json:"name"`
  // Parameters contain the k/v relationship for the aplication to bind to the service
  Parameters map[string]interface{} `json:"parameters,omitempty"`
  // BindingName captures the name of the service to bind to.
  BindingName string `json:"bindingName,omitempty"`
}
```

### Metadata specification

| Name | Canonical Form | Description |
| ----- | ----- | ----- |
| **Application.name** | Name | Name is derived from the application’s Name field, which is stored in the metadata of the discovery manifest, following Kubernetes structured resources format. |
| **Space.name** | Space | Captured at runtime only and it contains the name of the space where the application is deployed. |
| **labels** | Labels |  Labels capture the labels as defined in the `labels` field in the CF application manifest |
| **annotations** | Annotations | Annotations as defined in the `annotations` field in the CF application manifest |
| **Space.version** | Version | Captured at runtime and it defaults to 1. |


```go
type Metadata struct {
  // Name capture the `name` field int CF application manifest
  Name string `json:"name"`
  // Space captures the `space` where the CF application is deployed at runtime. The field is empty if the
  // application is discovered directly from the CF manifest. It is equivalent to a Namespace in Kubernetes.
  Space string `json:"space,omitempty"`
  // Labels capture the labels as defined in the `annotations` field in the CF application manifest
  Labels map[string]string `json:"labels,omitempty"`
  // Annotations capture the annotations as defined in the `labels` field in the CF application manifest
  Annotations map[string]string `json:"annotations,omitempty"`
  // Version captures the version of the manifest containing the resulting CF application manifests list retrieved via REST API.
  // Only version 1 is supported at this moment. See https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#manifest-schema-version
  Version string `json:"version"`
}
```

### Process specification

| Name | Canonical Form | Comments |
| ----- | ----- | ----- |
| **type** | Type | Only web or worker types are supported. |
| **Application.docker** |  | Image | Pull specification of the container image. This field is derived from the docker’s field in the application spec. See [application specification](https://docs.google.com/document/d/1zYBWSe6WYQzv6eLozi6jBL-TYVpOKktuzv2no92tG30/edit?pli=1&tab=t.0#heading=h.5vxln8xxilyw). |
| **command** | Command | The command used to start the process. |
| **disk\_quota** | DiskQuota | Example: 1G unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case
Note: In CF, limit for all instances of the **web** process; |
| **memory** | Memory | The value at the application level defines the default memory requirements for all processes in the application, when not specified by the process itself. The discovery process will consolidate the amount of memory specific to each process based on the information either in the application or the process fields. Example: 128MB unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case. Note: In CF, limit for all instances of the **web** process; |
| **health-check-http-endpoint** | Y  | Probe.Endpoint | health-check fields are captured in a Probe structure, common with the readiness-heath-check. See [Probe specification](#probe-specification). |
| **health-check-invocation-timeout** | Probe.Timeout | See [Probe specification](#probe-specification). |
| **health-check-interval** | Probe.Interval | See [Probe specification](#probe-specification). |
| **health-check-type** | N |  | Type of health check to perform; `none` is deprecated and an alias to `process` |
| **readiness-check-http-endpoint** | Probe.Endpoint | See [Probe specification](#probe-specification). |
| **readiness-check-invocation-timeout** | Probe.Timeout | See [Probe specification](#probe-specification). |
| **readiness-check-interval** | Probe.Interval | See [Probe specification](#probe-specification). |
| **readiness-health-check-type** | N |  | Type of health check to perform; `none` is deprecated and an alias to `process` |
| **instances** | Replicas | This field determines how many instances of the process will run in the application. |
| **log-rate-limit-per-second** | LogRateLimit | The log rate limit for all the instances of the process; unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case, or -1 or 0 |

```go
type Process struct {
  // Type captures the `type` field in the Process specification.
  // Accepted values are `web` or `worker`
  Type ProcessType `json:"type,omitempty"`
  // Image represents the pull spec of the container image.
  Image string `json:"image"`
  // Command represents the command used to run the process.
  Command []string `json:"command,omitempty"`
  // DiskQuota represents the amount of persistent disk requested by the process.
  DiskQuota string `json:"disk,omitempty"`
  // Memory represents the amount of memory requested by the process.
  Memory string `json:"memory,omitempty"`
  // HealthCheck captures the health check information
  HealthCheck Probe `json:"healthCheck"`
  // ReadinessCheck captures the readiness check information.
  ReadinessCheck Probe `json:"readinessCheck"`
  // Replicas represents the number of instances for this process to run.
  Replicas uint `json:"replicas"`
  // LogRateLimit represents the maximum amount of logs to be captured per second.
  LogRateLimit string `json:"logRateLimit,omitempty"`
}
```

### ProcessType specification

Represents a single process type as a string. Possible values are `worker`, or `web`.

The proposed specification doesn't support custom process types defined in CF
manifests or Procfiles.

```go
type ProcessType string

const (
  // Web represents a `web` application type
  Web ProcessType = "web"
  // Worker represents a `worker` application type
  Worker ProcessType = "worker"
)
```

## Probe specification

| Name | Canonical Form | Description |
| ----- | ----- | ----- |
| **health-check-http-endpoint** | Endpoint | HTTP endpoint to be used for health checks, specifying the path to be monitored. |
| **health-check-invocation-timeout** | Timeout | Maximum time allowed for each health check invocation to complete. |
| **health-check-interval** | Interval | Interval at which health checks are performed to monitor the application’s status. |
| **health-check-type** | N |  | Specifies the type of health check to perform (`none`, `http`, `tcp`, or `process`). Note: `none` is deprecated and an alias for process. |

```go
type Probe struct {
  // Endpoint represents the URL location where to perform the probe check.
  Endpoint string `json:"endpoint"`
  // Timeout represents the number of seconds in which the probe check can be considered as timedout.
  // https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#timeout
  Timeout uint `json:"timeout"`
  // Interval represents the number of seconds between probe checks.
  Interval uint `json:"interval"`
}
```

## Route specification

Captures the name of the route that will be shown as hostname.

By default, the route URL is set using the application name as the hostname
unless the `no-route` field is set to `true` or a route URL is explicitly defined.
Processes of the `worker` type are not designed to have any ports open.
If the application has globally defined routes, processes of the `web` type
inherit the routes specified in that field. \
Examples:
\---

	...
	routes:
	\- route: example.com
	  protocol: http2
	\- route: www.example.com/foo
	\- route: tcp-example.com:1234

| Name | Canonical Form | Description |
| ----- | ----- | ----- |
| **url** | URL  | `URL as defined in the route field value.`  |
| **protocol** | Protocol | It can be `HTTP`, `HTTPS` or `TCP`. |

```go
type Route struct {
  // URL captures the FQDN, path and port of the route.
  URL string `json:"url"`
  // Protocol captures the protocol type: http, http2 or tcp. Note that the CF `protocol` field is only available
  // for CF deployments that use HTTP/2 routing.
  Protocol RouteProtocol `json:"protocol"`
}

type RouteProtocol string

const (
	HTTP  RouteProtocol = "http"
	HTTPS RouteProtocol = "https"
	TCP   RouteProtocol = "tcp"
)
```

### User Stories [optional]

* As a DevOps engineer migrating multiple applications, I want an intermediate
  canonical form that abstracts the complexities of Cloud Foundry manifests,
  so that I can easily adapt and reuse it for Kubernetes deployment across
  various platforms.

* As a DevOps engineer migrating multiple applications, I want to clearly
  understand the relationship between Cloud Foundry manifest fields and their
  canonical equivalents, so that I can confidently map my application's
  configurations to a Kubernetes-native environment.

* As a DevOps engineer migrating an application with complex configurations, I
  want the migration tool to preserve the intent and critical settings of the
  original Cloud Foundry manifest, so that my application behaves consistently after migration.

* As a DevOps engineer migrating applications, I want the migration tool to
  focus only on generating a canonical form, so that I can independently choose
  how to apply the canonical form using my preferred Kubernetes templating approach.

### Implementation Details/Notes/Constraints [optional]

What are the caveats to the implementation? What are some important details that
didn't come across above. Go in to as much detail as necessary here. This might
be a good place to talk about core concepts and how they relate.

### Security, Risks, and Mitigations

**Carefully think through the security implications for this change**

_What are the risks of this proposal and how do we mitigate. Think broadly. How_
_will this impact the broader OKD ecosystem? Does this work in a managed services_
_environment that has many tenants?_

_How will security be reviewed and by whom? How will UX be reviewed and by whom?_

_Consider including folks that also work outside your immediate sub-project._

#### Canonical Manifest Misalignment
CF provides features that are not directly mappable to Kubernetes-native
concepts, requiring additional work during the migration process to ensure
compatibility.

- *Buildpacks* \
  Cloud Foundry utilizes buildpacks to handle application dependencies and
  runtime environments, which do not have a direct equivalent in Kubernetes.
  This means that applications relying on buildpacks will require additional
  configuration or alternative solutions when migrating to Kubernetes.
- *Docker Secrets* \
  In CF, secrets management is integrated into the platform, while Kubernetes
  uses a more granular approach with its own secrets management system. This
  discrepancy necessitates a different handling method for sensitive information
  during migration.
- *Services* \
  Services in CF are managed differently than in Kubernetes. For
  instance, CF abstracts many operational concerns, while Kubernetes requires
  explicit configuration for service discovery and networking. This difference
  can lead to complexities during migration as developers must adapt to the
  Kubernetes model of service management.


*Mitigation*

Users will have to do due diligence before deploying on K8S, including the
creation of resources prior to the deployment of the K8S assets, such as
`namespaces`, `services`, `secrets` and container images for `buildpacks`,
for instance. \
Developers may need to refactor applications or implement new solutions that
align with Kubernetes practices.

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

N/A

## Implementation History

- January 2025: Proposal created and approved.
- (Tentative) February-March 2025: Initial MVP with support for CF manifest located in the filesystem where the CLI runs.
  Some limitations might apply based on the use cases to implement first.
- (Tentative) April 2025: Planned tech-preview release.

## Drawbacks

- M2K already provides discovery for Cloud Form application manifests. We could extend
  it to reuse the discovery logic and avoid the cost of redoing what is already provided.
- Breaks with existing M2K CLI support for CF application discovery. Some existing discovery
  functionality might not be supported in the short term and users will have to migrate their
  templates to the new Helm based template engine.

## Alternatives

- Reuse the existing M2K functionality for discovery, at the expense of using a code that is not ideal to manage
  and we are not a principal stakeholder.

## Infrastructure Needed [optional]

- CI/CD pipelines for building and unit/integration testing.
- CI/CD pipelines for E2E testing for QE and releasing.
- Hosting provider for code, binaries and documentation for releases.
- Project Management tools for tasks, issues and bugs.
- If REST API discovery is implemented, a Korifi instance on a Kubernetes cluster for E2E
  testing with a suite of samples that cover the acceptance test criteria.