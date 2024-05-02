---
title: neat-enhancement-idea
authors:
  - "jortel@redhat.com"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2024-04-25
last-updated: 2024-04-25
status: provisional|implementable|implemented|deferred|rejected|withdrawn|replaced
see-also:
replaces:
superseded-by:
---

# Support Multiple Languages


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Summary

### Problem ###

Currently: Each addon defines a single container.
Currently: analysis is performed by running a task that specifies the `tackle2-addon-analyzer` addon. The name of the addon is hard-coded in the UI.  This addon image incorporates support goland and Java providers. 

The analyzer images are being refactored and the analyzer image will no longer bundle language providers. Instead, each provider will be packaged as a separate image. To support this, addons will need to be defined multiple containers. A _main_ container for the addon and additional (sidecar) containers (one for each provider).

Future features may require runtime selection of an addon based on the _kind-of_ task and the _kind-of_ subject (application|platform|...) the task is executed on.
Examples:
- application technology discovery.
- platform discovery.
- platform analysis.

Questions/Assumptions:
- Do we need to support multi-language applications?  _Yes_.
- Can the filtering of targets/rulesets and selection of providers be driven by `language` tags?  _Yes_.

## Motivation

Applications may be implemented by languages other than Java.
Addons other than analysis will be soon be introduced.

### Goals

- Dynamically determine which addon to execute based on facts known about the application in inventory.
- Dynamically determine which language providers to execute based on facts known about the application in inventory.
- User can run analysis without knowing the languages.
- Targets (rules) may be filtered by language.
- Targets (rules) may be pre-filtered by the UI when language is known.
- Language defaults to Java (until automated discovery feature added).
- User initiated task execution (Example: analysis) must be prioritized.

### Non-Goals

## Proposal

#### Filtering/Selection ####
Targets will be filtered and Providers will be selected by `Language` tag(s).  This will likely depend on the Application Discovery feature.

#### Addon Extensions ####

 An Addon Extension is primarily an optional container definition along with criteria defined by a new CR. Providers will be defined to the hub as an Addon Extension.  The first supported selection criteria will be by application tag matching.

#### Task Pod ####
The task pod will be configured with a _main_ container for the addon and a _sidecar_ container for each of the providers.  All containers will mount an `EmptyDir` volume into /shared for sharing files such as credentials and source code trees.

#### Custom Providers ####
Users may define new providers for testing using the UI by added a new Extension CR that is mapped to a new Language tag.  Then, add/replace the language tag so that extension (provider) is selected. 

Testing with the API is simpler.  The user would add the Extension, and submit an analysis tag using the API that references the extension explicitly.

### User Stories

### Implementation Details/Notes/Constraints

Constraints:
- Must support running the analyzer (addon) and providers in either:
  - same container
  - separate containers
- Must support _remote_ providers running outside the cluster.
- Must ensure all containers are terminated when analysis has completed.
- Must provide the user with provider output/logs.
- Must provide credentials to through provider settings. Example: maven settings.
- Must support TCP port assignment which is injected to the provider settings.
- Must support provider specific settings.

### Security, Risks, and Mitigations

## Design Details (core)

Support optional addon _Extensions_. Each extension is a (sidecar) container and most likely a _service_.  For example: an RDBMS or LSP provider.  All containers within a task pod are injected with the same items as the _main_ addon contianer:
- Shared (EmptyDir) volume.
- Security token.
- Task (id).

Both addon and extension selection may be either:
- explicit - (specified on the task)
- determined by matched criteria.

#### Selection ####

As selector uses Label-Selector syntax and supports the following:
- **tag**:_category_=_tag_
- **platform**:(source|target)=_kind_ 

Examples:

Java
selector: tag:Language=Java

Java or XML
selector: tag:Language=Java || tag:Language=XML

No language tag.
selector: ! tag:Language=

Java and Linux
selector:  tag:Language=Java && tag:OperatingSystem=Linux

#### Addon & Extension CRs ####

New _Extension_ CR (Spec):
``` 
kind: Extension
spec:
  addon (string):
  container (core.v1.Container):
  selector (string):
  metadata (object):
```
Fields:
- `extension` - declares compatibility with addons.
- `container` - defines the extension container.
- `selector` - defines selector for inclusion in the addon task pod.
- `metadata` - defines unstructured information.

Changed _Addon_ CR (Spec):
```
- image:
- imagePullPolicy:
- resources:

+ kind (string):
+ container  (core.v1.Container):
+ selector:
+ metadata (object):
```

Fields:
- `kind` - declares compatibility with a _kind_ of task.
- `container` - defines the addon container.
- `selector` - defines addon selector for task (kind).
- `metadata` - defines unstructured information.
- `schema` - _FUTURE_ defines the Task.Data schema.

For symmetry and completeness, the `metadata` field is added and the separate addon fields for the container are replaced with the entire k8s _Container_.   This is mainly a general improvement while we're changing the CR.

