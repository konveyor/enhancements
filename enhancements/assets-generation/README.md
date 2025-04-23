---
title: assets-generation-and-platform-awareness
authors:
  - "rromannissen"
reviewers:
  - "@dymurray"
  - "@jortel"
  - "@eemcmullan"
  - "@JonahSussman"
  - "@jwmatthews"
approvers:
  - "@dymurray"
  - "@jortel"
  - "@eemcmullan"
  - "@JonahSussman"
  - "@jwmatthews"
creation-date: 2024-11-20
last-updated: 2025-04-23
status: provisional
see-also:
  -    
replaces:
  -
superseded-by:
  -
---

# Assets generation and Platform Awareness


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

- Should there be a dynamic way of registering Platform Types, Discovery Providers and Generator Types? Should that be managed by CRs or could there be an additional mechanism? That would imply adding some dynamic behavior on the UI to render the different field associated with each of them.
- How could we handle the same file being rendered by two different _Generators_ (charts)? Is there a way to calculate the intersection of two different Helm charts?

## Summary

Since its first release, the insights that Konveyor could gather from a given application were either coming from the source code from the application itself (analysis), or from information provided by the different stakeholders involved in the management of the application lifecycle (assessment). This enhancement proposes a third way of surfacing insights about an application by gathering both runtime and deployment configuration from the very platform in which the application is running (discovery), and storing that configuration in a canonical model that can be leveraged by different Konveyor modules or addons.

Aside from that, the support that Konveyor provided for the migration process stopped when the application source code was modified for the target platform, leaving the application itself ready to be deployed but without the required assets to get it actually deployed in the target platform. For example, for an application to be deployed in Kubernetes, it is not only necessary to adapt the application source code to run in containers, but it is also necessary to have deployment manifests that define how that application can be deployed in a cluster, a Containerfile to build the image and potentially some runtime configuration files. This enhancement proposes a way to automate the generation of those assets by leveraging the configuration and insights gathered by Konveyor.


## Motivation

This enhancement aims at enabling Konveyor to eventually tackle the following use cases:

- Fast tracking the migration of containerized applications by automating the translation of deployment assets from one platform to the other. For example, by automating all the configuration gathering from an application deployed in Cloud Foundry and leveraging that information to generate custom tailored deployment manifests for Kubernetes.
- Generation of deployment assets for applications that haven't been containerized.
- Translation of application configuration between application servers (for example Weblogic to EAP).


### Goals

- Platform Awareness:
  - Enable Konveyor to retrieve information about applications directly from the platform in which they are running:
    - Deployment configuration.
    - Runtime configuration.
  - Flexible enough to obtain information from multiple platform types:
    - Container platforms.
    - Application servers.
    - Hypervisors and VMs.
    - Others...
- Assets generation:
  - Flexible enough to generate all assets required to deploy an application on k8s (and potentially other platforms in the future)
  - Provide opinionated best practices out of the box.
  - Allow organizations to create their own corporate assets easily:
    - Use templating as much as possible.
    - Build on industry standards.
    - Avoid requiring new users to learn new programming languages or proprietary APIs.

### Non-Goals

- Define a transformation logic when migrating between platforms or runtimes. The way the pieces in this enhancement are meant to work is the following:
  - Platform awareness is able to retrieve information about how an application is deployed in a certain platform, potentially including runtime configuration as well.
  - That information is translated into a well known canonical configuration model.
  - Assets generation allows user to use a standard templating engine to create assets (deployment manifests, configuration files, etc.) that suit their needs, leveraging the canonical configuration that gets exposed to the templates in a similar way to Ansible facts.
  - **Configuration discovery is orchestrated by Konveyor, but the actual transformation logic is modeled in templates like Helm charts, and managed by the template authors, who are assumed to be knowledgeable in the transformations required to meet their needs. Konveyor exposes the discovered configuration in a well known format so template authors can leverage it to establish equivalences between the source and target platforms.**
- Defining the logic of any Discovery Provider, as each one of them should have their own dedicated enhancements to specify their behavior in relation with the platform each one of them tackles. The aim of this enhancement is to establish the overall framework and component infrastructure to enable discovery and assets generation.

## Proposal

### Personas / Actors

#### Administrator

The administrator for the tool that has access to some application-wide configuration parameters that other users can consume but not change or browse.

#### Architect

A technical lead for the migration project that can create and modify applications and information related to it.


### User Stories

#### Platform Awareness

##### PA001

*As an Architect I want to be able to discover and retrieve configuration from applications deployed in a certain platform*

##### PA002

*As an Administrator I want to be able to manage different platform instances*

##### PA003

*As an Architect I want to be able to associate Source platforms from existing platform instances to applications*

