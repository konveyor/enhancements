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

Currently, keycloak is required to provide user authentication and authorization. Both the Hub
and the UI integrate with Keycloak using keycloak clients. The goal convert to OIDC (OpenID)
for AuthZ integration and remove the dependence on keycloak.  Further, to provide an
internal OIDC provider with optional delegation to an external OIDC provider (such as Keycloak
but can be anything).

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

The Hub inventory will be extended with new entities for users, roles, permissions, tokens, API keys, and external identity mappings. Users are assigned to roles, and roles are associated with permissions (represented as OAuth scopes). See [DESIGN.md](./DESIGN.md) for the complete data model.

The UI will be refactored to use OIDC for authentication (replacing the current Keycloak client integration) and will include new pages for managing users, roles, and permissions. The login page will be served from a ConfigMap managed by the operator, enabling branding customization without code changes.

All standard OIDC discovery endpoints will be implemented (`.well-known/openid-configuration`, authorization, token, JWKS, userinfo, introspection, and revocation), allowing any OIDC-compliant client to integrate with the Hub.

#### API Key Authentication

API keys provide a secure mechanism for programmatic access, addressing [RFE-266](https://github.com/konveyor/enhancements/issues/266). Users can generate API keys through the Hub API, which inherit the user's permissions at the time of creation. Keys are presented as Bearer tokens and validated using the same scope-based authorization model as JWT tokens.

The task manager and addon API will be refactored to use API keys instead of custom HMAC-signed JWTs. For backwards compatibility, existing HMAC tokens will continue to be honored for in-flight tasks.

API keys support optional expiration dates and can be explicitly revoked. The keys themselves are not stored in the database—only cryptographic digests—ensuring that compromise of the database does not directly expose API keys.

#### Security Considerations

- All sensitive data (passwords, refresh tokens) will be stored encrypted
- API keys are stored as hashed digests, never in plain text
- Token validation supports both RSA-signed JWTs (via JWKS) and API keys
- HMAC-signed tokens are deprecated but maintained for backwards compatibility
- Tokens and API keys can be explicitly revoked with immediate (API keys) or deferred (tokens on next refresh) effect
- OIDC clients are configured via operator-managed ConfigMaps rather than stored in the database

#### External Provider Integration

When configured with external providers, the Hub stores refresh tokens (encrypted) to enable ongoing validation. For LDAP/AD integrations, group-to-role mapping is expressed as simple YAML rules supporting both "any" (OR) and "and" (AND) logic, allowing flexible policy definition without code changes.

A README will be published documenting the expected roles and permission scopes catalog to support administrators who want to configure external OIDC providers or Keycloak realms to work with the Hub.

### Security, Risks, and Mitigations

The [go-oidc](https://github.com/luikyv/go-oidc) package is **OpenID certified** and is actively maintained. It has no reported CVEs.  AI code analysis
reports no vulnerabilities or backdoors.

## Design Details

See [DESIGN.md](./DESIGN.md) for complete technical specifications including:
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

