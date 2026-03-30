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

- **To discontinue dependence on keycloak**.
- To provide AuthZ _out-of-the-box_.
- To (optionally) delegate authentication to an external OIDC provider
- To (optionally) delegate authorization to an external OIDC provider.
- To discontinue seeding the realm in keycloak.
- To support user management in the tackle UI.
  - User CRUD.
  - Role CRUD with association scopes.
  - Grant roles to users.

### Non-Goals

## Proposal

Make the hub an OIDC provider. The security policy may be self-contained or configured
to delegate authentication and/or authorization to an external provider. The hub inventory
is augmented to include Users and Roles. Users may be associated to roles and roles may
be associated to permissions (scopes).

The UI will be updated to use OIDC (instead of keycloak) and be configured to use the
hub OIDC provider.  The UI will have pages to manage user, roles and permissions.

The Tackle CR will support configuring the hub to use an external OIDC provider but it will no longer install,
configure or seed it.  The installation and configuration of an external provider is the sole responsibility 
of the user.

The UI fragment used for the login page will be read from a ConfigMap managed by the operator.  Branding
customizations will be handled by the operator.

Publish a README.md that contains expected roles and a catalog of scopes to support user's bringing their
own external OIDC provider. This _may_ also include a recommended keycloak Realm specification.

### Security, Risks, and Mitigations

The [go-oidc](https://github.com/luikyv/go-oidc) package is **OpenID certified** and is actively maintained. It has no reported CVEs.  AI code analysis
reports no vulnerabilities or backdoors.

## Design Details

### Routes/Endpoints

Standard OIDC endpoints Provided by `go-oidc`

| Method | Endpoint Path                       | Purpose |
|--------|-------------------------------------|---------|
| GET    | `/.well-known/openid-configuration` | Discovery document – Tells clients all the endpoints, supported scopes, grant types, etc. |
| GET    | `/oauth2/authorize`                 | Authorization Endpoint – Starts the login flow (shows login form or redirects to external IdP) |
| POST   | `/oauth2/token`                     | Token Endpoint – Exchanges authorization code for access_token + id_token + refresh_token |
| GET    | `/oauth2/jwks`                      | JSON Web Key Set – Public keys used by clients to verify your JWT signatures |
| GET    | `/oauth2/userinfo`                  | UserInfo Endpoint – Returns user claims (optional, but commonly used) |
| POST   | `/oauth2/introspect`                | Token Introspection – Allows resource servers to validate opaque tokens (optional) |
| POST   | `/oauth2/revoke`                    | Token Revocation – Allows clients to revoke refresh tokens (optional but recommended) |

### High Level Model:

```mermaid
erDiagram
    USER }o--o{ ROLE : "granted"
    ROLE }o--o{ PERMISSION : "has"
    IDP_IDENTITY ||--|| USER : "EXTERNAL identity"
    IDP_IDENTITY ||--|| TOKEN : "delegated authentication"

    USER {
        uint id PK
        string username "unique"
        string password "encrypted"
        string email "unique"
    }

    ROLE {
        uint id PK
        string name "unique"
    }

    PERMISSION {
        uint id PK
        string name "human readable name"
        string scope "scope"
    }
    
    IDP_IDENTITY {
        uint id PK
        uint user_id FK
        string provider "Ex: google"
        string subject
        string refresh_token "(encrypted)"
        datetime expiration
        datetime last_authenticated
        datetime last_refreshed
    }
    
    TOKEN {
        uint id PK
        uint user_id
        uint idp_identity_id FK "0 = none" 
        string jti "jwt id"
        datetime expiration
        datetime revoked
        datetime last_authenticated
        datetime last_refreshed
    }
```

#### Notes:
- The Token table contains hub issued tokens.
- The _expiration_ column is mainly used for reaping.

### Login

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

### Login (external Authentication)

when and external provider is configured, the login page rendered by the hub will contain a
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

### Token Validation

```mermaid
sequenceDiagram
    participant UI as UI / API Client
    participant Hub as Hub Provider<br>(OIDC Provider)
    participant ProtectedAPI as Protected API / Resource Server
    participant DB as Database<br>(Optional)

    Note over UI,ProtectedAPI: Token Validation Flow (most common pattern)

    UI->>ProtectedAPI: GET /api/projects<br>Authorization: Bearer <access_token>
    activate ProtectedAPI

    ProtectedAPI->>ProtectedAPI: 1. Verify JWT Signature<br>(using Hub's JWKS)
    ProtectedAPI->>ProtectedAPI: 2. Validate Standard Claims<br>(iss, aud, exp, nbf, iat)

    alt Token is Invalid or Expired
        ProtectedAPI-->>UI: 401 Unauthorized
    else Token is Valid
        ProtectedAPI->>ProtectedAPI: 3. Extract Claims<br>(sub, scope, roles if present)

        %% Optional: Extra authorization check using your DB
        ProtectedAPI->>DB: 4. Optional: Lookup user roles<br>by sub (username)
        DB-->>ProtectedAPI: Current roles & scopes

        ProtectedAPI->>ProtectedAPI: 5. Check Required Scopes<br>e.g. HasScope("api:read") or role-based check

        alt Authorization Failed
            ProtectedAPI-->>UI: 403 Forbidden
        else Authorization Passed
            ProtectedAPI-->>UI: 200 OK + Response Data
        end
    end

    deactivate ProtectedAPI

    Note over ProtectedAPI: Common Validation Order:<br>1. Signature + Issuer<br>2. Expiration<br>3. Audience<br>4. Scopes / Claims<br>5. Custom business rules
```

### Token validation (external authentication)

```mermaid
sequenceDiagram
    participant UI as UI / Client
    participant Hub as Hub Provider
    participant ProtectedAPI as Protected API
    participant DB as Database
    participant ExternalIdP as External IdP (Google, etc.)

    Note over Hub,DB: During initial login: Store external refresh_token linked to local user

    UI->>ProtectedAPI: Request with Hub access_token
    activate ProtectedAPI

    ProtectedAPI->>ProtectedAPI: Verify Hub JWT signature + claims
    ProtectedAPI->>DB: Lookup user + stored external refresh_token

    alt Re-validation Enabled
        ProtectedAPI->>ExternalIdP: Attempt refresh<br>(POST /token with refresh_token)
        ExternalIdP-->>ProtectedAPI: Success → new external tokens<br>OR Failure (invalid_grant / revoked)
        
        alt External Refresh Failed (Revoked)
            ProtectedAPI->>ProtectedAPI: Mark session as invalid / force re-login
            ProtectedAPI-->>UI: 401 Unauthorized + prompt to re-authenticate
        else External Refresh Succeeded
            ProtectedAPI->>ProtectedAPI: Optional: Update stored tokens
            ProtectedAPI->>ProtectedAPI: Check local roles/scopes
            ProtectedAPI-->>UI: 200 OK
        end
    else No Re-validation (simple mode)
        ProtectedAPI->>DB: Only check local user status + roles
        ProtectedAPI-->>UI: 200 OK (until Hub token expires)
    end

    deactivate ProtectedAPI
```

### Test Plan

- Add hub binding tests for User, Role resources.
- Add authz tests using OIDC client.
- TBD

### Upgrade / Downgrade Strategy

