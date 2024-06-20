---
title: CLI for Konveyor
authors:
  - "@ashokponkumar,@HarikrishnanBalagopal"
reviewers:
  - ""
approvers:
  - ""
creation-date: 2022-09-21
last-updated: 2022-09-21
status: implementable
see-also:
  - "N/A"
replaces:
  - "N/A"
superseded-by:
  - "N/A"
---

# CLI for Konveyor

Create a unified CLI for the projects in the Konveyor community.

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined

## Summary 

At a high level this tool will 
- improve discoverability of the various projects in Konveyor and the CLI or UI tools they provide.
- provide a simple uniform process to install, invoke, update and uninstall multiple versions of all the tools.
- optionally, we can provide static code analysis and recommendations on what tools to use.

## Motivation

- Currently there are a lot of projects under the Konveyor community. Several of these projects produce tools to help users transform their apps. However, the end-user the experience is still that of interacting with several completely different projects.
- Recently Konveyor got accepted into CNCF as a Sandbox project. This gives us the opportunity to coalesce all the tools under a single umbrella.
- A single CLI tool can help give the end-user a seamless experience similar to the ones provided by Kubectl, IBM Cloud CLI, AWS CLI, etc.

## Non-goals

- This tool is not meant to be a replacement for the UI/Hub workflow.

## Proposal

There are three key areas for this tool to work on:

1. First, being one of the main ways users discover the tools. This will require the tools to be made availabe as plugins.
2. Second, implement a plugin manager to manage the life-cycle of a plugin. This reduces the barrier to entry. Prototype: https://github.com/konveyor/cli
3. Optional: Static code analysis engine to look at the user code and ask them a few question to decide which tools to recommend for their use case. This can look at stuff like programming languages and frameworks being used, databases being used, whether or not they have Dockerfiles, Kubernetes Yamls, Helm charts, Cloud Foundry yamls, Terraform yamls, etc.


This enhancement will be followed by an in-depth dive into what challenges
are involved in on-boarding new users and helping them test things out locally
before committing themselves to a particular solution.


The CLI also creates an appropriate directory structure to store all the plugins:

```
 $HOME
 |- .konveyor
 |  |- cache.yaml
 |  |- plugins
 |  |   |- <plugin-1-name>.yaml
 |  |   |- <plugin-1-name>
 |  |   |   └- v1.2.3
 |  |   |       └- linux-amd64
 |  |   |           |- plugin-1.tar.gz
 |  |   |           └- plugin-1/
 |  |   |               └- plugin-1-executable
 |  |   |               └- my-file-1
 |  |   |               └- my-file-2
 |  |   |- <plugin-2-name>.yaml
 |  |   └- <plugin-2-name>
 |  |       └- v2.3.4
 |  |           └- linux-amd64
 |  |               |- plugin-2.tar.gz
 |  |               └- plugin-2/
 |  |                   └- plugin-2-executable
 |  |                   └- my-file-3
 |  |                   └- my-file-4
```

We can create plugins in 2 ways:

- rename an executable with the prefix `konveyor-` and place it in the PATH (similar to Kubectl plugins)
- create a `<plugin-name>.yaml` file and add it to the CLI Github repo in the `plugins/` folder via a PR.

Plugin metadata yaml file format:
```
apiVersion: cli.konveyor.io/v1alpha1
kind: Plugin
metadata:
  name: move2kube
spec:
  homePage: https://move2kube.konveyor.io
  docs: https://move2kube.konveyor.io/concepts
  tutorials: https://move2kube.konveyor.io/tutorials
  shortDescription: Move2Kube creates all the resources required for deploying your application to Kubernetes.
  description: |
    Move2Kube creates all the resources required for deploying your application to Kubernetes.
    It supports translating from docker swarm/docker-compose, cloud foundry apps and even other non-containerized applications.
    Even if the app does not use any of the above, or even if it is not containerized it can still be transformed.
  versions:
    - version: v0.3.4
      platforms:
        - selector:
            matchLabels:
              os: darwin
              arch: amd64
          uri: https://github.com/konveyor/move2kube/releases/download/v0.3.4/move2kube-v0.3.4-darwin-amd64.tar.gz
          sha256: f58389b8116d43707ae8fb1d22dbc4ee23361f6dc7283d8d188d3723360690e8
          bin: move2kube/move2kube
        - selector:
            matchLabels:
              os: darwin
              arch: arm64
          uri: https://github.com/konveyor/move2kube/releases/download/v0.3.4/move2kube-v0.3.4-darwin-arm64.tar.gz
          sha256: b54e5b685035a14b588111058c7d238be094d6527a5b7ff7ff2bd3d381245bf5
          bin: move2kube/move2kube
        - selector:
            matchLabels:
              os: linux
              arch: amd64
          uri: https://github.com/konveyor/move2kube/releases/download/v0.3.4/move2kube-v0.3.4-linux-amd64.tar.gz
          sha256: 250e8c5aad8e821849e5b35bff9089b531cfaa8cef27124504d02da4a256f0dc
          bin: move2kube/move2kube
        - selector:
            matchLabels:
              os: windows
              arch: amd64
          uri: https://github.com/konveyor/move2kube/releases/download/v0.3.4/move2kube-v0.3.4-windows-amd64.tar.gz
          sha256: b80687e554f98bcf8f012b6c363be536245dce53a175afdbe54ac7bf9efdfa11
          bin: move2kube/move2kube
```

The provides UX like Kubectl plugins, Krew and Helm repo.
