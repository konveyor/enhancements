---
title: Containerless Distributions of Kantra
authors:
  - "@eemcmullan"
reviewers:
  - "@pranavgaikwad"
  - "@rromannissen"
  - "@shawn-hurley"
  - "@djzager"
approvers:
  - TBD
creation-date: 2024-08-09
last-updated: 2024-08-11
status: implementable
---

# Containerless Distributions of Kantra

## Summary

Kantra is the Konveyor CLI used to analyze and transform applications. As of now, it requires a container runtime due to the analyze and transform commands running in containers. 

This enhancement will discuss a new implementation of kantra that does not require a container runtime for analysis. This enhancement will also only focus on the first phase of this implementation: a containerless distribution that will run a Java analysis. 

## Motivation

Kantra currently requires a container runtime. This presents problems for some users. 
For example, an organization may have a strict regulations that do not allow installation of container runtimes on company computers. This results in the kantra CLI being unavailable to assist these companies with konveyor analysis.

There is also the issue that the current implementation of kantra is incompatible with some CI/CD tools such as Jenkins container agents and Tekton tasks.

### Goals

- Discuss the need for kantra analysis to run without a container runtime
- Discuss the user requirements for this implementation
- Discuss the technial implementation required in kantra

### Non-Goals

- Discuss containerless distributions of kantra outside of Java analysis

## Proposal

### User Stories 

As a user, I want to run a Java application analysis using kantra that does not require installation of a container runtime.

As a developer, I want to set up CI/CD pipelines with kantra using tools such as Jenkins.

## Implementation Details

### Requirements

Users will be expected to have or install several tools onto their machines, such as OpenJDK and Maven. We will produce and ship our [rulesets](https://github.com/konveyor/rulesets/), JDTLS, and the [Java Bundle](https://github.com/konveyor/java-analyzer-bundle).

The user will be instructed to set these packaged binaries and rules in a known location, such as `$HOME/.kantra/`. Alternatively, an install script can set these.

## Design Details

### Technical Implmentation

A new analyze subcommand can be introduced for kantra to recognize when to start the containerized version of analysis versus the non-containerized option. This could look like `kantra analyze-java-only <options>`. Another option for starting the analyze subcommand without running containers would be to detect existing packaged requirements at the expected location discussed in the [requirements](#requirements). However, in this case, we would want to provide an option to disable this behavior.

There will be a new package in kantra, `no-container`, that will be added for this case. This package will import the [analyzer](https://github.com/konveyor/analyzer-lsp), [external Java provider](https://github.com/konveyor/analyzer-lsp/tree/main/external-providers/java-external-provider), and the [static-report](https://github.com/konveyor/static-report). 

```
.
├── README.md
├── cmd
│   ├── analyze.go
│   ├── analyzr-java.go
│   ├── .....
├── pkg
│   ├── container
│   └── no-container
│       │   ├── no-container.go
│       │   ├── static-report.go
│       ├── java
│       │   ├── internal-java-provider.go
│       ├── dotnet
│   ├── .....
```

With the analyzer and external Java provider imported, kantra will create and init an internal Java provider, which will be defined as an analyzer `InternalProviderClient`. This Java provider will not require an additional binary nor require running outside of the analyzer engine. It will be started as an internal provider rather than a GRPC provider.

A Java provider config will be created in kantra, which will include the `binaryPath` as the packaged JDTLS binary instead of the Java provider address or binary.

```yaml
    {
        "name": "java",
        "binaryPath":  "$HOME/.kantra/jdtls/bin/jdtls",
        "initConfig": [
            {
                "location": "<input>",
                "providerSpecificConfig": {
                    "bundles": "$HOME/.kantra/jdtls/java-analyzer-bundle/java-analyzer-bundle.core/target/java-analyzer-bundle.core-1.0.0-SNAPSHOT.jar",
                    "lspServerPath": "$HOME/.kantra/jdtls/bin/jdtls"
                },
                "analysisMode": "full"
            }]
    },
```

Kantra will then run the packaged rules and/or custom rules with this provider. Once the analysis is completed, a static report will be generated internally in the output directory.

### Alternatives

A new repository that will contain the analyzer-lsp as well as the Java provider imported, with the Java provider integrated internally. This approach would look similar to the above approach except kantra would import this project.

A third approach is to package the current Java external provder, and use that binary to run the Java provider in kantra, which would run it separately from the analyzer as a GRPC provider. This approach would however increase the complexity of the overall requirements.

### Test Plan

Compare Java analysis output results across Windows, MacOS, and Linux with ARM64 and AMD64.

### Drawbacks

This implementation will include more requirements and set-up for the user compared to the current implementation of kantra, which only requires a container runtime. See [requirements](#requirements) for further details.