Example:
```
---
kind: Addon
apiVersion: tackle.konveyor.io/v1alpha1
metadata:
  namespace: konveyor-tackle
  name: analyzer
spec:
  container:
    name: addon
    imagePullPolicy: Always
    image: quay.io/jortel/tackle2-addon-analyzer:provider

---
kind: Extension
apiVersion: tackle.konveyor.io/v1alpha1
metadata:
  namespace: konveyor-tackle
  name: java
spec:
  addon: analyzer
  selector: tag:Language=Java || !tag:Language
  container:
    name: java
    imagePullPolicy: Always
    image: quay.io/konveyor/java-external-provider
    args:
    - --port
    - $(PORT)
    env:
    - name: PORT
      value: ${seq:8000}
    resources:
      limits:
        cpu: 1
        memory: 3Gi
      requests:
        cpu: 1
        memory: 3Gi
  metadata:
    resources:
    - selector: identity:kind=maven
      fields:
      - name: settings
        path: /shared/creds/maven/settings.xml
        key: maven.settings.path
    provider:
      name: java
      address: localhost:$(PORT)
      initConfig:
      - providerSpecificConfig:
          mavenSettingsFile: $(maven.settings.path)

```

#### API ####

Task API resource changes:
```
+ Extensions []string `json: "extensions,omitempty"`.
```
Example:
```
addon: analyzer
extensions:
- java
```

#### Port Injection ####

The task manager will provide a set of injector (macros).  Format: ${<_macro_}.  Injector macros may be specified in the container.Spec:
- command
- args
- env

Automatic port assignment will be supported using a _sequence_ (generator) macro. 
Format: ${seq:_pool_}.
Example:
```
container:
  name: Test
  env:
  - name: PORT
    value: ${seq:8000}
```

#### Kinds of Tasks ####

A task (definition) defines a _kind-of_ task which performs a specific action on a specific subject.

To prevent hard-coding tag names in the UI (anywhere), we may consider data-driven approaches.
Introducing _named_ task (definitions) which represent a known _kind-of_ task.  Each (definition) defines the input schema and how an addon (that implements the task) may be selected. Addons (optionally) declare that they implement a kind-of task.

New _Task_ CR:
- name
- dependencies:
- data (object)

Fields:
- `name` the kind-of task.
- `dependencies` list of task names used to define execution dependency.
- `data` - data object passed to the addon.
- `schema` - _FUTURE_ defines the Task.Data schema.

```
kind=Task
name: discovery
addon:
 - name: discovery
```
```
kind=Task
name: analysis
dependencies:
- discovery
```
```
kind=Task
name: platform-analysis
```
```
kind=Task
name: platform-discovery
```

Add `Task.Kind` which can be applied by the exiting task endpoints.  Either kind OR addon and extensions may be specified but not both.

```
+ Kind []string `json: "kind,omitempty"`.
```
Example:
```
kind: analyzer
```

Add a new read-only endpoints.
- /tasks/:id/attached - _download (tgz) attached files._
- /k8s/addons - _moved from /addons_
- /k8s/addons/:name - _moved from /addons/:name_, includes extensions.
- /k8s/extensions - _new extensions colleciton_
- /k8s/extensions/:name - _new extension by name_

#### Task Priority ####

Tasks will be run in order of:
- priority
- id

Task dependencies will be escalated as needed to prevent priority inversion. When (dep) tasks are already scheduled (Pending), the pod will be recreated referencing the updated priority class.

When the nodes are saturated and task pods are phase=Pending, the task manager may reschedule (delete) task  with lower priority in an effort for higher priority pods to be run by the node scheduler.  PriorityClass preemption should handle this.

Addon (adaptor)
---

#### Environment (variable) Injection ####

To injecting container environment variables into the Extension.metadata. 
The format is exactly like k8s environment variable: $(_variable_).  

Example:
```
container:
  name: Test
  env:
  - name: PORT
  - value: 8000
metadata:
  address: localhost:$(PORT)
    ...
```

---

Analyzer Addon (adaptor) 
---

delegated to: https://github.com/konveyor/tackle2-addon-analyzer/issues/83

#### Resource Injector ####

Aggregate the initConfig fragments from each Extension.metadata.

Inject information into the initConfig.
- credentials.

Considering: To support this, a set of injectors will be provided to get, store hub REST resource (fields). Fetched resources are assigned to variables that may be used in the metadata.
Resource:
- selector (format: \<kind>:\<key>=\<value>)
- fields[]:
  - name: - name.
  - path: - (optional) value stored at the specified path.
  - key:  - value stored in key referenced as $(_key_). Assigned to path, when specified.

Example:
```
metadata:
  resources:
  - kind: identity:kind=maven
    fields:
    - name: settings
      path: /shared/creds/maven/settings.xml
       key: settings.path
  - kind: identity=other
    fields:
    - name: user
       key: auth.user
    - name: password
       key: auth.password
  provider:
    initConfig:
     mavenSettingsPath: $(settings.path)
     user: $(auth.user)
     password: $(auth.password)
```

### Test Plan

### Upgrade / Downgrade Strategy

## Implementation History

## Alternatives

## Infrastructure Needed [optional]

