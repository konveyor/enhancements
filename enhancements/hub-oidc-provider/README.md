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

### OIDC

Make the hub an OIDC provider. The security policy may be self-contained or configured
to delegate authentication and/or authorization to an external provider. The hub inventory
is augmented to include Users and Roles. Users may be associated to roles and roles may
be associated to permissions (scopes).

The UI will be updated to use OIDC (instead of keycloak) and be configured to use the
hub OIDC provider.  The UI will have pages to manage user, roles and permissions.

The UI fragment used for the login page will be read from a ConfigMap managed by the operator.  Branding
customizations will be handled by the operator.

The hub may be configured to integrate with external (remote) auth providers. The primary mechanism will be OIDC but will
also support LDAP/ActiveDirectory.  In both cases, scopes (OIDC) and groups (LDAP) may need to be mapped to tackle roles. External
providers may be created, updated and deleted using the UI/API. Configuration includes connection information
credentials and group:role mapping rules. 

Publish a README.md that contains expected roles and a catalog of scopes to support user's bringing their
own external OIDC provider. This _may_ also include a recommended keycloak Realm specification.

All sensitive information will be stored encrypted.

Access tokens may be revoked (effective next refresh).

### API-Key

Add support for API-Keys (Reference [RFE-266](https://github.com/konveyor/enhancements/issues/266)).

#### Generation
POST /auth/apikey returns a 256-bit base64-encoded generated key which has been stored in the DB along with associated
permissions (scopes).

#### Revocation

DELETE /auth/apikey/:key

#### Authentication

HTTP requests with header: Authentication: Bearer `<key>` is detected and validated:
1. Find/match stored key.
2. Get mapped permissions (scopes).
3. Authorize endpoint access.

#### Addon tokens

Refit task manager and addon API authorization to use API-Key instead of custom JWT token generation.
For backwards compatibility, Tokens with SigningMethod=HMAC still honored to support in-flight tasks
with these tokens.  However, new tasks will be configured to present API-Keys.


### Security, Risks, and Mitigations

The [go-oidc](https://github.com/luikyv/go-oidc) package is **OpenID certified** and is actively maintained. It has no reported CVEs.  AI code analysis
reports no vulnerabilities or backdoors.

## Design Details

### Routes/Endpoints

Standard OIDC endpoints Provided by `go-oidc`

| Method | Endpoint Path                       | Purpose |
|--------|-------------------------------------|---------|
| GET    | `/.well-known/openid-configuration` | Discovery document – Tells clients all the endpoints, supported scopes, grant types, etc. |
| GET    | `/oidc/authorize`                   | Authorization Endpoint – Starts the login flow (shows login form or redirects to external IdP) |
| POST   | `/oidc/token`                       | Token Endpoint – Exchanges authorization code for access_token + id_token + refresh_token |
| GET    | `/oidc/jwks`                        | JSON Web Key Set – Public keys used by clients to verify your JWT signatures |
| GET    | `/oidc/userinfo`                    | UserInfo Endpoint – Returns user claims (optional, but commonly used) |
| POST   | `/oidc/introspect`                  | Token Introspection – Allows resource servers to validate opaque tokens (optional) |
| POST   | `/oidc/revoke`                      | Token Revocation – Allows clients to revoke refresh tokens (optional but recommended) |

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

### Login (external Idp)

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

### Login (LDAP | Activity Directory)

TLDR:
- ODDC (started)
- Query user using service account.
- Bind as user. (authentication)
- Query for group membership.
- Map groups to roles (and then scopes)
- OIDC (continued).


```mermaid
sequenceDiagram
    participant UI as UI / Client
    participant Hub as Hub Provider (OIDC)
    participant LDAP as LDAP / Active Directory
    participant ProtectedAPI as Protected API
    participant DB as Database

    Note over Hub,LDAP: Authentication uses Search + Bind pattern<br>Group retrieval via memberOf (AD) or group search

    %% === Initial Login Flow (Authorization Code Flow) ===
    UI->>Hub: GET /authorize (client_id, scope=openid profile email, redirect_uri...)
    Hub-->>UI: Redirect to Hub Login Page

    UI->>Hub: POST Login (username + password)
    activate Hub

    Hub->>LDAP: 1. Search for user (by sAMAccountName / uid) → get DN
    LDAP-->>Hub: User DN + basic attributes

    Hub->>LDAP: 2. Bind as user DN with provided password
    LDAP-->>Hub: Bind Success / Failure

    alt Authentication Failed
        Hub-->>UI: Login failed → show error
    else Authentication Succeeded
        Hub->>LDAP: 3. Fetch groups (memberOf attribute or group search)
        LDAP-->>Hub: List of group DNs / CNs (e.g. IT-Admins, Finance-Users...)

        Hub->>Hub: Apply YAML group rules (any/and + glob matching)<br>→ Compute internal roles & scopes
        Hub->>DB: Create / update local user session (optional: store last groups, roles)

        Hub-->>UI: Redirect back with authorization code
    end

    deactivate Hub

    UI->>Hub: POST /token (grant_type=authorization_code, code=...)
    activate Hub
    Hub->>Hub: Validate code + issue tokens
    Hub-->>UI: Access Token (JWT with roles/scopes) + ID Token + Refresh Token
    deactivate Hub

    Note over Hub,DB: Access Token contains pre-computed roles/scopes from LDAP groups

    %% === Subsequent API Requests ===
    UI->>ProtectedAPI: Request with Hub access_token
    activate ProtectedAPI

    ProtectedAPI->>ProtectedAPI: Verify Hub JWT signature + claims (roles/scopes)
    ProtectedAPI->>DB: Optional: Lookup user + session metadata

    alt Re-validation / Group Refresh Enabled
        ProtectedAPI->>Hub: Optional introspection or /userinfo (or direct LDAP check if designed)
        Hub->>LDAP: Re-fetch current groups (if needed)
        Hub->>Hub: Re-apply rules → updated roles
        Hub-->>ProtectedAPI: Updated claims / decision
    else Simple Mode (no re-validation)
        ProtectedAPI->>ProtectedAPI: Check local roles/scopes from token
    end

    alt Access Allowed
        ProtectedAPI-->>UI: 200 OK + response
    else Access Denied
        ProtectedAPI-->>UI: 403 Forbidden
    end

    deactivate ProtectedAPI
```

### Token Validation

```mermaid
sequenceDiagram
    participant UI as UI / API Client
    participant Hub as Hub Provider<br>(OIDC Provider)
    participant API as REST API
    participant DB as API Key Database / Store

    Note over UI,ProtectedAPI: Updated Bearer Token Validation Flow<br>(JWT RSA + JWT HMAC deprecated + API Keys)

    UI->>ProtectedAPI: GET /api/projects<br>Authorization: Bearer <token>
    activate ProtectedAPI

    ProtectedAPI->>ProtectedAPI: 1. Extract Bearer token

    alt Token is a JWT (contains exactly two '.' characters)
        ProtectedAPI->>ProtectedAPI: 2. Decode JWT Header (read alg)

        alt alg starts with "RS" (RSA)
            ProtectedAPI->>ProtectedAPI: 3a. Verify RSA Signature using Hub JWKS
            note right of ProtectedAPI: Recommended
        else alg starts with "HS" (HMAC)
            ProtectedAPI->>ProtectedAPI: 3b. Verify HMAC Signature using shared secret
            note right of ProtectedAPI: Deprecated but supported for backwards compat
        else Unsupported algorithm
            ProtectedAPI-->>UI: 401 Unauthorized
            note right of ProtectedAPI: Reject unknown algs including "none"
        end

        ProtectedAPI->>ProtectedAPI: 4. Validate Standard JWT Claims (iss, aud, exp, etc.)
        ProtectedAPI->>ProtectedAPI: 5. Extract Claims (sub, scope, roles...)

    else Token is NOT a valid JWT → Treat as API Key
        ProtectedAPI->>DB: 2b. Lookup API Key in DB
        activate DB
        DB-->>ProtectedAPI: Key record (active?, scopes/permissions)
        deactivate DB

        alt API Key invalid or revoked
            ProtectedAPI-->>UI: 401 Unauthorized
        else API Key valid
            ProtectedAPI->>ProtectedAPI: 3c. Map DB permissions to scopes
        end
    end

    alt Token validation failed
        ProtectedAPI-->>UI: 401 Unauthorized
    else Token is valid
        ProtectedAPI->>ProtectedAPI: 6. Perform Authorization Check<br>(unified scope check)
        alt Authorization fails
            ProtectedAPI-->>UI: 403 Forbidden
        else Authorization passed
            ProtectedAPI-->>UI: 200 OK + Response
        end
    end

    deactivate ProtectedAPI

    Note over ProtectedAPI: Key Notes<br/>RSA via JWKS is strongly preferred.<br/>HMAC is deprecated but honored for backwards compatibility.<br/>API Keys are mapped to the same scope model as JWTs.<br/>Always reject alg="none" and unknown algorithms.<br/>Consider logging HMAC usage for migration tracking.
```

### Token validation (external Idp)

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
        ProtectedAPI-->>UI: 200 OK (until Hub token expires)
    end

    deactivate ProtectedAPI
```

#### Policy

The role mapping policy can be expressed and edit by the UI as simple YAML.

```yaml
groups:
  # Administrators
  - any:
      - Engineering-Admins
      - Platform-Admins
      - IT-Admins
      - SEC-Global-Admins
      - Infrastructure-Admins
    roles:
      - admin

  # Managers
  - and:
      - Engineering
      - Engineering-Managers
      - Managers
    roles:
      - manager

  # Architects
  - any:
      - Engineering-Architects
      - Principal-Architects
      - Solution-Architects
      - Tech-Leads
    roles:
      - architect

  # Migrators
  - any:
      - Engineering-Migration-Team
      - Database-Admins
      - SRE-Migration
      - Platform-Migration
      - Data-Migration-Team
    roles:
      - migrator

  # High-privilege production access
  - and:
      - Engineering
      - Prod-Admins
    roles:
      - admin
      - migrator
```



### Test Plan

- Add hub binding tests for User, Role resources.
- Add authz tests using OIDC client.
- TBD

### Upgrade / Downgrade Strategy

