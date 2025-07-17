---
title: kai-solution-server
authors:
- "@fabianvf"
reviewers:
- @savitharaghunathan
- @jwmatthews
- @shawn-hurley
approvers:
- @savitharaghunathan
- @jwmatthews
- @shawn-hurley
creation-date: 2025-04-29
last-updated: 2025-07-07
status: provisional
see-also: []
replaces: []
superseded-by: []
---

# Kai Solution Server

## Summary

The Solution Server delivers two primary benefits to users of Kai and MCP-compatible tools:

- **Contextual Hints:** It surfaces examples of past migration solutions—including successful user modifications and accepted fixes—enabling Kai to offer actionable hints for difficult or previously unsolved migration problems.
- **Migration Success Metrics:** It exposes detailed success metrics for each migration rule, derived from real-world usage data. These metrics can be used by IDEs or automation tools to present users with a “confidence level” or likelihood of Kai successfully migrating a given code segment. The server itself does not compute confidence levels, but provides the necessary data for consuming tools to do so.

This approach leverages users’ accepted and rejected migration outcomes to continuously improve code generation quality, while providing transparency about the reliability of automated migrations.

## Motivation

Currently, Kai helps developers individually with migration issues. Leveraging organizational
knowledge from past migration attempts can significantly enhance migration outcomes and reduce
manual intervention. The Solution Server aims to achieve this by providing clear success metrics,
historical insights, and identifying migration tasks that consistently require manual resolution.

### Goals

- Provide clear visibility into the effectiveness of migration rules.
- Improve the success rates of automated migrations using historical data.
- Identify recurring migration challenges to guide improvements.

### Non-Goals (for Initial Release)

- Implementation of fine-grained RBAC or advanced multi-tenancy.
- Obfuscation or tokenization of source code prior to storage or hint generation.
- Centralized, cross-tenant aggregation of migration data (beyond org-wide hubs).
- External or third-party integration of solution data.
- Design or implementation details beyond what is required to define MVP behavior.

## Proposal

The Kai Solution Server will implement the following core functions:

### Data Collection

The Solution Server will collect detailed migration-related data from IDEs, including:
- Source code before migration
- Modified code after migration
- Metadata about migration tasks (languages, frameworks, rules triggered)
- Outcomes of migration attempts (accepted, rejected, modified)
- Specific user modifications when automated solutions are adjusted

### Storage and Organization

- Collected data will be structured to support easy retrieval, analysis, and reuse.
- Data organization will align closely with migration rules and associated metadata.

### Analysis and Insights

- Identify patterns in successful and unsuccessful migration solutions.
- Extract and summarize insights ("hints") that can directly inform future automated migration attempts.
- Provide raw metrics for IDE plugins to compute and display confidence levels as they see fit.

### Enhanced Context for Automation Tools

- Provide historical examples and success metrics directly to the IDE plugin.
- Enable any other MCP-compatible tools to leverage past successful solutions to improve current migrations.

### Issue Identification and Expert Review

- Automatically flag migration tasks or rules that consistently fail or require extensive manual intervention.
- Highlight flagged issues for expert review and potential process improvements.

## Scope of Initial Implementation

- Store and index user-accepted and rejected migration solutions and relevant metadata, scoped per Konveyor Hub instance.
- Make historical solution data and rule-level success metrics available via MCP API to IDE plugins and compatible tools.
- All data and metrics are available to any authenticated user of the associated Hub instance; there is no additional RBAC or fine-grained access control in the initial release.
- Only per-Hub data mining and insights are supported. Cross-Hub or org-wide aggregation is out of scope unless the deployment is already org-wide.
- No obfuscation, tokenization, or privacy-enhancing transformations are applied to stored source code or hints at this time.

Planned enhancements (RBAC, advanced data sharing, code anonymization/tokenization, and global mining across Hubs) are explicitly deferred to future work and are not addressed by this proposal.

## Infrastructure & Deployment

- The Solution Server will be deployed as a Kubernetes Deployment in the same namespace as the Konveyor Hub instance it supports.
- It exposes a service accessible only within the namespace. All access is routed through the Konveyor Hub’s proxy, which is responsible for authenticating and authorizing requests.
- A dedicated PostgreSQL database, managed by the konveyor operator, is required for storage. This is deployed alongside the Solution Server within the same namespace.
- No direct external access is permitted; all traffic must go through the Hub proxy.
- Each Konveyor Hub instance is expected to have its own isolated Solution Server and database. This ensures that migration data and metrics are tenant-scoped and are not shared between Hub instances, unless specifically designed for an org-wide Hub deployment.

## Security and Risks

- **Tenant Isolation:** By default, each Solution Server is isolated per Konveyor Hub instance and only accessible to users with credentials for that Hub. There is no data sharing across Hub instances.
- **Authentication and Authorization:** Access to the Solution Server is exclusively mediated by the Konveyor Hub proxy, which manages authentication. At present, there is no fine-grained RBAC—any user who can access the Hub can also query the Solution Server.
- **Data Protection:** All sensitive data—including source code, migration results, and user modifications—is encrypted at rest in the PostgreSQL database, and all network traffic between components is encrypted (TLS).
- **No Obfuscation/Tokenization (Initial Release):** For the initial release, all collected code and migration data is stored in cleartext for maximum utility. There is no plan to obfuscate or tokenize source data at this stage. The risks and potential future work on code anonymization or tokenization are acknowledged, but not in scope for this version.
- **Org-Wide Hubs:** In deployments where a Hub serves multiple teams/orgs, the Solution Server will be org-wide and accessible to any authenticated user for that Hub.
- **Risks & Limitations:** The lack of fine-grained RBAC or data obfuscation means that, for now, any user with Hub credentials can view or query any migration data collected by the Solution Server in their org. This may present security or IP risks in some deployments and will need to be revisited as RBAC and data-sharing features are developed in the future.
- **RBAC/Future Work:** Implementation of per-user or per-group access controls and exploring safe data-sharing methods (such as obfuscated hints or code tokenization) are explicitly deferred to future work.

## Initial Testing Focus

- Validate data collection, storage accuracy, and retrieval effectiveness.
- Confirm accuracy of computed success rates and insights (hints/metrics for IDE consumption).

## Potential Challenges

- Effectively managing large volumes of collected migration data.
- Maintaining high-quality and actionable insights from historical data.
- Ensuring users understand the privacy implications of solution sharing within a Hub.
