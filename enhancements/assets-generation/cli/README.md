---
title: konveyor-asset-generation-and-platform-awareness-cli-enhancement
authors:
  - "@JonahSussman"
  - "@savitharaghunathan"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2025-01-20
last-updated: 2025-01-22
status: provisional
see-also:
  - "https://github.com/konveyor/enhancements/pull/210"
---


# Konveyor Asset generation and Platform Awareness - CLI Enhancement

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created


## Open Questions
1. Do we need an interactive mode where users can preview and modify
   configurations during discovery or generation?
2. Should the CLI provide validation against Kubernetes best practices
   (e.g., resource limits, liveness probes) during artifact generation?
   Note: yaml provider in the analyzer that has this capability.
   We could leverage that if needed.

## Summary

This proposal introduces enhancements to the existing Kantra tool to support the
discovery and transformation of resources from a source platform to a target platform
(eg: Cloudfoundry -> Kubernetes). The goal is to create a modular CLI that outputs a canonical
representation of resources and generates deployment assets (e.g., Helm
charts). The enhancement will provide a foundational capability for platform
migrations, enabling a tech-preview release in an upcoming version of Kantra.

## Motivation

Organizations migrating workloads from one platform to another (e.g. Cloud Foundry to Kubernetes) 
require reliable tools to simplify resource discovery and deployment transformation.
Existing tools like Move2Kube (M2K) provide limited flexibility and lack a
modular approach, resulting in suboptimal user experiences. This enhancement
addresses these gaps by extending the Kantra CLI with commands for discovery and
asset generation while supporting a canonical configuration format for future
expansion.

We propose modifying the existing Kantra CLI instead of developing an entirely
new CLI to ensure a smoother adoption process for users. Leveraging Kantra's
established user base allows for quicker buy-in and ensures users have an
enhanced tool in their hands sooner. Additionally, this approach simplifies the
overall workflow for existing users already familiar with the tool. To achieve
modularity and maintainability, we will create a standalone library of
functionality to which the CLI will serve as a frontend, allowing for future
integration with other tools and workflows.

### Goals

- Create a standalone library for integration with other tools and workflows.
- Extend Kantra to support:
  - `discover` command: Output a YAML representation of source platform resources.
  - `generate` command: Produce assets for deployment to the target plaform. For the mvp,
     we will generate a values.yaml file based on the source platform config,
     and combine it with the user's templates to generate a helm chart
- Utilize the [canonical configuration enhancement](link: TODO) to serve as a consistent
  intermediary for platform transformations.
- Enhance logging and error reporting for better user experience and debugging.
- Provide detailed documentation and examples for both commands to facilitate adoption.

### Non-Goals

- Handling of sensitive resource data, such as secrets, is explicitly out of
  scope for this CLI. Instead, we will rely on Konveyor's secure encryption
  stores, which users already use with Konveyor.
- Automatic detection of application archetypes.
- Full compatibility with the Konveyor add-on process.
- Immediate support for platforms beyond Cloud Foundry and Kubernetes.
- Actual agent implementation. we will have stubs for the `agent` 
  subcommand for initial mvp.

## Proposal

### User Stories

#### Story 1: Application Discovery

As a developer, I want the CLI to analyze my Cloud Foundry application and
output a structured representation so I can understand the configuration and
resources involved.

#### Story 2: Deployment Artifact Generation

As a platform engineer, I want the CLI to generate manifests for my application,
making it easy to deploy on Kubernetes with minimal manual intervention.

### Implementation Details

