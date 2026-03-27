---
title: provide-builtin-authz
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

# Provide builtin AuthZ

Add builtin AuthZ functionality so that KeyCloak is no longer required.


## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] User-facing documentation is created

## Open Questions

## Summary

Currently, keycloak is required to provide user authentication and authorization. Both the Hub
and the UI integrate with Keycloak using keycloak clients. The goal convert to OIDC (OpenID)
for AuthZ integration and remove the dependence on keycloak.  Further, to provide an
internal OIDC provider with optional delegation to an external OIDC provider (such as Keycloak
but can be anything).

## Motivation

Eliminate dependence on Keycloak.

### Goals

- To provide AuthZ _out-of-the-box_.
- To (optionally) delegate authentication to an external OIDC provider
- To (optionally) delegate authorization to an external OIDC provider.
- To discontinue dependence on keycloak.
- To discontinue seeding the realm in keycloak.
- To support user management in the tackle UI.
  - User CRUD.
  - Role CRUD
  - User role assignment.

### Non-Goals

## Proposal

Make the hub an OIDC provider. The provider policy may be self-contained or configured
to delegate authentication and/or authorization to an external provider. Th hub inventory
is augmented to include Users and Roles. Users may be associated to roles and roles may
be associated to permissions (scopes).  Tokens will continue to contain scope claims.

```mermaid
sequenceDiagram
    participant UI as UI (Client Application)
    participant Hub as Hub Provider<br>(OIDC Provider)
    participant User as User
    participant DB as Database<br>(Users + Roles)

    Note over UI,Hub: Local Authorization Code Flow (No Consent)

    UI->>Hub: GET /oauth2/authorize<br>?client_id=...&scope=openid profile email
    activate Hub

    Hub->>User: Redirect to Login Page
    User->>Hub: Shows Login Form

    User->>Hub: POST login credentials<br>(username + password)
    activate Hub

    Hub->>DB: Authenticate User<br>(check username + password)
    DB-->>Hub: User record + associated Roles

    alt Invalid Credentials
        Hub->>User: Show Login Page with Error
        deactivate Hub
    else Valid Credentials
        Hub->>Hub: Set Subject (username)

        Hub->>DB: Get User Roles & Scopes
        DB-->>Hub: List of scopes from roles

        Hub->>Hub: Apply Authorization Layer<br>(merge requested scopes + role-based scopes)

        Hub-->>UI: Redirect back with authorization code
        deactivate Hub
    end

    Note over UI,Hub: Back-channel token exchange

    UI->>Hub: POST /oauth2/token<br>grant_type=authorization_code + code_verifier
    activate Hub

    Hub-->>UI: Access Token + ID Token + Refresh Token<br>(signed by Hub keys, containing role-based scopes)

    deactivate Hub

    Note over UI: UI now has tokens issued by Hub Provider<br>with local user identity + authorization from roles
```

When and external provider is configured, the login page rendered by the hub will contain a
button for this.  For example: "Login with Google".

```mermaid
sequenceDiagram
    participant UI as UI (Client Application)
    participant Hub as Hub <br>(OIDC Broker)
    participant ExternalIdP as External IdP<br>(e.g. Google)
    participant User as User
    participant DB as Database<br>(Users + Roles)

    Note over UI,Hub: Authorization Code Flow starts

    UI->>Hub: GET /oauth2/authorize<br>?client_id=...&scope=...&idp=google
    activate Hub

    Hub->>User: Redirect to External Login<br>(or show IdP selector)
    User->>ExternalIdP: Browser redirects to Google login page
    activate ExternalIdP

    ExternalIdP->>User: Show Google login / consent screen
    User->>ExternalIdP: Authenticates (username/password or SSO)

    ExternalIdP-->>User: Redirect back to Hub Provider<br>with authorization code
    deactivate ExternalIdP

    User->>Hub: Callback with code<br>(/oauth2/authorize/{callback_id}?code=...)

    Hub->>ExternalIdP: POST /token (exchange code)
    activate ExternalIdP
    ExternalIdP-->>Hub: ID Token + Access Token
    deactivate ExternalIdP

    Hub->>ExternalIdP: GET UserInfo (optional)
    ExternalIdP-->>Hub: User claims (sub, email, name, ...)

    Hub->>DB: Lookup or Create User<br>(map Google sub/email → local user)
    activate DB
    DB-->>Hub: Local user record + associated Roles

    Hub->>Hub: Apply Authorization Layer<br>(add role-based scopes, custom claims)
    deactivate DB

    alt Consent Required
        Hub->>User: Show Consent Screen<br>("Allow this app to access ...")
        User->>Hub: User approves (or denies)
    else Consent Already Given
        Note right of Hub: Skip consent
    end

    Hub-->>UI: Redirect back with authorization code
    deactivate Hub

    Note over UI,Hub: Back-channel token exchange

    UI->>Hub: POST /oauth2/token<br>grant_type=authorization_code + code_verifier
    activate Hub

    Hub-->>UI: Access Token + ID Token + Refresh Token<br>(signed by Hub keys, with local scopes/claims)

    deactivate Hub

    Note over UI: UI now has tokens issued by the Hub Provider<br>containing both external identity + local authorization (roles/scopes)
```

The UI will be updated to use OIDC (instead of keycloak) and be configured to use the
hub OIDC provider.  The UI will have pages to manage user, roles and permissions.

### Security, Risks, and Mitigations

The [go-oidc](https://github.com/luikyv/go-oidc) package is **OpenID certified** and is actively maintained. It has no reported CVEs.  AI code analysis
reports no vulnerabilities or backdoors.

## Design Details

### Test Plan

- Add hub binding tests for User, Role resources.
- Add authz tests using OIDC client.
- TBD

### Upgrade / Downgrade Strategy

