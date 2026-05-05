---
title: Crane 2.0 Discover and Extract
authors:
  - "@alaypatel07"
reviewers:
  - "@shawn-hurley"
approvers:
  - "@shawn-hurley"
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

# Crane 2.0 Discover and Extract

The Crane 2.0 effort is focused on creating a toolkit of small utilities to
help users backup and restore applications deployed on Kubernetes. The goal
of this document is to describe the goals on one specific command being
developed to discover and extract YAML dump of resources that are deployed
in the cluster. 

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined

## Summary 

At a high level this tool will 
- can be used by anyone who has access to resources in the namespace
- use the Kubernetes discovery API to find the GVK's available in a cluster
- will create a YAML dump of all the namespaced resources in a well-defined 
  directory structure. This directory structure will form the basis of how
  the YAMLs can be transformed and applied later on to get repeatable
  deployments.


## Motivation

The very first problem of backuping up applications on kubernetes is to
discover and extract the GVK's available in the cluster that needs to be
backed up. A conscious decision is made here to make sure to design this
tool with a restricted privileges of non-admin user. It will be more
empowering if the namespace owners backup the applications that they want
/need to, instead of requiring cluster-wide admin privilege to backup
the namespace. 

## Non-goals

- This tool will not be intended for a cluster admin, hence it will not backup
any GVKs that are not namespace scoped.

## Proposal

There are three key areas for this tool to work on, first being the discovery
 API to find the namespace scoped resources. Upstream Velero also uses the
Kubernetes discovery API. This tool will import the discovery package of
Velero to implement getting the resources. This enhancement will be followed
by and indepth dive into what challenges are involved in using the velero
package for Crane 2.0. During the extraction of resources, this tool will
skip "virtual resources" (resources that do not support "create" verb) as
well as resources that do not have "list" verb.

The second part of this tool is to create appropriate directory structures
 so that it can be reused. The directory structure proposed is as follows:
 
```
 export-dir
 |- resources
 |  |- namespace-1
 |  |   |- kind-a__name.yaml
 |  |   |- kind-a_name-1.yaml
 |  |   |- kind-b_name.yaml
 |  |- namespace-2
 |  |   |- kind-a_name.yaml
 |  |   |- kind-a_name-1.yaml
 |  |   |- kind-b_name.yaml
 |- failures
 |  |- namespace-1
 |  |   |- failed-kind.yaml
```

If the API call to dump the YAML failed for some resource, it will be in the 
`failures` directory. Example of a failure YAML is as follows:

```
apiresource:
  name: bindings
  singularname: ""
  namespaced: true
  group: ""
  version: ""
  kind: Binding
  verbs:
  - create
  shortnames: []
  categories: []
  storageversionhash: ""
err:
  errstatus:
    typemeta:
      kind: Status
      apiversion: v1
    listmeta:
      selflink: ""
      resourceversion: ""
      continue: ""
      remainingitemcount: null
    status: Failure
    message: the server does not allow this method on the requested resource
    reason: MethodNotAllowed
    details:
      name: ""
      group: ""
      kind: ""
      uid: ""
      causes: []
      retryafterseconds: 0
    code: 405    
```

The third part is providing a UX like kubectl. This tool will try to read the
default kube context, along with providing a `--namespace` flag to override
the local context.

## Packages

### Discovery helper

- the input to this library will be an authenticated discovery client
 interface, and the namespace which needs to be backed up
- the output will be a 
  - map of GVK->List of objects present in the namespace
  - map of GVK->failures 

This will make sure once the library function is ran, the output from
this can be used to create the files with the directory structure proposed
earlier