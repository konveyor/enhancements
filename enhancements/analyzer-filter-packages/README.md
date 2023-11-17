---
title: Analyzer - Filtering Incidents
authors:
  - "@pranavgaikwad"
reviewers:
  - "@fabianvf"
  - "@jmle"
  - "@jortel"
  - "@rromannissen"
  - "@shawn-hurley"
approvers:
  - "@fabianvf"
  - "@jmle"
  - "@jortel"
  - "@rromannissen"
  - "@shawn-hurley"
creation-date: 2023-11-16
last-updated: 2023-11-16
status: implementable
---

# Analyzer - Filtering Incidents

In analyzer-lsp, we need ability to filter incidents based on the packages / modules they belong to. The exact semantics or the business logic of how that is implemented change from provider to provider. For instance, in Java, the filter will be applied on packages, similarly, in Python, it would be applied on modules. But the general idea remains the same. This enhancement proposes a new option in the analyzer CLI that takes these filters as inputs.

## Motivation

Every language provider will support filtering incidents based on some criteria around packages/modules, etc. Currently, a similar filter exists which does filtering of dependencies. The new proposed filter applies on all incidents. It will help users exclude / include things at the granularity of incidents. We are proposing a generic way of doing this via the CLI so it applies to all providers. A CLI option will allow users to configure it per analysis run. It will establish generic semantics to pass the input for all providers.

### Goals

- Propose a new option to take incident filter via CLI

- Discuss usage scenarios of the option and how it will work

### Non-Goals

- Discuss provider specific internal implementation details

## Proposal

Introduce a new flag `--incident-selector` to analyzer CLI. The input of this flag will essentially be a label selector expression just like existing options `--label-selector` or `--dep-label-selector`. 

### Usage scenarios

Lets take a look at how a user will use this option:

- As a user, I want to filter incidents so that I can see only incidents in certain packages/modules.
  - Java example: Only output incidents in Java packages matching pattern `io.konveyor.demo.*`:
    
    ```
    --incident-selector (package=io.konveyor.demo.*)
    ```
  - Python example: Only output incidents in Python modules matching pattern `io_konveyor_demo.*`:
    ```
    --incident-selector (module=io_konveyor_demo.*)
    ```

- As a user, I want to filter incidents so that I can see all incidents except the ones in certain packages/modules.
  - Java Example: Exclude all incidents in Java packages matching pattern `io.konveyor.demo.*`:
    ```
    --incident-selector (!package=io.konveyor.demo.*)
    ```
  - Python Example: Exclude all incidents in Python modules matching pattern `io_konveyor_demo.*`:
    ```
    --incident-selector (!module=io_konveyor_demo.*)
    ```

- As a user, I want to filter incidents so that I can see incidents in a package but exclude certain subpackages within. 
  - Java Example: Include all incidents in Java packages matching pattern `io.konveyor.demo.*` but exclude `io.konveyor.config-utils.*`:
    ```
    --incident-selector (package=io.konveyor.demo.* && !package=io.konveyor.config-utils.*)
    ```

### Technical details

This new option will essentially be a field selector that will work on an incident's _Variables_ field:


```go
type Incident struct {
	URI        uri.URI                `yaml:"uri" json:"uri"`
	Message    string                 `yaml:"message" json:"message"`
	CodeSnip   string                 `yaml:"codeSnip,omitempty" json:"codeSnip,omitempty"`
	LineNumber *int                   `yaml:"lineNumber,omitempty" json:"lineNumber,omitempty"`
  // --incident-selector will work on following field
  Variables  map[string]interface{} `yaml:"variables,omitempty" json:"variables,omitempty"`
}
```

_Variables_ field above is populated by providers and contains extra information about an incident. The providers are free to populate it with any additional information about an incident. One advantage of `--incident-selector` is that its generic in a way that providers' responsibility is to just add the fields. The evaluation of selector expression, however, will be delegated to the engine. Now let's take a look at how this will work with one of the Java examples we discussed earlier.

Assume that, in Java analysis, we get two incidents: 

```yaml
- URI: file://source/main/java/io/konveyor/demo/test.java

- URI: file://source/main/java/io/konveyor/demo/config-utils/test.java

```

The Java provider will add `package` field to _Variables_ of the incidents:

```yaml
- URI: file://source/main/java/io/konveyor/demo/test.java
  Variables:
    package: io.konveyor.demo

- URI: file://source/main/java/io/konveyor/demo/config-utils/test.java
  Variables:
    package: io.konveyor.demo.config-utils
```

Now, if the user specifies `--incident-selector='(!package=io.konveyor.demo.config-utils)'`, the engine will only produce the first incident:

```yaml
- URI: file://source/main/java/io/konveyor/demo/test.java
  Variables:
    package: io.konveyor.demo
```

Similarly, the Python provider can add a field `module` on incidents:

```yaml
- URI: file://venv/tensorflow/core/excluded_ops.py
  Variables:
    module: tensorflow.core
```

### Drawbacks

- The selector will only work on string fields. But the values of _Variables_ will be _interface{}_ type. We could probably add another field _Labels_ on incidents if we have to be strict enough in the way providers set these values. But for now, _Variables_ should be enough to start. 
