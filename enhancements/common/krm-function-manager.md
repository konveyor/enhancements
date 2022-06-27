---
title: krm-function-manager
authors:
  - "@JonahSussman"
reviewers:
  - "@shawn-hurley"
  - "@djzager"
approvers:
  - "@shawn-hurley"
  - "@djzager"
creation-date: 2022-06-22
last-updated: 2022-06-23
status: provisional
see-also:
  - "KEP-2906: Kustomize Function Catalog"  
replaces:
  - "N/A"  
superseded-by:
  - "N/A"  
---

- [Kaffine: A KRM Function Manager](#kaffine-a-krm-function-manager)
  - [Release Signoff Checklist](#release-signoff-checklist)
  - [Open Questions](#open-questions)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
  - [Design Details](#design-details)
    - [Workflow](#workflow)
  - [Proposal](#proposal)
    - [User Stories](#user-stories)
      - [Story 1: Creating a new application](#story-1-creating-a-new-application)
      - [Story 2: Updating an existing application already using Kaffine](#story-2-updating-an-existing-application-already-using-kaffine)
    - [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints)
      - [Notes](#notes)
      - [CLI Commands](#cli-commands)
  - [Implementation History](#implementation-history)
  - [Drawbacks](#drawbacks)
  - [Alternatives](#alternatives)

# Kaffine: A KRM Function Manager

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

1. What is the exact format of the outputted data?
2. How could the system be extended to include the management of entire pipelines?

## Summary

This enhancement proposes a new tool, called **Kaffine** to help manage the discovery, installation, removal, and updating of KRM functions. The tool acts as a buffer between the low-level management of different KRM function versions and the high-level usage of them in an application.

This proposal utilizes [KEP-2906: Kustomize Function Catalog](https://github.com/kubernetes/enhancements/tree/master/keps/sig-cli/2906-kustomize-function-catalog#motivation) as a basis for its catalog system.

## Motivation

Currently, when creating an application that utilizes KRM functions, there is no standardized way to manage the versioning these functions. While it is possible to manage the versioning of the functions manually, it is incredibly un-ergonomic to do so. The simplest solution to avoid managing versions is to use the `latest` tag when developing the application. Unfortunately, this produces three major undesirable effects:

1. **Lack of reproducibility**: A developer using `latest` might start out with `v1` on their system. However, when they go to deploy on a different system, `latest` might pull a higher version with breaking changes.
1. **Security vulnerabilities**: Many enterprise solutions will simply not tolerate blindly updating to the latest version of a technology before it is proven.
1. **Dependant on container technology**: While KRM functions utilize container technology for now, if that requirement is ever relaxed to include raw binaries, relying on tags will simply not work. 

The other option is to lock down every instance of the function's usage to a specific version. While this solves some of the issues with `latest`, this method has issues of its own:

1. **Lack of version mobility**: Developers will have to update every single reference to the function to the newer version, causing a refactoring headache.
1. **Security vulnerabilities (still)**: A newer version of the function may be released with critical updates and bugfixes
1. **Auditability**: If the functions are defined in one place, you don't accidentally use two different versions, which may cause conflicts.

Additionally, as it currently stands there is no way to easily discover new KRM functions. While KPT has a [centralized function catalog](https://catalog.kpt.dev/), this severely limits the growth potential and adoption of KRM functions. 

This enhancement proposes a new tool, Kaffine, to help rectify these issues. Much like how one can install system packages manually using tools like dpkg, it is the norm to let apt or dnf manage these installations instead. Additionally, one can add additional catalogs to these tools to gain access to more functions, managed by the community.

### Goals

- Be able to add or remove catalogs of KRM functions to the tool
- Search the catalogs for functions and install them
- Update the installed functions
- Easily opt-in or opt-out of the system

### Non-Goals

- Running KRM functions
- Managing how applications use functions

## Design Details

A function catalog, going off of [KEP-2906](https://github.com/kubernetes/enhancements/tree/master/keps/sig-cli/2906-kustomize-function-catalog#motivation), is a YAML file listing one or more KRM functions, along with their associated versions. It can be a local file, a remote reference available from an HTTP(s) endpoint, or as an OCI artifact.

<!-- Is this a good idea? -->
Unless otherwise specified, all files are _project local_. All files and config will be stored in a local `.kaffine` directory. There is also a global config, which has a lower precedence than the local one.

### Workflow

<!-- Need to create an OpenAPI spec for config.yaml? -->
First, a user makes Kaffine aware of a catalog via `kaffine config add catalog <catalog>`. The catalog is appended to the `catalogs` list in the `config.yaml` file. The catalog is then saved in the `catalogs/` folder, with a name corresponding to the hash of the location. (`"example.com/catalogs/one" -> catalogs/<SHA-256>`)

Next, a user can search the catalogs for a function via `kaffine search <name>`. This searches the catalogs for a function with the specified name. Once the user finds a function they want to install, they can execute `kaffine install <name>`. This will add the function to a separate catalog stored in `installed.yaml`.

If at any point the user wants to check for updates to their functions, they can execute `kaffine update`. This will check if there is an updated version of each catalog, and download it if need be. Then, if there are any functions with higher versions than the ones selected by kaffine, it will prompt the user and ask if they want to update.

## Proposal

### User Stories

The following catalog is published at `example.com/catalogs/catalog.yaml` (adapted from KEP-2906):

```yaml
apiVersion: config.kubernetes.io/v1alpha1
kind: Catalog
metadata: 
  name: "example-co-functions"
spec: 
  krmFunctions: 
  - group: example.com
    names:
      kind: JavaApplication
    description: "A function that can handle Java apps"
    versions:
    - name: v2.0.0
      runtime: 
        container: 
          image: docker.example.co/functions/java:v2.0.0
    - name: v1.0.0
      runtime: 
        container: 
          image: docker.example.co/functions/java:v1.0.0
  - group: example.com
    names:
      kind: Logger
    description: "A function that adds our bespoke logging"
    versions:
    - name: v1.0.2
      runtime: 
        container: 
          image: docker.example.co/functions/logger:v1.0.2
    - name: v1.0.1
      runtime: 
        container: 
          image: docker.example.co/functions/logger:v1.0.1
    - name: v1.0.0
      runtime: 
        container: 
          image: docker.example.co/functions/logger:v1.0.0
  - group: example.com
    names:
      kind: SecretSidecar
    description: "A function that adds our bespoke secret sidecar"
    versions:
    - name: v3
      runtime: 
        container: 
          image: docker.example.co/functions/secrets:v3.0.0
```

#### Story 1: Creating a new application

Alice has in mind some functions that she wants to use that are created by Group Alpha. She adds the catalog their catalog via `kaffine config add catalog example.com/catalogs/catalog.yaml`. Then she searches which functions are available in the catalog.

```
$ kaffine search
KRM Functions:
NAME            DESCRIPTION                                     LATEST INSTALLED
JavaApplication A function that can handle Java apps            v2.0.0    ---
Logger          A function that adds our bespoke logging        v1.0.2    ---
SecretSidecar   A function that adds our bespoke secret sidecar v3        ---

$ kaffine install JavaApplication
Installed example.com/JavaApplication:v2.0.0

$ kaffine install Logger
Installed example.com/Logger:v1.0.2

$ kaffine list
KRM Functions:
NAME            DESCRIPTION                              INSTALLED
JavaApplication A function that can handle Java apps     v2.0.0
Logger          A function that adds our bespoke logging v1.0.2

$ kaffine list -o yaml
apiVersion: config.kubernetes.io/v1alpha1
kind: Catalog
metadata: 
  name: "kaffine-functions"
spec: 
  krmFunctions: 
  - group: example.com
    names:
      kind: JavaApplication
    description: "A function that can handle Java apps"
    versions:
    - name: v2.0.0
      runtime: 
        container: 
          image: docker.example.co/functions/java:v2.0.0
  - group: example.com
    names:
      kind: Logger
    description: "A function that adds our bespoke logging"
    versions:
    - name: v1.0.2
      runtime: 
        container: 
          image: docker.example.co/functions/logger:v1.0.2
```

She can now pass either the output of `kaffine list -o yaml` or the contents of `.kaffine/installed.yaml` to another application.

#### Story 2: Updating an existing application already using Kaffine

Bob wants to check if there are any updates to the functions he has installed.

```
$ kaffine list
KRM Functions:
NAME            DESCRIPTION                                     INSTALLED
JavaApplication A function that can handle Java apps            v1.0.0
Logger          A function that adds our bespoke logging        v1.0.0
SecretSidecar   A function that adds our bespoke secret sidecar v3    

$ kaffine update --check
Checking for updates...
example.com/JavaApplication v1.0.0 -> v2.0.0
example.com/Logger v1.0.0 -> v1.0.3
Checked for updates without installing.

$ kaffine update JavaApplication
example.com/JavaApplication v1.0.0 -> v2.0.0

$ kaffine update Logger
example.com/Logger v1.0.0 -> v1.0.3

$ kaffine list
KRM Functions:
NAME            DESCRIPTION                                     INSTALLED
JavaApplication A function that can handle Java apps            v2.0.0
Logger          A function that adds our bespoke logging        v1.0.3
SecretSidecar   A function that adds our bespoke secret sidecar v3    
```

### Implementation Details/Notes/Constraints

#### Notes

Two functions may have the same name. If you want to disambiguate between them in, you can prepend the group that the function is in followed by a `/` character. Example: `example-group/function-bar`.

Additionally, if you want to install a specific version (such as `v1.1`, `latest-stable`, `nightly`, et cetera…) you can append a `:` character followed by the label. Example: `function-bar:nightly`. If you do not provide a version, it will default to the highest one available.

<!-- MAYBE ALSO A GO LIBRARY TO MAP FUNCTION NAMES TO LOCATIONS USING CONFIG -->
#### CLI Commands

**Search**

```
kaffine search
kaffine search <function-name>
```

Searches all catalogs for the function with the specified name. If no name is provided, it’s functionally the same as: `kaffine search '*' -r`. If output is piped, `-o yaml` is automatically selected. Otherwise, `-o default` is the default.

_Options_

```
--regex, -r :: Treats <function-name> as a regular expression
--output, -o <format> :: Format to output, ex. yaml, default, etc...
--group, -G <group> :: Only matches functions with the specified group.
```

<!-- NOTE: pipelines and collections are for the future, don’t worry about them right now -->

_Examples_
```
$ kaffine search foo
KRM Functions:
NAME                   DESCRIPTION                           LATEST  INSTALLED
foo-function           Generic foo function                  v2.0.0     ---
foo-function-binary    Generic foo function, using a binary  v1.0.1     ---
make-fooable-pipeline  Makes resources fooable               v0.0.1     ---
frobnicate-the-foos    Frobnicates the foos                  v3.0.1     ---
```

<!-- Using something like KEP-2906: Kustomize Function Catalog. Bare bones, probably would want to bring in line with KEP-2906 -->

```
$ kaffine search foo -o yaml

apiVersion: config.kubernetes.io/v1alpha1
kind: Catalog
metadata: 
  name: "kaffine-catalog"
spec: 
  krmFunctions: 
  - group: catalog-1.com
    names:
      kind: foo-function
    description: "Generic foo function"
    versions:
    - name: v2.0.0
      runtime: 
        container: 
          image: docker.example.co/functions/foo-function:v2.0
    - name: v1.0.0
      ...
  - group: catalog-2.com
    names:
      kind: foo-function-binary
    description: "Generic foo function, using a binary"
    versions:
    - name: v1.0.1
      runtime: 
         exec:
          platforms:
          - bin: foo-amd64-linux
            os: linux
            arch: amd64
            uri: https://example.com/foo-amd64-linux.tar.gz
            sha256: <hash>
          - bin: foo-amd64-darwin
            os: darwin
            arch: amd64
            uri: https://example.com/foo-amd64-darwin.tar.gz
            sha256: <hash>
    - name: v1.0.0
      ...
```

**List**
```
kaffine list
```

Lists installed functions. If output is piped, `-o yaml` is automatically selected. Otherwise, `-o default` is the default.

_Options_
```
--output, -o <format> :: Format to output, ex. yaml, default, etc...
```

**Install**
```
kaffine install <function-name>
```

Tries to install the function with the given name. It searches the catalogs for the function with the given name. If it finds it, it adds it to the catalog in `.kaffine/installed.yaml`. If there are duplicates, it will return the duplicate functions.

**Remove**
```
kaffine remove <function-name>
```

Uninstalls the function with the given name

**Update**
```
kaffine update
kaffine update <function-name>
```

If no function name is provided, then it will simply update every function installed.

_Options_
```
--check, -c :: Just checks if there is an update, without overwriting
--version, v :: Installs a specific version
```

**Config**

```
kaffine config add catalog <URI>
kaffine config remove catalog <Name/URI>
kaffine config view
```

Updates the config stored in `.kaffine/config.yaml`. When adding a new catalog, the catalog is appended to the `catalogs` list in the file. Then, the catalog is saved in the `.kaffine/catalogs/` folder, with a name corresponding to the hash of its location (`"example.com/catalogs/one" -> catalogs/<SHA-256>`).

_Options_
```
--global, -g :: Updates the global config, as opposed to the local one
```

<!-- ### Security, Risks, and Mitigations -->



<!-- ### Test Plan -->

<!-- ### Upgrade / Downgrade Strategy -->

## Implementation History

## Drawbacks

- Introducing another tool may lead to unnecessary complexity in simple projects

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

<!-- ## Infrastructure Needed [optional] -->