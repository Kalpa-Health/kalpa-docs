# WellMed — Gateway Architectural Questions & Anomalies

Observations from codebase analysis that deviate from best practice or need clarification.
Discovered during: Gateway Documentation & Testing Plan (Feb 2026).

**Status key**: ❓ Open | ✅ Answered | 🔧 Fix planned | ⚠️ Risk flagged

---

## 1. Security & Authentication

### 1.1 Two JWT secrets with unclear separation
**Status**: ❓ Open
**File**: `internal/domain/user/service/user.go:48`, `internal/config/env.go`, `internal/middleware/jwt.go`
**Observation**: The gateway uses two different JWT secrets:
- `JWT_SECRET` — used in `user.Login` to **decode the inbound credential token** (the one the frontend sends with `{username, password}` embedded)
- `JWT_SECRET_KEY` — used in `authenticate.go` to **validate session tokens** on every protected request

**Risk**: If these two values are accidentally swapped in the `.env`, authentication silently breaks in hard-to-diagnose ways. The variable names (`JWT_SECRET` vs `JWT_SECRET_KEY`) don't communicate their distinct purposes.

**Recommendation**: Rename to `JWT_CREDENTIAL_TOKEN_SECRET` and `JWT_SESSION_TOKEN_SECRET`, or at minimum add a comment block in `config/env.go` explaining the two-secret design.

**Context for devs**: Is this two-secret design intentional? Who issues the inbound credential JWT — the frontend itself, or another service?

---

### 1.2 JWT claims validated but not forwarded downstream
**Status**: ❓ Open — confirming with devs
**File**: `internal/middleware/authenticate.go:44-67`
**Observation**: The `Authenticate` middleware extracts and validates four claims (`user_id`, `tenant_id`, `tenant_db`, `tenant_schema`), confirms they're present, then **discards them**. They are never stored in `c.Locals()`. The backbone receives tenant context only because the full `Authorization` header is forwarded via gRPC metadata.

**Risk**: Any gateway-level logic that needs to know the current user or tenant (e.g., audit logging, per-tenant rate limiting, future gateway-level validation) must re-parse the JWT from `c.Locals("grpcCtx")` — which is indirect and fragile.

**Best practice**: Store extracted claims in Fiber locals:
```go
c.Locals("user_id", userID)
c.Locals("tenant_id", tenantId)
```

**Impact on future work**: When the gateway starts calling external APIs (SATU SEHAT, P-Care), those calls will need tenant context at the gateway level. This pattern will need to change.

---

### 1.3 No timeout on user.Login gRPC call
**Status**: ⚠️ Risk flagged
**File**: `internal/domain/user/service/user.go`
**Observation**: Every other service method creates a `context.WithTimeout(ctx, 3*time.Second)` before calling the RPC client. `user.Login` does not — it passes `ctx` directly. The only protection is the global `RPC_TIMEOUT` interceptor (default 5s).

**Risk**: A slow backbone response on login holds the connection open for up to 5 seconds (vs 3s for all other calls). On a busy server with the 4 req/min rate limiter, this is unlikely to cause problems now, but inconsistent.

**Recommendation**: Add `cctx, cancel := context.WithTimeout(ctx, 3*time.Second)` before the `s.client.Login(ctx, req)` call, consistent with all other service methods.

---

## 2. Code Quality

### 2.1 Module path references old personal GitHub account
**Status**: 🔧 Fix planned
**File**: `go.mod` line 1
**Observation**: `module github.com/ingarso/kalpa-gateway` — the module path still points to `ingarso`'s personal account. The repo now lives under `Kalpa-Health` org.

**Impact**: All internal imports use `github.com/ingarso/kalpa-gateway/...`. This works fine since Go module paths are just strings, but it's misleading and will cause confusion for new team members.

**Recommendation**: Update `go.mod` to `module github.com/kalpa-health/kalpa-gateway` and run a global find-replace on all import paths. Should be done before the team onboards new developers.

---

