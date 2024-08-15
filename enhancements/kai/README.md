---
title: kai-integration-into-konveyor
authors:
  - "@fabianvf"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2024-08-14
last-updated: 2024-08-15
status: implementable

---

# Kai Integration into Konveyor

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are documented
- [ ] Test plan is defined
- [ ] User-facing docs are created

## Open Questions

1. What are the performance implications, and what changes are needed to the documented CPU/memory requirements.

## Summary

This enhancement integrates the AI component Kai into Konveyor. The goal is to boost Konveyor's capabilities by tracking incident resolutions and using them to automate code migrations. The integration is phased, starting with a proof of concept (now completed) and moving towards a fully polished release.

## Motivation

Integrating Kai with Konveyor improves application migrations. Kai’s AI-driven analysis and diff generation help manage and track incident resolutions, providing a valuable history for future analyses.

### Goals

- Integrate Kai as an API service with Konveyor.
- Ensure smooth deployment with minimal extra steps for system administrators.
- Enable konveyor authentication on the Kai API routes.

### Non-Goals

- No UI components for Kai are included.
- No major changes to Konveyor beyond API interactions.

## Proposal

### User Stories

#### Story 1
As a developer, I want to track and store changes in application analyses so I can easily reference resolved incidents from the IDE plugin and automate code migrations.

#### Story 2
As a system admin, I want to deploy Kai with Konveyor with minimal setup to streamline the process.

### Implementation Details

- **Completed**: Phase 1 - Prototype/POC
  - Kai runs alongside Konveyor in a development setup.
  - Developed a Konveyor importer that polls the API.
  - Added commit information to analysis reports.
  - Implemented filtering and polling mechanisms in the API.
  - Ensured historical analyses are correctly ingested.

- **Phase 2**: Prototype Integration
  - Finalize and release all Konveyor APIs developed in Phase 1.
  - Implement a manual archiving mechanism for analysis reports after Kai’s ingestion.
  - Improve the Konveyor importer to work with Konveyor's authentication flow, setting environment variables with access tokens.
  - Build a releasable image that includes the Kai project.
  - Update the Konveyor operator with a flag to deploy Kai alongside the Hub.
  - Automatically set archive variables and generate relevant keys during deployment.
  - Proxy Service: A proxy service for authentication is implemented. This service authorizes requests and proxies them to a Kubernetes service, ensuring that Kai’s endpoints are secured by Konveyor’s existing authentication mechanisms.

- **Phase 3**: Final Integration
  - Add the authentication flow into the IDE plugin.
  - Verify and test all Kai use cases in a production environment.
  - Finalize documentation and user-facing materials.
  - Ensure the integration is polished and ready for full production deployment.


### Security, Risks, and Mitigations

- Authentication is managed through the proxy service, integrating with Konveyor’s existing setup.

## Design Details

### Test Plan

- Unit tests for each API endpoint.
- End-to-end tests to ensure the integration works smoothly.
- Integration tests to verify interaction between Kai and Konveyor’s API.
- Eventual integration into the global-ci flow

### Upgrade / Downgrade Strategy

- Upgrades maintain compatibility with existing Konveyor setups.
- Downgrades disable Kai and revert to the standard Konveyor setup.

## Implementation History

- **Phase 1**: Prototype/POC - Completed; Kai runs in a dev environment.
- **Phase 2**: Prototype Integration - In Progress; Build a usable release.
- **Phase 3**: Final Integration - Finalize and polish for production.

## Drawbacks

- The integration adds complexity, which may require more resources for ongoing support.

## Infrastructure Needed

- CI/CD pipelines for testing and deploying Kai and Konveyor.