- **Command Design:**
  - `discover`: This command is responsible for analyzing an application from a
    specified platform and generating a canonical representation in YAML/JSON
    format. Details:
    - **Flags and Options:**
      - `--platform=<platform>` (required): Specifies the platform to discover.
        Supported value for MVP: cf.
      - `--input=<name>` (optional): Specifies the application to be analyzed.
      - `--use-agent=<agent_name>` (optional): Enables agent-based discovery for extracting data.
      - `--use-live-connection` (optional): Uses live platform connections for real-time discovery.
      - `--output=<file>` (optional): Writes the output to a specified file.
        Defaults to standard output.
      - `--list-platforms` (optional): Lists the available discovery providers.
        For mvp, it defaults to `cf`
      - `--log-level` (optional): Provides additional details about the discovery
        process in the logs.
    - **Behavior:**
      - Scans the provided application input (e.g., Cloud Foundry manifest
        files) and extracts relevant configuration details.
      - Outputs a structured canonical representation containing metadata (e.g.,
        application name, instances, routes), runtime information (e.g., route, etc ),
        and deployment settings.
      - **Error Handling:**
        - Detect and report missing or malformed input files with detailed error messages.
        - Provide retry options for transient errors during discovery.
      - **Discovery Methods:**
        - Via live connections using APIs or other supported interfaces.
        - Through filesystem access to the platform's installation paths (e.g., for Application Servers
          or Servlet Containers), implemented as an agent on the host.
          
  - `generate`: This command takes a canonical representation and a Helm
    template to produce deployment-ready Helm charts. Details:
    - **Flags and Options:**
      - `--input=<file>` (required): Specifies the canonical configuration file
        to use as input.
      - `--template=<path>` (required): Path to the Helm template to use for
        chart generation.
      - `--output-dir=<directory>` (optional): Directory to save the generated
        Helm chart. Defaults to stdio.
      - `--type=<type>` (optional): Specifies the type of generator. MVP
        supports helm. Defaults to `Helm`
    - **Behavior:**
      - Reads the canonical representation file to extract configuration
        details.
      - Applies the provided Helm template to the canonical form and the user provided templates,
        and generates Kubernetes manifests.
      - Validates the generated artifacts for completeness and correctness.
    - **Error Handling:**
      - Log errors when applying Helm templates.
      - Support a dry-run mode to identify potential issues without generating artifacts.

- **Library Architecture:**
  - Implement the core functionality as a standalone library in Go for reuse in
    Kantra and other projects.
  - Modularize the functionality to support future enhancements. 
- **Integration Path:**
  - Release as part of upcoming Kantra/Konveyor in tech preview.

### Security, Risks, and Mitigations

- Conduct code reviews with security specialists to mitigate risks.
- Validate CLI outputs to prevent injection vulnerabilities in generated
  artifacts.
- Validate that access to CF platform credentials and RBAC access rights don't
  violate pre-existing policies.

#### **Risks**
1. Modularity increases the maintenance burden.
   - **Mitigation:** Maintain clear documentation and well-defined APIs for each module.

## Design Details

### Test Plan

1. Unit tests for the discover and generate commands.
2. Integration tests for end-to-end workflows (eg: Cloud Foundry -> discovery manifest -> Helm chart).
3. Test each of the components separately: Discovery and Generate.
4. Ensure sensitive data is not exposed.


### Upgrade/Downgrade Strategy

1. Add versioning to canonical configuration files to ensure backward
compatibility.
2. Provide migration scripts for users upgrading from earlier Kantra versions.

### Documentation

1. User-facing documentation with examples for both discover and generate commands.
2. Developer guide for extending the CLI with new platforms or output formats.
3. Troubleshooting guide for common errors and issues.

## Implementation History

- January 2025: Proposal created.
<!-- - February 2025: Initial prototype of discover and generate commands. -->
<!-- - April 2025: Planned tech-preview release in Kantra 7.3. -->

## Drawbacks

- Increased complexity in CLI development and maintenance.
- Requires coordination across multiple teams (CLI, Hub, UI).

## Alternatives

1. **Develop a New Tool:**
   - Pros: Tailored design for the intended functionality.
   - Cons: Longer time-to-market and additional user onboarding efforts.

2. **Enhance Move2Kube:**
   - Pros: Builds on an existing migration tool.
   - Cons: Limited flexibility and potential user confusion due to overlapping scopes.

# Infrastructure Needed

- **CI/CD Pipelines:** For testing and releasing the CLI.
- **Sample Cloud Foundry Applications:** To validate discovery and generation.
