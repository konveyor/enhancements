---
title: tackle-lsp-query-engine
authors:
  - "@fabianvf"
  - "@shawn-hurley"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2022-09-14
last-updated: 2022-09-14
status: implementable
see-also: []
replaces: []
superseded-by: []

---

# Konveyor Analysis: Multi-language analyzer leveraging Language Server Protocol (LSP)

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions [optional]


## Summary

Application analysis in order to facilitate modernization is one of the core use cases for Konveyor. The addition of the [Windup](https://github.com/windup/windup) project to leverage the existing work for static code analysis of Java applications was a step forward, but Konveyor needs to have analysis capabilities in multiple major languages in order to provide a broader value proposition. In order to fulfill this need, we propose leveraging the [Language Server Protocol](https://langserver.org/). The Languge Server Protocol is a powerful protocol originally built to provide generic static code analysis for integrated development environments (IDEs), so that each IDE would not need to build their own code analysis engines. By leveraging the Language Server Protocol, and existing Language Servers, we will be able to efficiently augment Konveyor's language analysis capabilities, without needing to build out full language parsers ourselves. As a first step we propose a standalone tool that can be run independently of the hub, but the design should allow for eventual integration into Konveyor hub as an addon.


## Motivation

As the windup project proved, building a static code analysis engine from scratch is a monumental task, and would be a difficult pattern to quickly scale out to every language that users would want to analyze. By using existing Language Servers, built by language experts and thoroughly tested and used in IDEs, we hope to sidestep a large subset of the problems that windup needed to solve. Instead, we can focus our efforts on a much smaller rules engine that can efficiently translate human readable and writeable code analysis rules into LSP queries, and skip the actual meat of the code analysis entirely.

### Goals

- Build a standalone tool that can take rules and a codebase as input, and return a machine-readable report.
- Build a rules engine that can translate rules into LSP queries
- Design an expressive rules format that is easy to comphrehend and write
- Ensure it is easy to integrate a new Language Server
- Ensure it is easy to extend the rules format to allow for language-specific queries
- Ensure the design remains compatible with Konveyor hub's API for eventual integration

### Non-Goals

- Design a global rule format for Konveyor Hub
- Design a report format for Konveyor Hub


## Proposal

The implementation would be comprised of several components.

### Rules schema
There should be a fairly generic rules schema that allows rules to be written targeting multiple languages. One rule by itself should not target multiple languages, but the format should not change significantly across languages in order to facilitate readability and reusability. The basic structure of a rule should contain a condition, or set of conditions, and an action. When the condition evaluates to true, the action will be performed. An action in this case could be something as simple as reporting a message highlighting an issue.

Example: in Python, a reference to a CRD can be found at `kubernetes.client.ApiextensionsV1beta1Api.create_custom_resource_definition`, whereas in Golang that same reference may be something like `k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1beta1.CustomResourceDefinition`. Even though the actual import path is not identical, both are made up of an import and a reference, so the rules could be expressed like:

```yaml
python:
    import: kubernetes.client.ApiextensionsV1beta1Api
    references:
        - create_custom_resource_definition
        - delete_custom_resource_definition
        - read_custom_resource_definition
        - update_custom_resource_definition
        - patch_custom_resource_definition
    message: deprecated use … instead

golang:
    import: k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1beta1
    references:
        CustomResourceDefinition
        CustomResourceDefinitionSpec
        CustomResourceDefinitionStatus
    message: deprecated use … instead
```

**NOTE** The format above is purely an example and not meant to represent the actual design of the rule format

The format should allow for extension to support language specific features that are currently unknown.

### Rules Engine

The Rules Engine will process the rules format defined above and determine which language providers will need to be run, ensure they are initialized, and properly delegate queries to the appropriate language providers. It will also be responsible for performing all the actions specified in the rules, the language providers will only be responsible for evaluating individual conditional statements.

### Language providers

There should be an interface, likely over HTTP or stdin/stdout JSONRPC, that allows for:
- Initialization of a Language Server with the desired codebase
- Ready checks to determine when requests can begin
- Communication with the Language Server

### User Stories [optional]

TODO

#### Story 1

#### Story 2

### Implementation Details/Notes/Constraints [optional]


### Security, Risks, and Mitigations

TBD

## Design Details

### Test Plan

TBD

### Upgrade / Downgrade Strategy

TBD

## Implementation History

TBD

## Drawbacks

TBD

## Alternatives

TBD


## Glossary

- LS - Language Server provides an API for a given language of the protocol
- LSP - Language Server Protocol - the definition of the spec that a language server must conform to.
- Condition - A statement that evaluates to True or False and is used to determine when to perform an action
- Rule -  is a set of information that contains a condition and action
- Providers - An specific implementation to process an individual condition, for a given language
- Rules Engine - break down the rules conditions and determine providers to call, and then take appropriate action based on the conditions for the rule. Report back in the report format.
- Ruleset - set of rules (useful for someone providing a set of rules as well as interchange formats)
- Addon - The current Job/CR that exists for Konveyor hub to run something for a given analysis.
