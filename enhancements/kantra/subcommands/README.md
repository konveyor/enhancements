---
title: Kantra Subcommands and Analysis Config
authors:
  - "@eemcmullan"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2025-11-03
last-updated: 2025-11-06
status: implementable
---

# Kantra CLI Refactor

This enhancement introduces a refactor of the Kantra CLI to include a new set of 
subcommands and a configuration file for analysis options.


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created


## Summary

In the first iteration of the Kantra CLI, the goal was feature parity with 
Windup https://github.com/windup/windup with two major subcommands: `analyze` and 
`transform` for Java applications. 

Currently, the scope of Kantra has expanded to analyze multiple language applications,
gather insights and modify applicaitons running on platforms, and communicate 
with the Konveyor Hub for synchronized appliction analysis configurations.

We can improve the user experience of the Kantra CLI by updating the subcommands 
that make sense today. We can also set all analysis options from a config file, 
similar to the [centralized config](https://github.com/konveyor/enhancements/pull/241) 
anaysis profile. This file will include analyzer `providerConfig` options, as 
well as CLI analysis options.


### Goals

- Refactor the CLI subcommands and flags so that each subcommand is
  self-contained, improving discoverable actions for users.
- Include the ability to configure analysis options from a file.
- Maintain backwards compatibility.

### Non-Goals

- Discuss moving non-anlysis related subcommands.


## Proposal

The Kantra CLI will include the following subcommands:

- `kantra analyze`

- `kantra rules`

- `kantra config`

- `kantra provider`

- `kantra transform`

- `kantra discover`

- `kantra generate`

The Kantra CLI will also have the ability to read provider and analysis options
from a config file.


## Implementation Details

### Subcommands:

#### analyze

`kantra analyze` will only be responsible for options involving running analysis.
Such as:

- `kantra analyze --config-file <path/to/config> --output <output-dir>`

*There will also be an undetermined default path to this file.*

- `kantra analyze --input <path/to/app> --output <output-dir> --target <target>`

#### rules

`kantra rules` will provide options for interacting with rules and rulesets for 
analysis configuration. This includes both default rulesets and custom rulesets. 

The current `test` subcommand will fall under this new subcommand.

- `kantra rules --list-targets`

- `kantra rules test <path-to-test-rules>`

#### config

`kantra config` will be responsible for options to configure analysis config 
files and profiles. This will include:

- `kantra config login`

- `kantra config --sync <app>`

- `kantra config --list-profiles`

#### provider

`kantra provider` will handle functionality around using providers, including 
installing and removing providers. This will be included once Kantra supports 
[socket communication](https://github.com/konveyor/enhancements/pull/231).

- `kantra provider --list-providers`

- `kantra provider --install <provider>`

- `kantra provider --remove <provider`

#### transform

`kantra transform` will remain as is.

#### discover

`kantra discover` will remain as is.

#### generate

`kantra generate` will remain as is.


### Analysis Config File

Below is an example config file that can be used to read and set all
analysis options from the provider, as well as any CLI analysis options.

```yaml
providerConfig:
  name: "java"
  binaryPath: "/path/to/java/binary"
  useSocket: true
  initConfig:
    location: "/path/to/application/source/code"
    analysisMode: "full"
    providerSpecificConfig:
      name: "java" 
      bundles: "/path/to/bundle"
      lspServerPath: "/path/to/language/server/binary"
      depOpenSourceLabelsFile: "/path/to/maven.default.index"
      dependencyProviderPath: "/path/to/dependency/provider/binary"
analysisConfig:
  rules:
    targets:
    - "cloud-readiness"
    - "quarkus"
    sources:
    - ""
    labelSelectors:
    - ""
    rulesets:
    - "path/to/custom-rules"
    - "path/to/other/custom-rules"
    useDefaultRules: false
    analyzeKnownLibraries: true
  scope:
    depAanlysis: true
    withKnownLibs: false
    incidentSelector: "!package=com.example.apps"
  output:
    location: "path/to/output-dir"
    skipStaticReport: false
    overwrite: true
    jsonOutput: false

```

*A profile config will take priority over the CLI analysis config.*.  

*Analysis input flags will take priority over the CLI analysis config.*


### Backwards Compatibility 

To ensure backwards compatibility with the current subcommands, the Kantra CLI will 
accept current input values that will then be set to the refactored subcommand/option. 
For example:

`kantra analyze --list-targets`

The refactored CLI will read this input to the new `rules` subcommand. 


## Test Plan

- Verify analysis results from passing in config for analysis options.

- Verify results from new subcommands, such as `list-providers` and `list-targets`.

- Verify backwards compatibility with previous subcommands.


## Drawbacks

Some current CLI options will be moved to a new subcommand.
