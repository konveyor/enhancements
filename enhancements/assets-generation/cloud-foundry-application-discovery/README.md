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

# Cloud Foundry Application Discovery to discovery manifest


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

 The goal for this enhancement is to define manifest output format from the
 result of the discovery process of a Cloud Foundry Application manifest. This
 output manifest, also referred as discovery manifest, is then consumed by the
 generate operation to render the desired assets.

## Motivation

The challenge brought by the templating language severely impacts the usability
and acceptance of the tool. Thus the existence of this enhancement to provide a
similar tool to Konveyor, extensible, and that improves on the templating engine, so
that it offers a pluggable design that can be used to implement well known
templating engines.

Overall, Konveyor provides the analysis of an application with the goal of finding
potential challenges to migrating such application to a target platform. Assert generation
enables Konveyor to enables users the ability to auto generate target manifests for the
intented platform.

### Goals

* Identify and understand Cloud Foundry Application manifests (v3) fields.
* Define the discovery manifest for Cloud Foundry Application manifests (V3). The
  discovery of a Cloud Foundry application will populate the fields of the discovery
  manifest so that they can be used as `values.yaml` in the transformation with Helm templates.
* Extract and process Cloud Foundry application manifest into a new discovery
  manifest, capturing the intent of the original field value and with the foresight
  of the future application of the given field in a Kubernetes platform.
* Provide documentation for developers to understand the relationship between
  the original manifest and resulting discovery manifest.

### Non-Goals

* Transformation provider for Kubernetes using Helm Charts.

## Proposal

To migrate applications from Cloud Foundry to Kubernetes, it is essential to
translate these manifests into an intermediate format, or discovery manifest, that
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
| **protocol** | string | (Optional) Protocol to use for this route. Valid protocols are `http`, `http2`, and `tcp`. |

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

