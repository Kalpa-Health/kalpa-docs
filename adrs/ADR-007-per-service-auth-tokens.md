# ADR-007: Per-Service Identity Tokens for Inter-Service Authentication

**Status:** Accepted
**Date:** 2026-03-05
**Author:** Alex
**Reviewers:** Hamzah (CTO)

---

## 1. Context

1.1 Gateway and Consultation both call Backbone's gRPC endpoints. Both currently authenticate using a single shared credential (the "backbone token") injected via environment variable.

1.2 A single shared token means that any rotation event — scheduled, forced by a security incident, or compliance-driven — requires redeploying all client services simultaneously. As the number of services calling Backbone grows (pharmacy, cashier, scheduler), this blast radius grows with it.

1.3 Additionally, Backbone's access logs cannot distinguish which service made a given call. All callers appear identical, preventing per-service audit trails or per-service permission scoping.

1.4 The "Backbone is server, MSes are clients" framing does not justify shared credentials. Standard practice for server-to-server auth is that the server (Backbone) issues and validates distinct credentials per client — the same principle as AWS IAM roles or OAuth2 client credentials, one client_id per service.

---

## 2. Decision

2.1 **Each service that calls Backbone is assigned its own credential.** Backbone validates the credential and maps it to a named caller identity (e.g., `gateway`, `consultation`).

2.2 **Token rotation is per-service.** Rotating gateway's token does not require redeploying consultation, and vice versa.

2.3 **Backbone logs include the caller identity** on every inbound gRPC request, enabling per-service audit trails and incident scoping.

2.4 **All permissions remain equivalent for now.** The initial implementation gives all callers the same access level. Per-service authorization scoping is deferred until a concrete need emerges (see Section 5).

---

## 3. Consequences

### 3.1 Positive

- Staged rotation: any single service's token can be rotated independently with zero impact on other services.
- Audit trail: Backbone logs identify the calling service, enabling targeted incident response.
- Least privilege foundation: per-service identity is the prerequisite for per-service authorization if needed later.
- Scales cleanly: every new service (pharmacy, cashier, scheduler) gets its own credential at bootstrap with no changes to existing services.

### 3.2 Negative

- Slightly higher operational overhead: N tokens to manage instead of 1.
- SSM parameter manifest grows by one entry per service per environment.

### 3.3 Risks

- **Token sprawl:** Mitigated by the existing SSM parameter manifest tooling (`wellmed-infrastructure`). Each service's token lives in a predictable path: `/wellmed/{env}/{service}/BACKBONE_API_KEY`.
- **Missed rotation:** If a rotation SOP isn't documented, individual tokens may be forgotten. Mitigated by listing all service tokens in the SSM manifest and rotating them as a group during scheduled rotation cycles — even though they can be rotated independently, the standard cycle still covers all of them.

---

## 4. Alternatives Considered

| Alternative | Pros | Cons | Why Not Chosen |
|-------------|------|------|----------------|
| Shared token (current) | Simple, one secret to manage | Simultaneous rotation blast radius; no caller identity in logs | Unacceptable as service count grows |
| mTLS per service | Strong mutual authentication, no shared secret | Significant PKI complexity, cert rotation tooling needed | Correct long-term direction but too heavy for current team size; revisit when adding external-facing services |
| OAuth2 client credentials (external IdP) | Industry standard, fine-grained scopes | Requires IdP infrastructure (Keycloak, Auth0, etc.) | Overhead not justified at current scale; viable path if Padma Medical Group integration requires it |

---

## 5. Implementation Notes

5.1 **Backbone:** Accept a map of `{token → service_name}` loaded from environment at startup. Reject requests with an unrecognised token. Log `caller=<service_name>` on every inbound gRPC request via interceptor.

5.2 **Per-service env vars:** Each service carries its own `BACKBONE_API_KEY`. The backbone carries one env var per client: `BACKBONE_API_KEY_GATEWAY`, `BACKBONE_API_KEY_CONSULTATION`, etc.

5.3 **SSM manifest:** Add one SSM parameter per service per environment to `wellmed-infrastructure`. Naming convention: `/wellmed/{env}/{service}/BACKBONE_API_KEY`.

5.4 **Authorization scoping (deferred):** The caller identity is available in context for future use. Do not implement per-service permissions until a concrete requirement exists — avoid over-engineering the auth layer.

5.5 **Migration path:** Replace the shared `BACKBONE_API_KEY` in gateway and consultation `.env` / SSM with service-specific values. Backbone's validation map replaces the single-token check. Zero downtime if done as a rolling deploy (add new keys first, remove old shared key after all services are updated).

---

## 6. References

- ADR-006: Domain Service Boundaries (establishes which services call Backbone)
- `wellmed-infrastructure/ssm-manifests/` — SSM parameter manifest tooling
- AWS IAM principle of least privilege (analogous pattern)

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-05 | Alex | Initial ADR — per-service auth tokens decision |
