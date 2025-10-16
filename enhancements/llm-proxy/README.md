---
title: llm-proxy
authors:
  - "@fabianvf"
reviewers:
  - "@dymurray"
  - "@jwmatthews"
  - "@jortel"
  - "@rromannissen"
  - "@shawn-hurley"
  - "@djzager"
  - "@ibolton"
  - "@sjd78"
approvers:
  - "@dymurray"
  - "@jwmatthews"
  - "@rromannissen"
  - "@jortel"
  - "@shawn-hurley"
creation-date: 2025-10-09
last-updated: 2025-10-09
status: provisional
see-also:
  -
replaces:
  -
superseded-by:
  -
---

## Summary

Introduce a lightweight LLM proxy service, deployed alongside the Hub and Solution Server in Kubernetes.
The proxy provides a centralized, administrator-controlled entry point for LLM access, ensuring that real LLM credentials never leave the cluster.

When authentication is enabled, the proxy validates incoming requests using JWT tokens issued by Keycloak. End users (for example, IDE plugins or other clients) authenticate using only their Hub credentials, while the proxy uses a cluster-side secret to connect to the underlying LLM provider.

This design decouples user identity from backend LLM credentials, enabling administrators to configure, rotate, or restrict LLM access centrally without distributing API keys to every client.

## Motivation

The current model has each client independently managing its connection to the LLM provider. This creates:

- **Security exposure:** API keys are widely distributed across clients.
- **Operational friction:** Every client needs its own configuration and credentials.
- **Limited control:** Admins cannot easily standardize or restrict LLM usage policies.

The **LLM Proxy** addresses these issues by introducing a single, managed access point that:
- Lets admins configure and secure the LLM provider credentials once.
- Allows users to access LLMs using their existing Hub identity (Keycloak-backed JWTs).
- Simplifies configuration distribution through the **Centralized Config** system.
- Lays the groundwork for better auditing and usage tracking in future iterations.

### Goals
- Deploy a dedicated LLM Proxy using llama-stack in Kubernetes.
- Authenticate users via JWT validation against the Hub's Keycloak instance.
- Connect to LLM providers using credentials stored in a Kubernetes secret.
- Integrate proxy endpoint metadata into the **Centralized Config** blob for downstream clients.
- Ensure admins can configure, rotate, and control LLM connections without exposing credentials.

### Non-Goals
- No full OIDC login flow; only JWT validation.
- No initial implementation of request logging or metering (future consideration).
- No support for dynamic multi-provider routing in this phase.

---

## Proposal

### Architecture

The LLM Proxy will be deployed as its own Kubernetes Deployment, co-located with the Hub and Solution Server.
It will expose its own ingress, and when auth is enabled, validate JWT tokens against the hub Keycloak instance.
It will read LLM credentials from a Kubernetes secret that must be manually created, similar to the Solution Server.

#### High-Level Flow
```
[User / IDE / Hub Client]
      │  (creds)       │  (JWT)
      V                │
 ┌──────────────┐      │
 │   Hub API    │      │
 └──────────────┘      │
                       │
                       V
 ┌──────────────────────┐
 │      LLM Proxy       │  <- validates JWT (Keycloak)
 │   (llama-stack)      │  <- uses Secret for LLM auth
 └──────────────────────┘
       │
       V
   [External LLM]
```

### Authentication

- Clients send `Authorization: Bearer <JWT>` obtained from the Hub/Keycloak.
- The proxy validates tokens using Keycloak’s JWKS endpoint:
  ```
  https://<keycloak>/realms/<realm>/protocol/openid-connect/certs
  ```
- On success, forwards the request to the configured LLM backend.
- On failure, returns standard 401/403 responses.

No login or token exchange occurs inside the proxy; it only performs token validation.

### Configuration & Integration

- Admins define the LLM backend configuration (provider, model, credentials) via standard cluster configuration (ConfigMaps/Secrets).
- The Hub consumes proxy metadata (endpoint, model info).
- The Hub passes that metadata downstream to IDE clients via the **Centralized Config EP (#241)**.
- This allows users to connect to LLMs automatically through their Hub identity, without direct credential management.
    - This assumes that the IDE implements using the Hub JWT as an API key for the LLM configuration.

### Security & Auditability

**Strengths:**
- Removes API key exposure from clients.
- Enforces per-user authentication via JWT validation.
- Enables centralized credential rotation.
- Provides a single, auditable ingress for LLM interactions.

**Future Enhancements:**
- Structured request/response logging.
- User-to-request correlation

### Implementation Details (Initial Phase)

**Credential Reuse (Solution Server -> Proxy):**
- For the initial rollout, the proxy will reuse the existing Solution Server LLM credentials.

**llama-stack wiring:**
- Run llama-stack in OpenAI-compatible mode; set `base_url` and default model.
- Mount Secret as env/file; map to provider adapter (e.g., `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc.).
- Enforce JWT validation middleware when auth is enabled for the hub.

**Networking & Discovery:**
- Expose a stable ClusterIP Service and Ingress/Route (for IDE clients).
- Hub reads proxy endpoint from Kubernetes resources (or mounted config) and publishes it via Centralized Config.
- IDEs point at the proxy’s OpenAI-compatible endpoint, using the JWT as the `OPENAI_API_KEY`.

**Operational Bits:**
- Readiness checks: JWKS fetch + provider health (optional HEAD/GET).
- Logs: redact `Authorization` headers and sensitive fields; include request IDs and user `sub` (claim) only.
- Resource limits sized similar to Solution Server; allow horizontal scaling (stateless).


### Security Considerations

- All tokens are validated via Keycloak’s JWKS.
- Logs must redact all tokens and sensitive headers.
- LLM credentials remain cluster-scoped and isolated.
- If Keycloak is unreachable, proxy should degrade safely (deny all requests).

### Test Plan

- Integration test with valid/invalid JWTs.
- E2E flow with Hub -> Proxy -> LLM provider.
- Secret rotation test to confirm runtime reload.

### Future Work

- Multi-provider support (Ollama, Anthropic, Fireworks, etc.).
- Request/response logging and metrics.
- Optional caching and rate limiting.
- Audit log correlation between Hub users and proxy requests.

#### Future Enhancement: Admin UI for LLM Configuration

**Goal:** Let Hub admins configure LLM providers and policies in the **Hub UI**, with changes propagated through Centralized Config. No provider credentials are ever exposed to non-admin users or IDEs.

**Scope (Phase 2/3):**
- **Providers:** Add/remove providers (OpenAI/Fireworks/Ollama/Anthropic/etc.), set base URLs, default models.
    - Needs investigation to determine how/if this would work with llama-stack configuration
- **Observability (later):** Usage dashboard by user/role, error rates, provider latency; export to cluster logging/metrics.

**Flows:**
1) Admin configures provider + credentials in Hub UI -> Hub backend writes/updates Secrets + ConfigMap(s) that are used by llama-stack.
2) Centralized Config includes proxy endpoint + model catalog + policy snapshot.
3) Proxy hot-reloads config (or is restarted by the operator) to apply changes.

---

## Drawbacks

- Adds one more service to the cluster footprint.
- Depends on Keycloak uptime for auth validation.
- Initial duplication of credentials with Solution Server (temporary).

---

## Graduation Criteria

- Deployed alongside Hub and Solution Server.
- Validates tokens against live Keycloak.
- Hub integration with centralized config for proxy endpoint exposure.
- Successful end-to-end inference with user JWT authentication.
