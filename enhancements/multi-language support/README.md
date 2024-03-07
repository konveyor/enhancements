---
title: Multi-language Support for Kantra Analysis
authors:
  - "@eemcmullan"
reviewers:
  - "@fabianvf"
  - "@shawn-hurley"
  - "@pranavgaikwad"
  - "@jortel"
  - "@mansam"
approvers:
  - "@fabianvf"
  - "@shawn-hurley"
  - "@pranavgaikwad"
creation-date: 2024-02-14
last-updated: 2024-03-07
status: implementable
---

# Multi-language Support for Analysis

## Summary

As of now, we can analyze Java applications with kantra using the analyzer-lsp and its Java provider. We want to expand this use case and support multiple language providers, thereby allowing users with different types of applications to perform application analysis. Multi-language support can be broken down into **three parts**: 
- Discovery of language(s) in an application 
- analyzer-lsp managing external language providers
- Each supported provider comes with default rulesets

### Motivation

With a growing interest in application modernization, tools such as kantra that allow for application analysis are becoming increasingly more necessary. To satisfy this need, we have to expand the types of applications that we can analyze.

### Goals

- Create the ability to discover application languages in kantra to determine providers that are necessary to run analysis.
- Enable all analyzer-lsp providers to be external (containing their own image and running independently). The analyzer-lsp Java provider will need to be moved out from the core analyzer flow.
- Write and provide default rulesets for each supported provider.

### Non-Goals

- Further discussion of community providers and rulesets
- Support for reaching providers by other methods than what is used today (e.g. URL endpoint)

## Proposal

Introduce multi-language support in kantra analysis by integrating a discovery feature in kantra to determine which application language(s) are being used, separate the analyzer-lsp providers out into external providers, and create default rulesets for each supported provider.

### Usage scenarios

Lets take a look at how a user will use this feature:

- As a user, I want to run analysis on my TypeScript application against the default set of TypeScript rules. Here, as part of the discovery process, a defult set of TypeScript rules will run against the input application as custom rules were not specified.

    ```
    kantra analyze --input=<my_ts_app> --output=<output_dir>
    ```

- As a user, I want to run analysis on my Python application against a custom Python rule.

    ```
    kantra analyze --input=<my_python_app> --output=<output_dir> --rules=<custom_rule_dir> --enable-default-rulesets=false
    ```
    
- As a user, I want to run analysis on my Java and TypeScript application against both default Java and TypeScript rules.

    ```
    kantra analyze --input=<my_java_ts_app> --output=<output_dir>
    ```
    
    
- As a user, I want to run analysis on my application that does not have a supported provider. 

    ```
    kantra analyze --input=<Rust_app> --output=<output_dir>
    ```
    
    - Output: 
    
    ```
    Error: Rust is not supported. Please see <link> for list of supported providers or provide a provider config with --unsupported-provider
    ```

## Design Details

### Provider Discovery

We can utilize https://github.com/devfile/alizer to discover application languages from kantra analyze source application.

```
import "github.com/devfile/alizer/pkg/apis/recognizer"

languages, err := recognizer.Analyze("source-application")
```

This will give an output similar to:

```
[
  {
    "Name": "Go",
    "Aliases": ["golang"],
    "Weight": 90.5,
    "Frameworks": [],
    "Tools": ["1.19"],
    "CanBeComponent": true
  }
]
```

We can then complete our discovery process by grabbing each language `Name` to store in a slice for later use. Further details on how kantra will handle each discovered provider will be discussed below in the `kantra changes` section. 


### analyzer-lsp External Providers

Each provider we support will remain in-tree with analyzer-lsp, but contain their own Dockerfile, entrypoints, and commands. This will enable these providers to run independently and in parallel. Also, with this provider structure, outside contributors can more easily add additional "community" providers.

```
├── demo-rules
├── docs
├── engine
├── examples
├── internal
│    ├── external-providers
│       ├── generic-external-provider
│            ├── go.mod
│            ├── go.sum
│            ├── cmd
│                ├── analyzer
│                └── dep
│            ├── pkg
│            ├── Dockerfile
│       ├── java
│            ├── go.mod
│            ├── go.sum
│            ├── cmd
│                ├── analyzer
│                └── dep
│            ├── pkg
│            ├── Dockerfile
├── provider
│   ├── grpc
│   ├── lib
```


Currently, the Java provider is baked into the analyzer as an "internal" provider. To begin the process of providers working as external providers, this will need to be pulled out into its own packages, as shown in the tree diagram above. 

The Java provider image will need to build with https://github.com/konveyor/java-analyzer-bundle to extend the Java language server to fully support the Java provider's capabilities. 

Part of enabling external providers will entail each provider having their own `provider`, `capabilities`, and `serviceClient` to interface with. These struct fields can be modified for each provider, depending on certain requirements and features for each language and language server. Example JavaScript provider:

