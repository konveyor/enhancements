---
title: neat-enhancement-idea
authors:
  - "@JonahSussman"
  - "@savitharaghunathan"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2025-01-20
last-updated: 2025-01-20
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

## Summary

This proposal introduces enhancements to the existing MTA CLI tool (Kantra) to support the discovery and transformation of resources from Cloud Foundry (PCF) to Kubernetes. The goal is to create a modular CLI that outputs a canonical representation of resources and generates deployment artifacts (e.g., Helm charts). The enhancement will provide a foundational capability for platform migrations, enabling a tech-preview release in version 7.3 of Kantra and integration into downstream workflows.

## Motivation

Organizations migrating workloads from Cloud Foundry to Kubernetes require reliable tools to simplify resource discovery and deployment transformation. Existing tools like Move2Kube (M2K) provide limited flexibility and lack a modular approach, resulting in suboptimal user experiences. This enhancement addresses these gaps by extending the Kantra CLI with commands for discovery and asset generation while supporting a canonical configuration format for future expansion.

We propose modifying the existing Kantra CLI instead of developing an entirely new CLI to ensure a smoother adoption process for users. Leveraging Kantra's established user base allows for quicker buy-in and ensures users have an enhanced tool in their hands sooner. Additionally, this approach simplifies the overall workflow for existing users already familiar with the tool. To achieve modularity and maintainability, we will create a standalone library of functionality to which the CLI will serve as a frontend, allowing for future integration with other tools and workflows.

### Goals

- Create a standalone library for integration with other tools and workflows.
- Extend the MTA CLI (Kantra) to support:
  - `discover` command: Output a YAML/JSON representation of PCF resources.
  - `generate` command: Produce Helm charts for deployment to Kubernetes.
- Utilize the canonical configuration enhancement to serve as a consistent intermediary for platform transformations.

### Non-Goals

- Handling of sensitive resource data, such as secrets, is explicitly out of  scope for this CLI. Instead, we will rely on MTA's secure encryption stores, which users already use with MTA.
- Automatic detection of application archetypes.
- Full compatibility with the Konveyor add-on process.
- Immediate support for platforms beyond PCF and Kubernetes.

## Proposal

### User Stories

#### Story 1: Application Discovery

As a developer, I want the CLI to analyze my Cloud Foundry application and output a structured representation so I can understand the configuration and resources involved.

#### Story 2: Deployment Artifact Generation

As a platform engineer, I want the CLI to generate manifests for my application, making it easy to deploy on Kubernetes with minimal manual intervention.

### Implementation Details

- **Command Design:**
  - `discover`: This command is responsible for analyzing an application from a specified platform and generating a canonical representation in YAML/JSON format. Details:
    - **Flags and Options:**
      - `--platform=<platform>` (required): Specifies the platform to discover. Supported value for MVP: pcf.
      - `--output=<file>` (optional): Writes the output to a specified file. Defaults to standard output.
      - `--verbose` (optional): Provides additional details about the discovery process in the logs.
    - **Behavior:**
      - Scans the provided application input (e.g., Cloud Foundry manifest files) and extracts relevant configuration details.
      - Outputs a structured canonical representation containing metadata (e.g., application name, instances, routes), runtime information (e.g., memory, disk), and deployment settings.
  - `generate`: This command takes a canonical representation and a Helm template to produce deployment-ready Helm charts. Details:
    - **Flags and Options:**
      - `--input=<file>` (required): Specifies the canonical configuration file to use as input.
      - `--template=<path>` (required): Path to the Helm template to use for chart generation.
      - `--output-dir=<directory>` (optional): Directory to save the generated Helm chart. Defaults to the current directory.
      - `--type=<type>` (optional): Specifies the type of generator. MVP supports helm.
    - **Behavior:**
      - Reads the canonical representation file to extract configuration details.
      - Applies the provided Helm template to generate deployment artifacts for Kubernetes.
      - Validates the generated artifacts for completeness and correctness.

- **Library Architecture:**
  - Implement the core functionality as a standalone library in Go for reuse in Kantra and other projects.
  - Modularize the functionality to support future enhancements.
**Integration Path:**
  - Release as part of Kantra 7.3 in tech preview.

### Security, Risks, and Mitigations

- Conduct code reviews with security specialists to mitigate risks.
- Validate CLI outputs to prevent injection vulnerabilities in generated artifacts.

## Design Details

### Test Plan

1. Unit tests for the discover and generate commands.
2. Integration tests for end-to-end workflows (PCF → canonical → Helm charts).
3. Manual validation with sample PCF applications.

### Upgrade/Downgrade Strategy

Add versioning to canonical configuration files to ensure backward compatibility.

Provide migration scripts for users upgrading from earlier Kantra versions.

## Implementation History

- January 2025: Proposal created.
<!-- - February 2025: Initial prototype of discover and generate commands. -->
<!-- - April 2025: Planned tech-preview release in Kantra 7.3. -->

## Drawbacks

- Increased complexity in CLI development and maintenance.
- Requires coordination across multiple teams (CLI, Hub, UI).

## Alternatives

- Develop a completely new tool instead of enhancing Kantra.

# Infrastructure Needed

N/A
