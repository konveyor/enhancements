---
title: Crane 2.0 Plugin Passthrough 
authors:
  - "@shawn-hurley"
reviewers:
  - "@jmontleon"
  - "@alaypatel07"
  - "@sseago"
approvers:
  - "@alaypatel07"
creation-date: 2021-04-30
last-updated: 2021-04-30
status: implementable
see-also:
  - "N/A" 
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# Crane 2.0 Plugin Passthrough


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined


## Open Questions [optional]

1. How to handle a conflict on names of flags?
2. Should some of the information be used in a config file in the future?

## Summary 

A plugin developer wants the ability to ask for more information to more fully create the correct transformations for
a kubernetes manifest.
Because plugins **MUST NOT** have access to clusters, a plugin user must give the information that in has previously determined from a cluster. 

## Motivation

As a user creates and uses plugins, they will need to add extra information during the creation of transformations so
that the plugins can make more complete changes.
This information can range from information needed for the specific destination cluster or configuring the gitops/kustomize structure. This can also be used to add metadata or change important information like the image registry.
Plugins having no other side effect than a patch file and not requiring access to clusters is a direct motivation for
needing the user to provide the information.

### Goals

1. A plugin can register a well-defined set of information that a user **may** provide when running a transformation.
2. A plugin must handle some set of information not being passed, either with sane default, by ignoring that transformation.
3. Users of the transformation CLI **MUST** pass this information in a repeatable and standard way.

### Non-goals

1. Grant plugins can interact with each other or see changes that each other are making.
2. Handle a plugin that must have access outside of the disk.
3. Handle permissions for what that plugin can do; it is the responsibility of the user to trust a plugin.

## Proposal

We will provide a call via standard-in for a plugin to tell the `BinaryPlugin` type that it has specific optional fields as strings. If the plugin is expecting some numeric type, they will have to validate it on their side. A particular plugin error is created when this validation fails to communicate this type of failure. 

We will create a Standard-In call, `Metadata`, which when called will cause a binary plugin to print out the `PluginMetadata` type. The Plugin library will be responsible for this and will be responsible for determining the versions of Response and Request's that it can handle. We will also add a version for the plugin itself and a name of the plugin. It will also return with a set of optional information that could be added.

### User Stories [optional]

#### Story 1

As a user, I would like to change the registry information for a set of registries like `crane transform --optional="--registry-replace quay.io/testing->registry.redhat.io/testing"`.

#### Story 2

As a user, I would like to see all the optional flags I can pass in for a set of transformations by calling `crane transform optionals`.

The output should be:

```
     --registry-replace string Plugin-in: kubernetes Version: v1 Will find all the images in an object with a pod spec to change the image registry. An example is quay.io/testing->registry.redhat.io/testing
```

#### Story 3

As a plugin author, I should expect that the extra information is added to the `Run` method while determining the patches that I would like to create.

#### Story 4

As a plugin author, I want to state the input and out that a given version of my plugin can use and that they can see this in a listing.  `crane plugins` output:
```
<Plugin Name>   <Version>   <request version>   <response version>
```

### Implementation Details/Notes/Constraints [optional]

1. We have to handle the fact that we don't know where to load the plugins before the command is executed.
2. We have to handle validation failures at a lower level and communicate them back.
3. We have to handle mismatches in plugin versions and communicate clearly. 


### Risks and Mitigations


## Design Details

A new type in the plugin package will be created 

```go 
    type PluginMetdata struct {
        Name string
        Version string
        RequestVersion RequestVersion
        ResponseVersion ResponseVersion
        OptionalFields []string
    }

    type RequestVersion string

    const(
        v1 RequestVersion = "v1"
    ) 

    type ResponseVersion string

    const(
        v1 ResponseVersion = "v1"
    )
```

When creating a plugin, you will also have to pass in Name, version, and OptionalFields in the CLI package

```go
func NewCustomPlugin(name, version string, optionalFields []string, runFunc func(*unstructured.Unstructured) (transform.PluginResponse, error))
```

We will also unexport the CustomPlugin type.

### Test Plan

Part of this effort will be adding e2e tests to validate that the output of a given set of commands conforms to what we would expect. This effort should be focused on writing simple ginkgo and gomego tests. 