##### PA004

*As an Architect I want to be able to retrieve configuration for existing applications that have an associated source platform instance at bulk or on a per application basis*

##### PA005

*As an Architect I want to be able to discover applications deployed in an existing platform instance and use that to populate the application inventory, including the configuration for each individual application*

#### Assets generation

##### AG001

*As an Architect I want to be able to generate assets (configuration files, deployment manifests or any file that might be relevant) to deploy an application in a given target platform.*

##### AG002

*As an Architect I want to be able to author templates to render the required assets using well known templating engines*

##### AG003

*As an Architect I want to be able to author templates using Helm Charts*

##### AG004

*As an Architect I want to be able to manage repositories that contain templates to be used to render assets*

##### AG005

*As an Architect I want to be able to assign templates to target platforms*

##### AG006

*As an Architect I want to be able to associate target platforms with archetypes*

##### AG007

*As an Architect I want to be able to override variables contained in the application configuration retrieved during discovery*

##### AG008

*As an Architect I want to be able to store the generated assets in a repository, that could be the one from the application itself or a different configuration repository*


### Design Details

#### Platform Awareness

##### Platform Instance

First class entity to model a platform instance in Konveyor.

- Managed in the administration perspective
- Potential fields:
  - Name
  - Platform Type (Kubernetes, Cloud Foundry, EAP, WebSphere…)
  - URL
  - Credentials (From the credentials vault in Konveyor)
  - Extra fields depending on the type (TBD)

##### Changes in the Application entity

There will be platform related fields in the Application entity. These fields should be considered optional, as applications can still be managed manually or via the CSV import without the need for awareness of the source platform.

A _Source Platform_ section (similar to the Source Code and Binary sections) should be included in the _Application Profile_, including the following fields:

- _Platform Type_ (Kubernetes, Cloud Foundry, EAP, WebSphere…).
- _Platform Instance_ (From the list of available Platform Instance entities in the system).
- _Location_: A field section expressing the coordinates of the application inside the associated Platform Instance. Fields on this section will depend on the selected Platform Type, as they will be platform dependent. For example, depending on the platform they could be:
    - K8s: Namespace, Service ID…
    - EAP: Profile, Server Group…

