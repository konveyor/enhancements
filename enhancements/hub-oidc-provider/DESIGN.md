# Design Details: Builtin Auth

This document contains the detailed design specifications for the builtin Auth enhancement.

## Architecture Overview

The Hub will function as an OIDC provider with the following components:
- User and role-based access control (RBAC) managed in the Hub inventory
- Support for local authentication or delegation to external providers (OIDC, LDAP/Active Directory)
- API key-based authentication for programmatic access
- Token issuance, validation, and revocation

## Routes and Endpoints

### Resource Management Endpoints

| Method        | Path                 | Purpose                        |
|---------------|----------------------|--------------------------------|
| ANY           | /auth/users          | User collection                |
| ANY           | /auth/roles          | Roles collection               |
| GET           | /auth/permissions    | Permission collection          |
| GET \| DELETE | /auth/tokens         | Issued tokens                  |
| ANY           | /auth/apikeys        | API-Key collection             |
| GET           | /auth/idp/identities | Remote IDP identity collection |


### Standard OIDC Endpoints

| Method | Path                              | Purpose                                                                                        |
|--------|-----------------------------------|------------------------------------------------------------------------------------------------|
| GET    | /.well-known/openid-configuration | Discovery document – Tells clients all the endpoints, supported scopes, grant types, etc.      |
| GET    | /oidc/authorize                   | Authorization Endpoint – Starts the login flow (shows login form or redirects to external IdP) |
| POST   | /oidc/token                       | Token Endpoint – Exchanges authorization code for access_token + id_token + refresh_token      |
| GET    | /oidc/jwks                        | JSON Web Key Set – Public keys used by clients to verify your JWT signatures                   |
| GET    | /oidc/userinfo                    | UserInfo Endpoint – Returns user claims (optional, but commonly used)                          |
| POST   | /oidc/introspect                  | Token Introspection – Allows resource servers to validate opaque tokens (optional)             |
| POST   | /oidc/revoke                      | Token Revocation – Allows clients to revoke refresh tokens (optional but recommended)          |

## Data Model

### Entity Definitions

**Entities:**
- **User** - Known users in the system
- **Role** - Named groups of permissions (scopes)
- **Permission** - Named permissions mapped to scopes
- **IdpIdentity** - Identities authenticated by remote IdP provider. Contains the refresh token.
- **Token** - Issued (valid) tokens
- **APIKey** - Issued (valid) API-Keys

### Entity Relationship Diagram

```mermaid
erDiagram
    USER }o--o{ ROLE : "granted"
    ROLE }o--o{ PERMISSION : "has"
    USER ||--o{ API_KEY : "owns"
    TASK ||--o{ API_KEY : "owns"
    IDP_IDENTITY ||--|| USER : "EXTERNAL identity"
    IDP_IDENTITY ||--|| TOKEN : "delegated authentication"

    USER {
        uint id PK
        string userid "unique"
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

    API_KEY {
        uint id PK
        uint user_id FK
        uint task_id FK
        string digest "unique, hashed-secret"
        datetime expiration
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
        uint user_id FK
        uint idp_identity_id FK "0 = none"
        string jti "jwt id"
        datetime expiration
    }
    
    TASK {
    uint id PK
    }
```

### Implementation Notes

- The Token table contains hub-issued tokens only
- The Token._expiration_ column is mainly used for reaping expired tokens
- API-Keys are stored in the DB but cached in memory for performance and to mitigate DB pressure

## Authentication Flows

### Local Authentication Flow

The standard OIDC authorization code flow with local credential verification:

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

### External OIDC Provider Flow

When an external OIDC provider (e.g., Google, Keycloak) is configured, the Hub acts as a broker:

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

### LDAP / Active Directory Flow

LDAP authentication uses the search + bind pattern with group-based role mapping:

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

## Token Validation

### Standard Token Validation Flow

Supports JWT (RSA and deprecated HMAC) and API keys:

```mermaid
sequenceDiagram
    participant UI as UI / API Client
    participant ProtectedAPI as Protected API
    participant Hub as Hub Provider<br>(OIDC Provider)
    participant DB as API Key Database / Store

    Note over UI,ProtectedAPI: Bearer Token Validation Flow<br>(JWT RSA + JWT HMAC deprecated + API Keys)

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

### External IdP Token Validation

For users authenticated via external providers, validation includes refresh token checks:

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

## API Key Authentication

### Generation

API keys are generated via POST to `/auth/apikey` and inherit permissions from the creating user's roles.

**Request:**
```yaml
POST /auth/apikey

expiration: # OPTIONAL (datetime)
```

**Response:**
```yaml
id: 18
expiration: # OPTIONAL
key: cvP1sjff7_X2dCEIzUPf8f0IzKSbwiSDf1dZChZuRxY
```

**Implementation details:**
- Keys are 256-bit base64URL-encoded strings
- The key is stored as a hashed digest in the database
- Permissions (scopes) are inherited from the user's role assignments at creation time
- Keys can have optional expiration dates

### Authentication

API keys are presented as Bearer tokens:
```
Authorization: Bearer <key>
```

**Validation process:**
1. Extract the key from the Authorization header
2. Hash and lookup the stored key in the database
3. Verify the key is not expired or revoked
4. Retrieve associated permissions (scopes)
5. Authorize endpoint access based on scopes

### Addon Tokens

The task manager and addon API authorization will be refactored to use API keys instead of custom JWT token generation:
- New tasks will be configured with API keys for authentication
- Legacy support: Tokens with `SigningMethod=HMAC` will still be honored for backwards compatibility with in-flight tasks
- This provides a unified authentication mechanism across all programmatic access

## LDAP/Active Directory Group Mapping

### Policy Schema

The group-to-role mapping policy is expressed in YAML and can be managed through the UI. The policy defines how LDAP/AD group memberships map to internal roles.

**Structure:**
- Each mapping rule contains a conditional expression (`any` or `and`) and a list of `roles` to assign
- **`any`**: User matches if they belong to **at least one** of the listed groups (logical OR)
- **`and`**: User matches if they belong to **all** of the listed groups (logical AND)
- **`roles`**: List of internal role names to assign when the condition matches

**Evaluation rules:**
- All rules are evaluated for each user during authentication
- A user may match multiple rules and accumulate roles from all matches
- The final set of permissions (scopes) is the union of all assigned roles
- Group names are matched exactly (case-sensitive) against groups returned from LDAP/AD

### Example Policy

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

## Token Revocation

Tokens and API keys can be explicitly revoked:

- **DELETE /auth/tokens/:id** - Revokes a specific token (effective on next refresh)
- **DELETE /auth/apikeys/:id** - Revokes a specific API key (effective immediately)

Revocation is tracked in the database to ensure revoked credentials are not accepted.

## Configuration Management

### Client Configuration

OIDC clients are loaded from a ConfigMap seeded by the operator. This allows the operator to manage client registrations without requiring database access.

### Login UI

The login page UI fragment is read from a ConfigMap managed by the operator. This enables:
- Branding customizations managed by the operator
- Consistent UI updates across installations
- Separation of presentation from application logic

### Security

All sensitive information (passwords, refresh tokens) is stored encrypted in the database. API keys are stored as cryptographic hashes (digests) only, never in plain text.