```go
type JavaScriptProvider struct {
	ctx context.Context
}
```

```go
func (p *JavaScriptProvider) Capabilities() []provider.Capability {
	return []provider.Capability{
		{
			Name:            "referenced",
			TemplateContext: openapi3.SchemaRef{},
		},
		{
			Name:            "dependency",
			TemplateContext: openapi3.SchemaRef{},
		},
	}
}
```

```go
type JavaScriptServiceClient struct {
	rpc        *jsonrpc2.Conn
	cancelFunc context.CancelFunc
	cmd        *exec.Cmd

	config       provider.InitConfig
	capabilities protocol.ServerCapabilities
}
```

The `provider`s can then independently run as a server listenting for gRPC calls on a certain port, and the `serviceClient` will provide its language server-dependent `capabilities` to the provider server. These will most likely be different for each provider as many will support different methods.

### Kantra Changes

#### New Flags

A new flag can be introduced that will list the default supported providers: `kantra analyze --list-providers`

 ```sh
java-external-provider
python-external-provider
yaml-external-provider
...
..
 ```

To disable a provider, we can support an additional option `--disable-provider`. This will accept default supported providers, such as `--disable-provider java-external-provider`.
For example, for a Java and TypeScript application analysis (assuming both are default providers), but we only want to run a TypeScript analysis, the default Java provider can be disabled. 

To allow a user to run a community provider, as well as override a supported provider, `--unsupported-provider` will be available. This flag will be a mapping of the provider name to a yaml with the provider's container definition, and will be added to the podman pod spec during kantra's runtime (discussed more below).

`--unsupported-provider javaProvider="java_prov.yaml",rustProvider="rust_prov.yaml"`

#### Provider Mapping

In order to initialize the necessary providers for an application analysis, there will need to be a default mapping of providers to the language(s) found during the discovery process. 

```go
const (
  JavaExternalProvider = "java-external-provider"
  PythonExternalProvider = "python-external-provider"
)
```

```go
// used to map default provider to discovery language
var defaultProvidersFromDiscovery = map[string]string{
  JavaExternalProvider: "java",
  PythonExternalProvider: "python"
}
```

If a provider mapping is not found, kantra will look for `--unsupported-provider`. 

If a language is discovered that does not have a supported provider mapping, and `--unsupported-provider` is not set for this provider, kantra will log a warning and continue analysis with other providers.

#### Running the providers

To run each provider, we can start a podman pod with a container for the analyzer-lsp engine as well as an additional container for each provider that is found during the initial discovery process.

For each found provider, kantra will dynamically build a `providerSettings[]` config file at runtime. For providers supported by Konveyor, a `providerSettings` config will be kept in kantra for simple use. In the case of community providers, we will also require an additional flag to be set such as `--provider-settings` in which the user can set the appropriate config. 

Alongside the `providerSettings`, a yaml file will also be dynamically created to define the podman pod and each provider container. 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kantra
spec:
  containers:
    - name: analyzer-lsp
      image: quay.io/konveyor/analyzer-lsp:latest
      command:
      - /usr/local/bin/analyzer-lsp
      args:
      - "--provider-settings=/opt/settings.json", "--rules=/opt/rulesets/java"
    - name: java-provider
      image: quay.io/konveyor/java-provider:latest
      command:
      - /usr/local/bin/java-provider
      args:
      - "--port=8080"
```

As discussed previosuly, if using `--unsupported-provider`, this configuration will be added to the podman pod spec for community providers, and override the configuraton for supported providers.

Example analyze for a Java application, which would use a supported provider:

```
kantra analyze --input=<java_app> --output=<output_dir> --rules=<custom_rule_dir>
```

Example analyze for a Rust application, which would use a community provider:

```
kantra analyze --input <rust_app> --output <output_dir> --rules <custom_rule_dir> --provider-settings <community_provider_settings>
    --unsupported-provider rustProvider="rust_prov.yaml"
```

### Default Rulesets for Supported Providers

As of now, we have a set of default rules for the Java provider: https://github.com/konveyor/rulesets/
We will want to add more default rulesets for each provider. For delivery of these rulesets, they will be packaged in a known location in each providers' Dockerfile. In kantra, this/these path(s) will be set to `--rules`, unless this default behavior is disabled by the user.

There will be some rules that can be applicable to multiple providers. For example: 

```yaml
name: os/windows
description: This is a ruleset for Windows operating system specific rules while migrating
  to Linux operating system.
```

These multi-use rulesets can use labels such as `konveyor.io/provider=go` and `konveyor.io/provider=java` to filter the common rules for relevant providers. For each running provider, their individual rulesets can be evaluated, as well as searching in `rulesets/common` for any rules with these provider labels. 


## Open Questions/Thoughts

- Conditions that support multiple providers
- Community rulesets
- Location of providers' default rulesets
