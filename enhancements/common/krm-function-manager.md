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
  - "N/A"  
replaces:
  - "N/A"  
superseded-by:
  - "N/A"  
---

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

## Motivation

Currently, when creating an application that utilizes KRM functions, there is no standardized way to manage the versioning these functions. While it is possible to manage the images of the functions manually, it is incredibly un-ergonomic to do so. The simplest solution to avoid managing versions is to use the `latest` tag when developing the application. Unfortunately, this produces three major undesirable effects:

1. **Lack of reproducibility**: A developer using `latest` might start out with `v1` on their system. However, when they go to deploy on a different system, `latest` might pull a higher version with breaking changes.
1. **Security vulnerabilities**: Many enterprise solutions will simply not tolerate blindly updating to the latest version of a technology before it is proven.
1. **Dependant on container technology**: While KRM functions utilize container technology for now, if that requirement is ever relaxed to include raw binaries, relying on tags will simply not work. 

The other option is to lock down every instance of the function's usage to a specific version. While this solves some of the issues with `latest`, this method has issues of its own:

1. **Lack of version mobility**: Developers will have to update every single reference to the function to the newer version, causing a refactoring headache.
1. **Security vulnerabilities (still)**: A newer version of the function may be released with critical updates and bugfixes

Additionally, as it currently stands there is no way to easily discover new KRM functions. While KPT has a [centralized function catalog](https://catalog.kpt.dev/), this severely limits the growth potential and adoption of KRM functions. 

This enhancement proposes a new tool, Kaffine, to help rectify these issues. Much like how one can install system packages manually using tools like dpkg, it is the norm to let apt or dnf manage these installations instead. Additionally, one can add additional repositories to these tools to gain access to more functions, managed by the community.

### Goals

- Be able to add or remove repositories of KRM functions to the tool
- Search the repositories for functions and install them
- Update the installed functions
- Easily opt-in or opt-out of the system

### Non-Goals

- Running KRM functions
- Managing how applications use functions
<!-- - Actually installing or uninstalling containers -->

## Proposal

### User Stories

#### Story 1: Creating a new application

Alice has in mind some functions that she wants to use that are created by Group Alpha. She adds the repo their repo via `kaffine config add repo group-alpha.example.com`. Then she searches which functions are available in the repo.

```
$ kaffine search -l group-alpha
KIND      NAME          DESCRIPTION           LATEST  INSTALLED
function  alpha-foo-er  A function that foos  v2.0.0     ---
function  alpha-bar-er  A function that bars  v1.0.3     ---
function  alpha-baz-er  A function that bazs  v3.1.0     ---

$ kaffine install alpha-foo-er
Installed group-alpha/alpha-foo-er:v2.0.0

$ kaffine install alpha-bar-er
Installed group-alpha/alpha-baz-er:v3.1.0

$ kaffine list
KIND      NAME          DESCRIPTION           INSTALLED
function  alpha-foo-er  A function that foos  v2.0.0
function  alpha-baz-er  A function that bazs  v3.1.0

$ kaffine list > functions.yaml
$ cat functions.yaml
apiVersion: config.kubernetes.io/v1alpha1
kind: ResourceList
items:
- kind: KrmFunction
  repo: group-alpha.example.com
  name: alpha-foo-er
  description: "A function that foos"
  versions:
    - name: v2.0.0
      tags: ['latest-stable']
      runtime:
        container:
          image: images.example.com/functions/alpha-foo-er:v2.0.0
          sha256: a428de...
- kind: KrmFunction
  repo: group-alpha.example.com
  name: alpha-baz-er
  description: "A function that bazs"
  versions:
    - name: v3.1.0
      tags: ['latest-stable']
      runtime:
        container:
          image: images.example.com/functions/alpha-baz-er:v2.0.0
          sha256: a428de...
```

She can now pass `functions.yaml` to another application.

#### Story 2: Updating an existing application already using Kaffine

Bob wants to check if there are any updates to the functions he has installed.

```
$ kaffine list
KIND      NAME          DESCRIPTION           INSTALLED
function  alpha-foo-er  A function that foos  v1.0.0
function  alpha-bar-er  A function that bars  v0.0.3
function  alpha-baz-er  A function that bazs  v2.1.0

$ kaffine update --check
Checking for updates...
group-alpha/alpha-foo-er v1.0.0 -> v2.0.0
group-alpha/alpha-bar-er v0.0.3 -> v1.0.3
group-alpha/alpha-baz-er v2.1.0 -> v3.1.0
Checked for updates without installing.

$ kaffine update alpha-foo-er
group-alpha/alpha-foo-er v1.0.0 -> v2.0.0

$ kaffine update alpha-bar-er
group-alpha/alpha-bar-er v0.0.3 -> v1.0.3

$ kaffine list
KIND      NAME          DESCRIPTION           INSTALLED
function  alpha-foo-er  A function that foos  v2.0.0
function  alpha-bar-er  A function that bars  v1.0.3
function  alpha-baz-er  A function that bazs  v2.1.0
```

### Implementation Details/Notes/Constraints

#### Notes

A repo has 2 main parts: a URI and a name. The URI is the location of the repo. It can be a server somewhere or a path on the local filesystem. The name is a user-friendly name of the repo. If two repos conflict in naming, use the URI instead.

If you want to disambiguate between functions in different repos, you can prepend the name of the repo followed by a `/` character. Example: `repo-foo/function-bar`. If you do not provide a repo, it will search every repo you have for the specified function.

Additionally, if you want to install a specific version (such as `v1.1`, `latest-stable`, `nightly`, et cetera…) you can append a `:` character followed by the label. Example: `function-bar:nightly`. If you do not provide a version, it will default to `latest-stable`.

<!-- MAYBE ALSO A GO LIBRARY TO MAP FUNCTION NAMES TO LOCATIONS USING CONFIG -->
#### CLI Commands

**Search**

```
kaffine search
kaffine search <function-name>
```

Searches all repositories for the function with the specified name. If no name is provided, it’s functionally the same as: `kaffine search '*' -r`. If output is piped, `-o yaml` is automatically selected. Otherwise, `-o default` is the default.

_Options_

```
--regex, -r :: Treats <function-name> as a regular expression
--output, -o <format> :: Format to output, ex. yaml, default, etc...
--location, -l <location> :: The location to search.
    all :: (default) Searches all repos
    repo <repo-name/uri> :: Only searches in the specified repo
```

<!-- all :: (default) Searches installed functions and all repos
repos :: Only searches in all repos
repo <repo-name/uri> :: Only searches in the specified repo
local :: Only searches installed functions -->

<!-- NOTE: pipelines and collections are for the future, don’t worry about them right now -->

_Examples_
```
$ kaffine search foo
 
KIND        NAME                   DESCRIPTION                           LATEST  INSTALLED
function    foo-function           Generic foo function                  v2.0       ---
function    foo-function-binary    Generic foo function, using a binary  v1.0.1     ---
pipeline    make-fooable-pipeline  Makes resources fooable               v0.0.1     ---
collection  frobnicate-the-foos    Frobnicates the foos                  v3.0.1     ---
```

<!-- Using something like KEP-2906: Kustomize Function Catalog. Bare bones, probably would want to bring in line with KEP-2906 -->

```
$ kaffine search foo -o yaml
apiVersion: config.kubernetes.io/v1alpha1
kind: ResourceList
functionConfig:
   query: "foo"
   regex: false
items:
- kind: KrmFunction
  repo: repo1.example.com
  name: foo-function
  description: "Generic foo function"
  versions:
    - name: v2.0
      tags: ['latest-stable']
      runtime:
        container:
          image: docker.example.co/functions/foo-function:v2.0
          sha256: a428de...
    - name: v1.0
      ...
- kind: KrmFunction
  repo: repo2.example.com
  name: foo-function-binary
  description: "Generic foo function, using a binary"
  versions:
    - name: v1.0.1
      tags: ['latest-stable']
      runtime:
        binary:
          image: repo2.example.com/something/foo-function-binary-v1.tar.gz
          sha256: b539ef...
    - name: v2.0.0-nightly
      tags: ['nightly']
      ...
- ...
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

Tries to install the function with the given name. It searches the repos for the function with the given name. If it finds it, it installs it. If there are duplicates, it will return the duplicate functions.

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
kaffine config add repo <URI>
kaffine config remove repo <Name/URI>
kaffine config view
```

_Options_
```
--global, -g :: Updates the global config, as opposed to the local one
```

### Security, Risks, and Mitigations

**Carefully think through the security implications for this change**

What are the risks of this proposal and how do we mitigate. Think broadly. How
will this impact the broader OKD ecosystem? Does this work in a managed services
environment that has many tenants?

How will security be reviewed and by whom? How will UX be reviewed and by whom?

Consider including folks that also work outside your immediate sub-project.

## Design Details

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