### 2.2 `gorm` imported but not used
**Status**: ❓ Open
**File**: `go.mod`
**Observation**: `gorm.io/gorm v1.31.0` is in `go.mod` but the gateway never imports or uses it. The gateway has no direct database access — all data flows through gRPC to the backbone.

**Risk**: Dead dependency — increases binary size and attack surface. Any gorm vulnerability requires a gateway update even though gorm isn't used.

**Recommendation**: Run `go mod tidy` to remove it, unless it's intentionally retained for a planned future feature (gateway-level DB access).

**Question for devs**: Is there a plan to add direct DB access to the gateway? If not, remove gorm.

---

### 2.3 `conn *grpc.ClientConn` field declared but never assigned in RPC structs
**Status**: ⚠️ Risk flagged
**Files**: All 39 files in `internal/rpc/` (e.g., `rpc/patient.go`, `rpc/transaction.go`)
**Observation**: Every `*Rpc` struct has a `conn *grpc.ClientConn` field and a `Close()` method that checks `if c.conn != nil`. However, the `conn` field is **never assigned** — only the `cli` (generated gRPC client) is set via `m.Conn()`. So `Close()` on any RPC struct is always a no-op.

**Risk**: Low (the manager's `Close()` properly closes the shared connection). But if anyone calls `TransactionRpc.Close()` expecting to clean up, it silently does nothing. The dead field adds confusion.

**Recommendation**: Either assign `conn: m.Conn()` in each constructor, or remove the `conn` field and `Close()` method from all RPC structs and rely solely on `grpcx.Manager.Close()`.

---

### 2.4 Inconsistent logging in user service
**Status**: 🔧 Fix planned
**File**: `internal/domain/user/service/user.go:48`
**Observation**: Uses `fmt.Printf("Error decoding token: %v\n", err)` instead of the structured logger (`logrus`) used everywhere else. This log will not appear in structured JSON log output and cannot be filtered or sampled.

**Recommendation**: Replace with `log.WithError(err).Error("failed to decode credential token")`.

---

### 2.5 `generateTokenID` function defined but never called
**Status**: 🔧 Fix planned
**File**: `internal/middleware/jwt.go:66`
**Observation**: `func generateTokenID(uuid string) string` is defined but never referenced. Go's compiler doesn't flag unused functions (only unused variables/imports), so this slipped through.

**Recommendation**: Delete this function unless it's needed for an upcoming feature.

---

## 3. Architecture — Future Readiness

### 3.1 Gateway architecture needs extension for external API calls
**Status**: ❓ Planning required
**Context**: The gateway is planned to call external APIs programmatically (SATU SEHAT, P-Care, etc.) in addition to serving the frontend/BFF.

**Current state**: The gateway is a pure HTTP-to-gRPC proxy. All external integrations currently live in the backbone. The gateway has no outbound HTTP client infrastructure.

**Implications of adding external API calls at the gateway level**:

1. **Tenant context gap** (linked to Q1.2): External API calls to SATU SEHAT need to be scoped to the correct tenant/clinic. Currently tenant context is only available via the forwarded JWT — not extracted into usable locals. This needs fixing before adding outbound calls.

2. **Auth token management**: SATU SEHAT uses OAuth2 client credentials (per-org tokens, 1-hour expiry). The gateway will need a token cache layer (Redis is already available) separate from the user session cache.

3. **Service-to-service auth**: When the gateway calls external APIs on behalf of a request, it needs different credentials than the user's JWT. The current single-credential-per-request model doesn't cover this.

4. **Consider: gateway vs backbone for external calls**: There are two valid architectures:
   - **Gateway calls SATU SEHAT directly**: Lower latency, simpler for simple proxying, but gateway becomes more complex
   - **Gateway delegates to backbone via gRPC**: Backbone already has SATU SEHAT integration logic (confirmed from WellMed backbone codebase); consistent with current architecture; backbone handles retry/queue logic

   **Recommendation**: Keep external integrations in the backbone. Have the gateway proxy those calls via gRPC the same way it proxies all other calls. This keeps the gateway thin and avoids duplicating integration logic.

5. **Rate limiting**: SATU SEHAT has strict per-org rate limits. If the gateway calls SATU SEHAT directly, the rate limiting logic must be tenant-aware. The backbone already handles this.

---

## 4. Outstanding Questions for Dev Team

| # | Question | Urgency |
|---|---|---|
| 4.1 | Q1.1: Is the two-JWT-secret design intentional? Who issues the credential token? | High |
| 4.2 | Q1.2: Should middleware claims be stored in Fiber locals? | Medium |
| 4.3 | Q2.2: Is there a plan to use gorm in the gateway, or should it be removed? | Low |
| 4.4 | Q3.1: Should future SATU SEHAT calls go through the gateway or stay in the backbone? | High — before planning integration work |
| 4.5 | **Backbone decomposition & service routing**: Backbone currently owns 37 domain modules covering EMR, Cashier, and Pharmacy workflows — everything routes through a single `BACKBONE_GRPC_ADDRESS`. Target architecture splits these into separate gRPC services. Three decisions block this work: (1) which modules move to EMR vs. stay in Backbone, (2) who owns saga orchestration after the split, (3) how gateway routing changes. See standalone section below. | High — blocks all service decomposition planning |

---

### 4.5 Backbone Decomposition & Service Routing
**Status**: ❓ Open — team decision required
**Context**: `wellmed-backbone/internal/domain/` — all 37 modules

**Observation**: Backbone currently owns all clinic domain logic across 37 modules in a single binary. The system architecture spec (§2.3.2) documents this as the known current state — the other services haven't been split out yet. The internal structure is already correct: every module has independent `handler/`, `service/`, `repository/`, and `dto/` layers. The PostgreSQL schema is also pre-partitioned (`emr`, `cashier`, `pharmacy`, `backbone` schemas per tenant DB). The seams exist; the decomposition path is extracting modules into new binaries, not untangling a ball of mud.

**Module mapping (current → target):**

| Target Service | Backbone Modules |
|----------------|-----------------|
| **Stay in Backbone** | `tenant`, `user`, `employee`, `role`, `permission`, `menu`, `unicode`, `autolist`, `address`, `reference_service`, `consumer`, `card_identity`, `people` |
| **→ EMR** | `patient`, `visit_patient`, `visit_registration`, `visit_registration_referral`, `visit_examination`, `assessment`, `treatment`, `examination_treatment`, `medical_service_treatment`, `referral`, `practitioner_evaluation`, `frontline`, `data_sync`, `family_relationship` |
| **→ Cashier** | `billing`, `invoice`, `transaction`, `item`, `item_rent`, `service_price`, `card_stock` |
| **→ Pharmacy** | `pharmacy`, `pharmacy_sale`, `medicine` |

**Three decisions that block decomposition:**

1. **Which modules route through Backbone vs. get their own gRPC address?** EMR is the obvious first split. Does every EMR call go `Gateway → EMR` directly, or do some workflows (e.g., visit creation triggering a billing record) still call through Backbone for saga coordination?

2. **Saga ownership after the split.** The saga framework lives in Backbone. A visit creation saga currently spans patient, visit, pharmacy_sale, and billing modules — all within one process. After the split, who orchestrates cross-service sagas? Options: (a) EMR calls Cashier directly via gRPC within its own saga steps, (b) Backbone retains a thin saga-coordinator role and individual services implement their domain logic, (c) async via RabbitMQ for cross-service steps.

3. **Gateway routing impact.** Currently one env var: `BACKBONE_GRPC_ADDRESS`. After each split: add `EMR_GRPC_ADDRESS`, `CASHIER_GRPC_ADDRESS`, etc. Each requires a new RPC client struct in the gateway and updated module routing. This is straightforward gateway work — but needs to be sequenced with the backbone split, not done independently.

**Recommendation**: Decide (2) first — saga architecture determines everything else. EMR is the right first split candidate: largest chunk, most domain-specific, and already a named Lite-tier service in the product spec.

---

*Last updated: 2026-03-02 | Source: Gateway codebase analysis (Plan v2.0), System architecture doc v1.1, backbone codebase analysis*
