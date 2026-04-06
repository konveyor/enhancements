---
title: provide-builtin-auth
authors:
  - "jortel@redhat.com"
reviewers:
  - TBD
approvers:
  - TBD
creation-date: 2026-3-27
last-updated: 2026-3-27
status: implementable
---

# Provide builtin Auth

Add builtin AuthN and AuthZ functionality so that KeyCloak is no longer required.


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

- What kinds of authentication options needed to query LDAP.

## Summary

Currently, Keycloak is required to provide user authentication and authorization. Both the Hub and the UI integrate with Keycloak using Keycloak clients. The goal is to convert to OIDC (OpenID Connect) for authentication and authorization integration and remove the dependence on Keycloak. This will be achieved by providing an internal OIDC provider within the Hub, with optional delegation to external OIDC providers (such as Keycloak, Google, Okta, or others).

## Motivation

Eliminate dependence on Keycloak.

### Goals

- **To discontinue dependence on keycloak**.
- To discontinue seeding the realm in keycloak.
- To provide OIDC-based Auth _out-of-the-box_.
- To delegate AuthN, AuthZ to an _external_ OIDC provider. (option)
- To delegate AuthN, AuthZ to an _external_ LDAP, Active Directory with group mapping. (option)
- To provide RBAC (users, roles, permissions) management in the inventory.
- To provide for API-Key authentication.  An API-Key is a generated secret used for application integration.

### Non-Goals

## Proposal

This proposal transforms the Hub into an OIDC-compliant identity provider, eliminating the hard dependency on Keycloak while providing flexible authentication options.

### User Stories

**As a cluster administrator**, I want to deploy Konveyor without requiring Keycloak, so that I can reduce infrastructure complexity and resource overhead.

**As a security administrator**, I want to integrate Konveyor with my organization's existing identity provider (OIDC, LDAP, or Active Directory), so that I can enforce centralized authentication policies and maintain a single source of truth for user identities.

**As a developer**, I want to use API keys for programmatic access to Konveyor APIs, so that I can build automated integrations without managing user credentials.

**As a Hub administrator**, I want to manage users, roles, and permissions directly within the Hub UI, so that I can control access without needing to configure external identity systems for simple deployments.

### High-Level Approach

#### OIDC Provider

The Hub will implement a standards-compliant OIDC provider supporting the authorization code flow with PKCE. This provider can operate in several modes:

1. **Self-contained mode**: Users, roles, and permissions are managed entirely within the Hub's inventory. Authentication uses locally stored credentials.

2. **External OIDC delegation**: The Hub acts as an OIDC broker, delegating authentication to external providers (Google, Keycloak, Okta, etc.) while maintaining local authorization (role and permission management). This allows organizations to leverage existing identity providers while keeping fine-grained access control within Konveyor.

3. **LDAP/Active Directory integration**: The Hub authenticates users against LDAP or Active Directory using the standard bind pattern, queries group memberships, and maps groups to local roles using configurable YAML-based rules.

The Hub inventory will be extended with new entities for users, roles, permissions, tokens, API keys, and external identity mappings. Users are assigned to roles, and roles are associated with permissions (represented as OAuth scopes).

The UI will be refactored to use OIDC for authentication (replacing the current Keycloak client integration) and will include new pages for managing users, roles, and permissions. The login page will be served from a ConfigMap managed by the operator, enabling branding customization without code changes.

All standard OIDC discovery endpoints will be implemented (`.well-known/openid-configuration`, authorization, token, JWKS, userinfo, introspection, and revocation), allowing any OIDC-compliant client to integrate with the Hub.

#### Default Role Definitions

The system will include predefined roles tailored to common migration project personas (administrators, architects, project managers, migration specialists). These roles provide sensible permission defaults out-of-the-box while remaining fully customizable. Organizations can use these as-is for quick deployments or customize them to match their specific governance requirements.

#### Flexible Role and Permission Management

Role and permission definitions support multiple management approaches to accommodate different operational models:
- **Operator-managed**: Roles can be defined in configuration files and deployed consistently across environments
- **UI-managed**: Administrators can create, modify, and delete roles and permissions through the web interface
- **Hybrid**: Organizations can seed baseline roles via configuration while allowing runtime customization through the UI

This flexibility enables both centralized governance (via operator control) and decentralized administration (via UI self-service).