| Name |  Discovery Specification | Description |
| ----- | ----- | ----- |
| **applications** | Application | Direct mapping to a slice of discovery manifests, each one representing the discovery results of a CF application. See [app-level specification](#application-specification) |
| **space** | Metadata.Space | See [metadata specification](#metadata-specification). This field is only populated at runtime. |
| **version** | Metadata.Version | The manifest schema version; currently the only valid version is 1, defaults to 1 if not provided. This field is only populated at runtime. |

#### Application specification

| Name | Discovery Specification | Comments |
| ----- | ----- | ----- |
| **name** | Metadata.Name | Name is derived from the application’s Name field, which is stored in the metadata of the discovery manifest, following Kubernetes structured resources format.
See [metadata specification](#metadata-specification). |
| **buildpacks** | BuildPacks | This field in CF specify how to build your application (e.g., "nodejs\_buildpack", "java\_buildpack"). |
| **docker** | Docker | The value of the docker image pullspec and the username. See [docker specification](#docker-specification). |
| **env** | Env | Direct mapping from the application's `Env` field |
| **no-route** | Routes | Processes will have no route information in the discovery manifest. Defaults to false. See [route specification](#route-specification). |
| **processes** | Processes | See [process specification](#process-specification) |
| **random-route** | Routes | See [route specification](#route-specification). |
| **routes** | Routes | See [route specification](#route-specification). |
| **services** | Services | See [service specification](#service-specification). |
| **sidecars** | Sidecars | See [sidecar specification](#sidecar-specification). |
| **metadata** | Metadata | See [metadata specification](#metadata-specification). |
| **timeout** | Timeout | Maximum time allowed for an application to respond to readiness or health checks during startup.If the application does not respond within this time, the platform will mark the deployment as failed. Defaults to 60 seconds and maximum is 180 seconds. Can be changes in the Cloud Foundry Controller.|
| **instances** | Instances | Number of CF application instances. Defaults to 1. |
| **stack** | Stack | Stack is derived from the `stack` field in the application manifest. The value is captured for information purposes because it has no relevance in Kubernetes. |

```go
type Application struct {
	Metadata Metadata `yaml:",inline" validate:"required"`
	// Env captures the `env` field values in the CF application manifest.
	Env map[string]string `yaml:"env,omitempty"`
	// Routes represent the routes that are made available by the application.
	Routes RouteSpec `yaml:"routes,inline,omitempty"`
	// Services captures the `services` field values in the CF application manifest.
	Services Services `yaml:"services,omitempty"`
	// Processes captures the `processes` field values in the CF application manifest.
	Processes Processes `yaml:"processes,omitempty"`
	// Sidecars captures the `sidecars` field values in the CF application manifest.
	Sidecars Sidecars `yaml:"sidecars,omitempty"`
	// Stack represents the `stack` field in the application manifest.
	// The value is captured for information purposes because it has no relevance
	// in Kubernetes.
	Stack string `yaml:"stack,omitempty"`
	// Timeout specifies the maximum time allowed for an application to
	// respond to readiness or health checks during startup.
	// If the application does not respond within this time, the platform will mark
	// the deployment as failed. The default value is 60 seconds and maximum to 180 seconds, but both values can be changed in the Cloud Foundry Controller.
	// https://github.com/cloudfoundry/docs-dev-guide/blob/96f19d9d67f52ac7418c147d5ddaa79c957eec34/deploy-apps/large-app-deploy.html.md.erb#L35
	// Default is 60 (seconds).
	Timeout int `yaml:"timeout" validate:"min=0,max=180"`
	// BuildPacks capture the buildpacks defined in the CF application manifest.
	BuildPacks []string `yaml:"buildPacks,omitempty"`
	// Docker captures the Docker specification in the CF application manifest.
	Docker Docker `yaml:"docker,omitempty"`
	// Instances captures the number of instances to run concurrently for this application. Default is 1.
	Instances int `yaml:"instances" validate:"required,min=1"`
}
```

### Docker specification

| Name | Discovery Specification | Description |
| ----- | ----- | ----- |
| **image** | Image | Pullspec of the container image. |
| **username** | Username | (Optional) Username to authenticate against the container registry.|

```go
type Docker struct {
	// Image represents the pullspect where the container image is located.
	Image string `yaml:"image" validate:"required"`
	// Username captures the username to authenticate against the container registry.
	Username string `yaml:"username,omitempty"`
}
```

### Sidecar specification

| Name | Discovery Specification | Description |
| ----- | ----- | ----- |
| **name** | Name | Name of the sidecar |
| **process\_types** | ProcessTypes | ProcessTypes captures the different process types defined for the sidecar. Compared to a Process, which has only one type, sidecar processes can accumulate more than one type. See [processtype specification](#processtype-specification).|
| **command** | Command | Command to run this sidecar |
| **Memory** | Memory | (Optional) The amount of memory to allocate to the sidecar. |

```go
type SidecarSpec struct {
	// Name represents the name of the Sidecar
	Name string `yaml:"name" validate:"required"`
	// ProcessTypes captures the different process types defined for the sidecar.
	// Compared to a Process, which has only one type, sidecar processes can
	// accumulate more than one type.
	ProcessTypes []ProcessType `yaml:"processType" validate:"required,oneof=worker web"`
	// Command captures the command to run the sidecar
	Command string `yaml:"command" validate:"required"`
	// Memory represents the amount of memory to allocate to the sidecar.
	// It's an optional field.
	Memory string `yaml:"memory,omitempty"`
}
```

### Service specification

Maps to Spec.Services in the discovery manifest. Only \`name\`, \`parameters\`, and `bindng\_name` CF
fields are captured.

| Name | Discovery Specification | Description |
| ----- | ----- | ----- |
| **name** | Name | Name of the service required by the application |
| **parameters** | Parameters | key/value pairs for the application to use when connecting to the service. |
| **binding\_name** | BindingName | Name of the service to bind to. |

```go
type ServiceSpec struct {
	// Name represents the name of the Cloud Foundry service required by the
	// application. This field represents the runtime name of the service, captured
	// from the 3 different cases where the service name can be listed.
	// For more information check https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#services-block
	Name string `yaml:"name" validate:"required"`
	// Parameters contain the k/v relationship for the aplication to bind to the service
	Parameters map[string]interface{} `yaml:"parameters,omitempty"`
	// BindingName captures the name of the service to bind to.
	BindingName string `yaml:"bindingName,omitempty"`
}
```

### Metadata specification

| Name | Discovery Specification | Description |
| ----- | ----- | ----- |
| **Application.name** | Name | Name is derived from the application’s Name field, which is stored in the metadata of the discovery manifest, following Kubernetes structured resources format. |
| **Space.name** | Space | Captured at runtime only and it contains the name of the space where the application is deployed. |
| **labels** | Labels |  Labels capture the labels as defined in the `labels` field in the CF application manifest |
| **annotations** | Annotations | Annotations as defined in the `annotations` field in the CF application manifest |
| **Space.version** | Version | Captured at runtime and it defaults to 1. |


```go
type Metadata struct {
	// Name capture the `name` field int CF application manifest
	Name string `yaml:"name" validate:"required"`
	// Space captures the `space` where the CF application is deployed at runtime. The field is empty if the
	// application is discovered directly from the CF manifest. It is equivalent to a Namespace in Kubernetes.
	Space string `yaml:"space,omitempty"`
	// Labels capture the labels as defined in the `annotations` field in the CF application manifest
	Labels map[string]*string `yaml:"labels,omitempty"`
	// Annotations capture the annotations as defined in the `labels` field in the CF application manifest
	Annotations map[string]*string `yaml:"annotations,omitempty"`
	// Version captures the version of the manifest containing the resulting CF application manifests list retrieved via REST API.
	// Only version 1 is supported at this moment See https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#manifest-schema-version
	// Defaults to 1
	Version string `yaml:"version"`
}
```

### Process specification

| Name | Discovery Specification | Comments |
| ----- | ----- | ----- |
| **type** | Type | Only web or worker types are supported. |
| **command** | Command | The command used to start the process. |
| **disk\_quota** | DiskQuota | Example: 1G unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case
Note: In CF, limit for all instances of the **web** process; |
| **lifecycle** | Lifecycle | The lifecycle attribute specifies which application lifecycle to use for staging and running the application. Three variants are supported at the moment: `buildpack`, `cnb`, and `docker`. Defaults to `buildpack`. See https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#lifecycle|
| **memory** | Memory | The value at the application level defines the default memory requirements for all processes in the application, when not specified by the process itself. The discovery process will consolidate the amount of memory specific to each process based on the information either in the application or the process fields. Example: 128MB unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case. Note: In CF, limit for all instances of the **web** process. Defaults to `1G`. https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#manifest-schema-version|
| **health-check-http-endpoint**  | Probe.Endpoint | health-check fields are captured in a Probe structure, common with the readiness-heath-check. See [Probe specification](#probe-specification). Defaults to `/`. See https://github.com/SchemaStore/schemastore/blob/926649610d04226ec3b37c58418d4340e4b1d36c/src/schemas/json/cloudfoundry-application-manifest.json#L172|
| **health-check-invocation-timeout** | Probe.Timeout | See [Probe specification](#probe-specification). Defaults to `1 second`. See https://github.com/SchemaStore/schemastore/blob/926649610d04226ec3b37c58418d4340e4b1d36c/src/schemas/json/cloudfoundry-application-manifest.json#L176|
| **health-check-interval** | Probe.Interval | See [Probe specification](#probe-specification). Defaults to `30 seconds`. See https://github.com/SchemaStore/schemastore/blob/926649610d04226ec3b37c58418d4340e4b1d36c/src/schemas/json/cloudfoundry-application-manifest.json#L180 |
| **health-check-type** | Type of health check to perform; `none` is deprecated and an alias to `process`. Defaults to `port`. See https://github.com/SchemaStore/schemastore/blob/926649610d04226ec3b37c58418d4340e4b1d36c/src/schemas/json/cloudfoundry-application-manifest.json#L168|
| **readiness-check-http-endpoint** | Probe.Endpoint | See [Probe specification](#probe-specification). Defaults to `/`. See https://github.com/SchemaStore/schemastore/blob/926649610d04226ec3b37c58418d4340e4b1d36c/src/schemas/json/cloudfoundry-application-manifest.json#L188|
| **readiness-check-invocation-timeout** | Probe.Timeout | See [Probe specification](#probe-specification). Defaults to `1 second`. See https://github.com/SchemaStore/schemastore/blob/926649610d04226ec3b37c58418d4340e4b1d36c/src/schemas/json/cloudfoundry-application-manifest.json#L192|
| **readiness-check-interval** | Probe.Interval | See [Probe specification](#probe-specification). Defaults to `30 seconds`. See https://github.com/SchemaStore/schemastore/blob/926649610d04226ec3b37c58418d4340e4b1d36c/src/schemas/json/cloudfoundry-application-manifest.json#L196|
| **readiness-health-check-type**  | Type of health check to perform; `none` is deprecated and an alias to `process`. Defaults to `process`. https://github.com/SchemaStore/schemastore/blob/926649610d04226ec3b37c58418d4340e4b1d36c/src/schemas/json/cloudfoundry-application-manifest.json#L184 |
| **instances** | Replicas | This field determines how many instances of the process will run in the application. Defaults to 1. See https://github.com/SchemaStore/schemastore/blob/926649610d04226ec3b37c58418d4340e4b1d36c/src/schemas/json/cloudfoundry-application-manifest.json#L263|
| **log-rate-limit-per-second** | LogRateLimit | The log rate limit for all the instances of the process; unit of measurement: `B`, `K`, `KB`, `M`, `MB`, `G`, `GB`, `T`, or `TB` in upper case or lower case, or -1 or 0. Defaults to `16K`. See https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#log-rate-limit-per-second |

```go
type ProcessSpec struct {
	// Type captures the `type` field in the Process specification.
	// Accepted values are `web` or `worker`
	Type ProcessType `yaml:"type" validate:"required,oneof=web worker"`
	// Command represents the command used to run the process.
	Command string `yaml:"command,omitempty"`
	// DiskQuota represents the amount of persistent disk requested by the process.
	DiskQuota string `yaml:"disk,omitempty"`
	// Memory represents the amount of memory requested by the process.
	Memory string `yaml:"memory" validate:"required"`
	// HealthCheck captures the health check information
	HealthCheck ProbeSpec `yaml:"healthCheck"`
	// ReadinessCheck captures the readiness check information.
	ReadinessCheck ProbeSpec `yaml:"readinessCheck"`
	// Instances represents the number of instances for this process to run.
	Instances int `yaml:"instances" validate:"required,min=1"`
	// LogRateLimit represents the maximum amount of logs to be captured per second. Defaults to `16K`
	LogRateLimit string `yaml:"logRateLimit" validate:"required"`
	// Lifecycle captures the value fo the lifecycle field in the CF application manifest.
	// Valid values are `buildpack`, `cnb`, and `docker`. Defaults to `buildpack`
	Lifecycle LifecycleType `yaml:"lifecycle,omitempty" validate:"required,oneof=buildpack cnb docker"`
}

type LifecycleType string

const (
	BuildPackLifecycleType LifecycleType = "buildpack"
	CNBLifecycleType       LifecycleType = "cnb"
	DockerLifecycleType    LifecycleType = "docker"
)
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

| Name | Discovery Specification | Description |
| ----- | ----- | ----- |
| **health-check-http-endpoint** | Endpoint | HTTP endpoint to be used for health checks, specifying the path to be monitored. |
| **health-check-invocation-timeout** | Timeout | Maximum time allowed for each health check invocation to complete. |
| **health-check-interval** | Interval | Interval at which health checks are performed to monitor the application’s status. |
| **health-check-type** | Type  | Specifies the type of health check to perform (`port`, `process`, `http`). |

```go
type ProbeSpec struct {
	// Endpoint represents the URL location where to perform the probe check.
	Endpoint string `yaml:"endpoint" validate:"required"`
	// Timeout represents the number of seconds in which the probe check can be considered as timedout.
	// https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html#timeout
	Timeout int `yaml:"timeout" validate:"required,min=0"`
	// Interval represents the number of seconds between probe checks.
	Interval int `yaml:"interval" validate:"required,min=0"`
	// Type specifies the type of health check to perform.
	Type ProbeType `yaml:"type" validate:"required,oneof=http process port"`
}

type ProbeType string

const (
  PortProbeType ProbeType = "port"
  ProcessProbeType ProbeType = "process"
  HTTPProbeType ProbeType = "http"
)
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

| Name | Discovery Specification | Description |
| ----- | ----- | ----- |
| **route** | Route  | `Route as defined in the route field value.`  |
| **protocol** | Protocol | It can be `http`, `http2` or `tcp`. |

```go

type RouteSpec struct {
	//NoRoute captures the field no-route in the CF Application manifest.
	NoRoute bool `yaml:"noRoute,omitempty"`
	//RandomRoute captures the field random-route in the CF Application manifest.
	RandomRoute bool `yaml:"randomRoute,omitempty"`
	//Routes captures the field routes in the CF Application manifest.
	Routes Routes `yaml:"routes,omitempty"`
}

type Route struct {
	// Route captures the domain name, port and path of the route.
	Route string `yaml:"route" validate:"required"`
	// Protocol captures the protocol type: http, http2 or tcp. Note that the CF `protocol` field is only available
	// for CF deployments that use HTTP/2 routing.
	Protocol RouteProtocol `yaml:"protocol,omitempty" validate:"required,oneof=http http2 tcp"`
	// Options captures the options for the Route. Only load balancing is supported at the moment.
	Options RouteOptions `yaml:"options,omitempty"`
}

type RouteOptions struct {
	// LoadBalancing captures the settings for load balancing. Only `round-robin` or `least-connections` are supported
	LoadBalancing LoadBalancingType `yaml:"loadBalancing,omitempty" validate:"oneof=round-robin least-connections"`
}

type LoadBalancingType string

const (
	RoundRobinLoadBalancingType      LoadBalancingType = "round-robin"
	LeastConnectionLoadBalancingType LoadBalancingType = "least-connection"
)

type RouteProtocol string

const (
	HTTPRouteProtocol  RouteProtocol = "http1"
	HTTP2RouteProtocol RouteProtocol = "http2"
	TCPRouteProtocol   RouteProtocol = "tcp"
)
```

### User Stories [optional]

* As a DevOps engineer migrating multiple applications, I want an intermediate
  discovery manifest that abstracts the complexities of Cloud Foundry manifests,
  so that I can easily adapt and reuse it for Kubernetes deployment across
  various platforms.

* As a DevOps engineer migrating multiple applications, I want to clearly
  understand the relationship between Cloud Foundry manifest fields and their
  discovery equivalents, so that I can confidently map my application's
  configurations to a Kubernetes-native environment.

* As a DevOps engineer migrating an application with complex configurations, I
  want the migration tool to preserve the intent and critical settings of the
  original Cloud Foundry manifest, so that my application behaves consistently after migration.

* As a DevOps engineer migrating applications, I want the migration tool to
  focus only on generating a discovery manifest, so that I can independently choose
  how to apply the discovery manifest using my preferred Kubernetes templating approach.

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

#### Discovery Manifest Misalignment
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

### Discovery Manifest specification

The following yaml are examples of a Cloud Foundry Application manifest and its discovery manifest
generated from performing the discovery on the source manifest:

```yaml
---
version: 1
space: default
applications:
- name: app1
  buildpacks:
  - ruby_buildpack
  - java_buildpack
  env:
    VAR1: value1
    VAR2: value2
  routes:
  - route: route.example.com
  - route: another-route.example.com
    protocol: http2
  services:
  - name: my-service1
  - name: my-service2
  - name: my-service-with-arbitrary-params
    binding_name: my-binding
    parameters:
      key1: value1
      key2: value2
  stack: cflinuxfs4
  metadata:
    annotations:
      contact: "bob@example.com jane@example.com"
    labels:
      sensitive: true
  processes:
  - type: web
    command: start-web.sh
    disk_quota: 512M
    health-check-http-endpoint: /healthcheck
    health-check-type: http
    health-check-invocation-timeout: 10
    instances: 3
    memory: 500M
    log-rate-limit-per-second: 1KB
    timeout: 10
  - type: worker
    command: start-worker.sh
    disk_quota: 1G
    health-check-type: process
    instances: 2
    memory: 256M
    log-rate-limit-per-second: 1KB
    timeout: 15
- name: app2
  env:
    VAR1: value1
  processes:
  - type: web
    instances: 1
    memory: 256M
    log-rate-limit-per-second: 1KB
  sidecars:
  - name: authenticator
    process_types: [ 'web', 'worker' ]
    command: bundle exec run-authenticator
    memory: 800M
  - name: upcaser
    process_types: [ 'worker' ]
    command: ./tr-server
    memory: 2G
```

When this manifest is run through the discovery process, it will generate a discovery manifest for each application discovered. For this
example both applications are combined into a single yaml:

```yaml
---
name: app1
space: default
labels:
    sensitive: "true"
annotations:
    contact: bob@example.com jane@example.com
version: "1"
env:
    VAR1: value1
    VAR2: value2
routes:
    - route: route.example.com
    - route: another-route.example.com
      protocol: http2
services:
    - name: my-service1
    - name: my-service2
    - name: my-service-with-arbitrary-params
      parameters:
        key1: value1
        key2: value2
      bindingName: my-binding
processes:
    - type: web
      command: start-web.sh
      disk: 512M
      memory: 500M
      healthCheck:
        endpoint: /healthcheck
        timeout: 10
        interval: 30
        type: http
      readinessCheck:
        endpoint: /
        timeout: 1
        interval: 30
        type: process
      instances: 3
      logRateLimit: 1KB
    - type: worker
      command: start-worker.sh
      disk: 1G
      memory: 256M
      healthCheck:
        endpoint: /
        timeout: 1
        interval: 30
        type: process
      readinessCheck:
        endpoint: /
        timeout: 1
        interval: 30
        type: process
      instances: 2
      logRateLimit: 1KB
stack: cflinuxfs4
timeout: 60
buildPacks:
    - ruby_buildpack
    - java_buildpack
instances: 1
---
name: app2
space: default
version: "1"
env:
    VAR1: value1
processes:
    - type: web
      memory: 256M
      healthCheck:
        endpoint: /
        timeout: 1
        interval: 30
        type: port
      readinessCheck:
        endpoint: /
        timeout: 1
        interval: 30
        type: process
      instances: 1
      logRateLimit: 1KB
sidecars:
    - name: authenticator
      processType:
        - web
        - worker
      command: bundle exec run-authenticator
      memory: 800M
    - name: upcaser
      processType:
        - worker
      command: ./tr-server
      memory: 2G
timeout: 60
instances: 1
```

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