A read-only _Configuration_ dictionary should also be browsable in the _Application Profile_. For more about _Configuration_ see the [Canonical Configuration model section](#canonical-configuration-model) section.

_Target Platforms_ will be surfaced in the _Application Profile_ as read only data (can't be manually and individually associated to a single application) and inherited from the archetype.

##### Canonical Configuration model


- YAML dictionary containing platform and runtime configuration for a given application.
- Sections are populated by [discovery](#discovery) and analysis.
- Documented way of storing configuration:
  - Keys are documented and have a precise meaning.
  - Similar to Ansible facts, but surfacing different concerns related to the application runtime and platform configuration.
- RBAC protected.
- Injected in tasks by the addon.

##### Discovery

Discovery should be considered the act of retrieving application information and configuration from a platform. There will be two differentiated scenarios:

- When applications already exist in the inventory and contain _Source Platform_ information (_Platform Type_, _Platform Instance_ and _Location_), users should be able to run discovery and populate the _Configuration_ dictionary. This discovery could be run on a per application basis or at bulk if all selected applications contain _Source Platform_ information.
- On application import, targeting an existing platform with some criteria to look for applications and associated configuration. Criteria would depend on the source platform and their associated [Discovery Providers](#discovery-providers). Some examples depending on potential source platforms could be:
  - Kubernetes:
    - Namespace patterns (include/exclude)
    - Match criteria (service, deployment, route…)
  - EAP:
    - Server group

##### Discovery Providers

Abstraction layer responsible of collecting configuration around an application on a given platform:
- Live connection via API or similar methods.
- Through the filesystem accessing the path in which the platform is installed (suitable for Application Servers and Servlet containers). This would likely be modeled as an agent deployed on the platform host itself.

Configuration discovery could happen in different stages during the lifecycle of an application to avoid storing sensitive data:
- *Initial discovery*:
  - Configuration dictionary gets populated with non sensitive data. Sensitive data gets redacted or defaults to dummy values.
- *Template instantiation*:
  - A second discovery retrieval happens to obtain the sensitive data and inject it in the instantiated templates (the actual generated assets) without storing the data in the Configuration dictionary.

Discovery providers are custom tailored for the particularities of each platform and should be able to differentiate regular configuration from sensitive data.

#### Assets Generation

##### Generators

First class entity in Konveyor to wrap templates. Fields include:
- _Name_
- _Icon_
- _Generator Type_: Will only include Helm for the moment, but in the future we could include other types like Ansible or other templating engines. Generator type will determine the image that gets used to handle the generator task.
- _Description_
- Repository containing the template files:
  - _Repository type_ (Git/SVN)
  - _URL_
  - _Root Path_
  - _Branch_
  - _Credentials_
- _Variables_: List of prefixed variables that will be injected on template instantiation (The `helm template` command for example). Variables that match name with the ones coming from the Configuration dictionary will override their value.
- _Parameters_: List of parameters the user will be asked for when generating assets with this template. Similar to [Surveys](https://ansible.readthedocs.io/projects/awx/en/latest/userguide/job_templates.html#surveys) in Ansible AWX.

##### Archetypes, Target Platforms and Generators

Multiple _Generators_ can be associated with an _Archetype_ through [_Target Platforms_](https://github.com/konveyor/enhancements/issues/186):
- Foster reusability.
- The generated assets for an archetype would be the product of the union of the instantiation of all templates from all _Generators_ associated with that _Archetype_:
  - Generators will have an order of precedence when associated with _Target Platforms_. If the same file is output by two generators, the one that was produced by the _Generator_ with the top level of precedence will be included in the resulting fileset.

![Archetypes, Target Platforms and Generators](images/archetypes-targetplatforms-generators.png?raw=true "Archetypes, Target Platforms and Generators")

For example, using the previous diagram, let's assume the _EAP on OpenShift Generator_ had top precedence and the _OpenShift Generator_ was second for the _OpenShift_ _Target Platform_. If both generators output the same file (for example Deployment.yml), the one produced by the _EAP on OpenShift Generator_ would be the one added to the resulting generated assets fileset.

##### Templating engine

In a first iteration, leverage the [Helm templating engine](https://helm.sh/docs/chart_template_guide/functions_and_pipelines/), as it is the lingua franca for Kubernetes related resource definition, although the solution should be open to other technologies in the future, such as Ansible for more complex assets generation.

##### Template instantiation

Template instantiation should be considered the act of injecting values in a template to render the target assets (deployment descriptors, configuration files...). For the Helm use case in this first iteration, the process could be as follows:

- The hub generates a values.yaml file based on the intersection of the _Configuration_ dictionary for the target application, the fixed _Variables_ set in the Generator and the _Parameters_ the user might have provided when requesting the generation, in inverse order of preference (_Parameters_ have top preference over the others, then _Variables_ and finally the _Configuration_ dictionary). That file should also include values inferred from other information stored in the application profile such as tags.
- The values.yaml file is injected by the addon in a _Generator_ task pod that will execute the `helm template` command to render the assets.
- The generated assets are then placed in a branch of the repository associated with the application.

![Template Instantiation](images/template-instantiation.png?raw=true "Template Instantiation")

From a UI/UX perspective, when requesting assets generation for a given application, users would be prompted with the following:
- Values for the _Parameters_ configured in the associated _Generator_(s)
- Target repository for the generated assets. Will default to the application repository and the `generated_assets` branch, but could be used to store configuration in a different configuration repository if that the pattern the organization uses:
  - _URL_
  - _Root Path_
  - _Branch_
  - _Credentials_
- Option to skip template instantiation and simply copy the charts to the target repository and inject the configuration as a values file. This is helpful when template instantiation is handled by an external orchestrator like a CI/CD pipeline.

##### Repository Augmentation

- Generated assets could be stored in a branch from the target application repository, or if needed, on a separate configuration repository if the application has adopted a GitOps approach to configuration management.
- Allow architects to seed repositories for migrators to start their work with everything they need to deploy the applications they are working on right away → Ease the change, test, repeat cycle.
- Aligned with the Seed work step from the Do work stage in the [Konveyor Unified Experience enhancement](https://github.com/konveyor/enhancements/tree/master/enhancements/unified_experience#step-4-do-work).


### Functional Specification

TBD

### Implementation Details/Notes/Constraints

#### Sensitive Data Management

The _Discovery Provider_ determines which fields are sensitive and replaces the sensitive data with a _ref_(placeholder). Only the _Discovery Provider_ can identify sensitive data.
This would be an option on the discovery process (default: false).


##### Input

Sample CF manifest:

```yaml
...
name: Elmer
username: rabbit
password: slayer24
...
```

##### Discovery Provider Output

manifest.yaml
```yaml
...
name: Elmer
username: $(secret-0001)
password: $(secret-0002)
...
```

secret.yaml
```yaml
...
secret-0001: rabbit
secret-0002: slayer24
...
```
The addon would store the application manifest (and secret in the hub). The secret would be stored encrypted.

The hub API could render the application manifest with the secret references substituted depending on user permissions.

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

## Infrastructure Needed

TBD