#### Resource-Based Authorization Model

Authorization is enforced using a resource-and-action permission model. Permissions define what actions (create, read, update, delete) can be performed on which resources (applications, assessments, tasks, etc.). This maps naturally to REST API operations and provides fine-grained access control without requiring administrators to understand OAuth scope syntax.

Permissions can be scoped broadly (all operations on a resource) or narrowly (specific operations on specific sub-resources), supporting both simple and complex authorization requirements.

#### API Key Authentication

API keys provide a secure mechanism for programmatic access, addressing [RFE-266](https://github.com/konveyor/enhancements/issues/266). Users can generate API keys through the Hub API, which inherit the user's permissions at the time of creation. Keys are presented as Bearer tokens and validated using the same scope-based authorization model as JWT tokens.

The task manager and addon API will be refactored to use API keys instead of custom HMAC-signed JWTs. For backwards compatibility, existing HMAC tokens will continue to be honored for in-flight tasks.

API keys support optional expiration dates and can be explicitly revoked. The keys themselves are not stored in the database—only cryptographic digests—ensuring that compromise of the database does not directly expose API keys.

#### Integration

Client tools such as KAI and Kantra will obtain an API-Key by posting to the API-Key endpoint.  The
request body will include the userid, password and _optional_ lifespan (minutes). For Go clients
such as Kantra, this will behave much like the current token-based authentication.

#### Task-Scoped Authentication

API keys can be associated with either users or individual tasks. This enables task-specific authentication where a running task (such as an analysis addon) has exactly the permissions it needs for its operation, independent of the user who initiated it. This separation improves security by limiting the blast radius of compromised credentials and enables better audit trails for automated operations.

#### Security Considerations

- All sensitive data (passwords, refresh tokens) will be stored encrypted
- API keys are stored as hashed digests, never in plain text
- Token validation supports both RSA-signed JWTs (via JWKS) and API keys
- HMAC-signed tokens are deprecated but maintained for backwards compatibility
- Tokens and API keys can be explicitly revoked with immediate (API keys) or deferred (tokens on next refresh) effect
- OIDC clients are configured via operator-managed Secrets rather than stored in the database

#### External Provider Integration

When configured with external providers, the Hub stores refresh tokens (encrypted) to enable ongoing validation. For LDAP/AD integrations, group-to-role mapping is expressed as simple YAML rules supporting both "any" (OR) and "and" (AND) logic, allowing flexible policy definition without code changes.

A README will be published documenting the expected roles and permission scopes catalog to support administrators who want to configure external OIDC providers or Keycloak realms to work with the Hub.

#### Development and Testing Support

The authentication system supports a development mode where authentication can be disabled entirely, simplifying local development and automated testing scenarios. This mode is controlled by configuration and clearly separated from the production authentication path to prevent accidental deployment with authentication disabled.

#### Operational Flexibility

The system is designed to adapt to different organizational requirements without architectural changes. Deployments can start simple with local authentication and evolve to integrate with enterprise identity providers as needs change, without requiring migration or re-architecture.

### Security, Risks, and Mitigations

The [go-oidc](https://github.com/luikyv/go-oidc) package is **OpenID certified** and is actively maintained. It has no reported CVEs.  AI code analysis
reports no vulnerabilities or backdoors.

## Design Details

See [DESIGN.md](./DESIGN.md) for complete technical specifications (for illustration not approval) including:
- API endpoints and routes
- Data model and entity relationships
- Authentication flow diagrams (local, external OIDC, LDAP/AD)
- Token validation sequences
- API key implementation details
- LDAP/AD group mapping policy schemas and examples

### Test Plan

**Unit Testing:**
- Hub model bindings for User, Role, Permission, Token, APIKey, and IdpIdentity resources
- API key generation, validation, and revocation logic
- Token issuance and validation logic
- Group-to-role mapping policy evaluation

**Integration Testing:**
- OIDC authorization code flow with local authentication
- OIDC delegation to external providers (using test IdP)
- LDAP/AD authentication and group mapping (using test LDAP server)
- API key authentication and authorization
- Token refresh and revocation
- UI authentication flows using the Hub OIDC provider

**End-to-End Testing:**
- Complete user workflows: login, access protected resources, logout
- Role and permission management through the UI
- API key creation and usage for programmatic access
- External provider integration scenarios

### Upgrade / Downgrade Strategy

