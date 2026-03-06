# Service: Backbone

**Version:** 1.2
**Date:** 2026-03-05
**Status:** Active — core gRPC backend
**Maintained by:** Alex (Kalpa Inovasi Digital)

---

## 1. Overview

1.1 Backbone is the core internal gRPC backend for WellMed with three roles: (a) canonical domain model owner for `patient`, `user`, `employee`, and system-wide mastering data; (b) sole saga orchestrator for all cross-service write operations; (c) auth, tenant config, and feature flags. Business logic belongs to domain services — Backbone coordinates, it does not own business decisions.

1.2 The gateway (`wellmed-gateway-go`) routes reads directly to domain services (target state) and cross-service writes to Backbone for saga orchestration.

1.3 For the full system context, see [`wellmed-system-architecture.md §2.1`](../wellmed-system-architecture.md).

---

## 2. Repository

| Item | Value |
|------|-------|
| **Repo** | `wellmed-backbone` |
| **Language** | Go 1.25+ |
| **gRPC address** | `BACKBONE_GRPC_ADDRESS` (env on gateway) |
| **gRPC port** | `:50051` |
| **Module** | `github.com/ingarso/wellmed-backbone` |

---

## 3. Key Responsibilities

- **Canonical domain model ownership** — `patient`, `user`, `employee`, and system-wide mastering data. No module duplicates these models (ADR-006 §2.1).
- **Saga orchestration** — sole coordinator for all cross-service write operations. Composes step payloads, manages saga state (PostgreSQL audit + Redis cache), executes compensation on failure (ADR-005).
- **Canonical visit record reception** — receives FHIR-shaped visit outcome from Consultation on doctor sign-off. This is the record used for SATU SEHAT sync, closed invoices, and audit (ADR-002 §2.4).
- **Auth** — JWT issuance, validation, Redis session management.
- **Multi-tenant database access** — one PostgreSQL database per tenant, per-service schema isolation.
- **POS pass-through** — Transaction, Billing, Invoice services are saga-orchestrated downstream calls to `wellmed-pos`, not business logic Backbone owns.

---

## 4. gRPC Interface Catalog

4.1 14 services registered at `internal/app/application.go`. Post ADR-002 Phase 1 completion, these split into three categories:

### 4.1 Staying in Backbone

| # | Service | Key Methods |
|---|---------|-------------|
| 1 | UserService | Login, RefreshToken |
| 2 | UnicodeService | GetAll, GetByFlag |
| 3 | MenuService | GetAll |
| 4 | AutoListService | GetAll |
| 5 | RoleService | GetAll, Store |
| 6 | PatientService (canonical) | GetAll, GetById, Store |
| 7 | ItemService | GetAll, Store |
| 8 | PharmacySaleService | GetAll, GetById, Store |
| 9 | TransactionService | → POS pass-through |
| 10 | BillingService | → POS pass-through |
| 11 | InvoiceService | → POS pass-through |

### 4.2 Extracted to `wellmed-consultation` — Phase 1 Complete

These 10 services moved to `wellmed-consultation` at `:50052` as of 05–06 Mar 2026 (ADR-002 Phase 1, ADR-010). They are no longer registered in Backbone.

| # | Service | Now In |
|---|---------|--------|
| — | AssessmentService | `wellmed-consultation` `:50052` |
| — | TreatmentService | `wellmed-consultation` `:50052` |
| — | VisitPatientService | `wellmed-consultation` `:50052` |
| — | VisitRegistrationService | `wellmed-consultation` `:50052` |
| — | VisitRegistrationReferralService | `wellmed-consultation` `:50052` |
| — | VisitExaminationService | `wellmed-consultation` `:50052` |
| — | ReferralService | `wellmed-consultation` `:50052` |
| — | FrontlineService (incl. FinalizePrescription) | `wellmed-consultation` `:50052` |
| — | PrescriptionService | `wellmed-consultation` `:50052` |
| — | MedicationRequestService | `wellmed-consultation` `:50052` |

See [`services/consultation.md`](consultation.md) for the full gRPC interface catalog.

### 4.3 Extracted to `wellmed-pharmacy` — Phase 1 Complete

These 4 services are proxied from Backbone to `wellmed-pharmacy` at `:50054` as of 06 Mar 2026 (ADR-010). Gateway calls Backbone; Backbone proxies via `PHARMACY_GRPC_ADDRESS`.

| # | Service | Now In |
|---|---------|--------|
| — | PrescriptionService (pharmacy-side) | `wellmed-pharmacy` `:50054` |
| — | MedicationRequestService (pharmacy-side) | `wellmed-pharmacy` `:50054` |
| — | DispenseService | `wellmed-pharmacy` `:50054` |
| — | PharmacySaleService | `wellmed-pharmacy` `:50054` |

See [`services/pharmacy.md`](pharmacy.md) for the full gRPC interface catalog and Phase 1 stub status.

### 4.4 New Backbone Services

| # | Service | Purpose | ADR |
|---|---------|---------|-----|
| 20 | SagaCallbackService | ReportStepResult — standard callback for all modules | ADR-005 |
| 21 | CanonicalVisitService | WriteVisitRecord — receives sign-off from Consultation | ADR-002 |
| 22 | SagaStatusService | GetStatus — saga state for frontend polling | ADR-005 |

---

## 5. Multi-Tenant Design

5.1 Each tenant (clinic) has a dedicated PostgreSQL database. Data is further partitioned by year-based schema (`schema_2026`, `schema_2027`, etc.).

5.2 The `ConnectionManager` manages lazy connection pooling per database. On first use, a connection is opened and cached.

5.3 Tenant identity flows from JWT claims (`tenant_db`, `tenant_schema`) → `ConnectionManager` → `SET LOCAL search_path` on each query session.

5.4 DB and schema identifiers are validated before use as PostgreSQL identifiers to prevent SQL injection.

---

## 6. Saga Framework

6.1 Backbone uses a single non-blocking saga pattern (`internal/saga/`) for all cross-service write operations (ADR-005). There are no blocking execution modes.

6.2 **Pattern:** Backbone publishes a RabbitMQ message to trigger each step. The receiving module processes the step and responds via gRPC callback (`SagaCallbackService.ReportStepResult`).

6.3 **Callback envelope:**

| Field | Type | Purpose |
|-------|------|---------|
| `saga_id` | string | Saga instance identifier |
| `step_id` | string | Step identifier within saga |
| `status` | enum | COMPLETED / FAILED / REJECTED |
| `payload` | bytes | JSON-encoded step result |
| `error_code` | string | Module-specific error code (e.g., `POS_INSUFFICIENT_STOCK`) |
| `idempotency_key` | string | Prevents duplicate processing |
| `tenant` | string | Tenant identifier for DB routing |

6.4 **State machine:** `pending → in_flight → completed / failed / compensated`. PostgreSQL saga audit table is the source of truth. Redis provides fast state cache for in-flight sagas and the `SagaStatusService` polling endpoint.

6.5 **Compensation:** On step failure or timeout, Backbone executes compensation in reverse order for completed steps. Compensation steps use the `compensation` retry profile (2s base, 10 retries, ~3min max).

6.6 **REJECTED vs FAILED:** A `REJECTED` callback means the payload was malformed (infra error) — Backbone retries without compensation. A `FAILED` callback means a business rule rejection — Backbone triggers the compensation sequence.

6.7 **Rule:** Backbone must never absorb business logic from modules. It composes payloads, coordinates steps, and manages state. Business decisions live in the domain modules.

6.8 **RabbitMQ convention:** Exchange `wellmed.saga`. Routing key pattern: `saga.{saga_type}.{step_name}.{trigger|compensate}`.

6.9 Full framework documentation: [`wellmed-backbone/internal/saga/README.md`](../../wellmed-backbone/internal/saga/README.md)

---

## 7. Auth Architecture

7.1 JWT with HMAC-SHA256, stored in Redis with JTI per session (single-session model).

7.2 Whitelist endpoints: `/user.UserService/Login`, `/user.UserService/RefreshToken`.

7.3 Token carries full tenant context: `tenant_db`, `tenant_schema`, `tenant_id`, `user_id`, `employee_id`.

7.4 Session invalidation: login replaces Redis JTI, invalidating all previous tokens for that user.

---

## 8. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Database-per-tenant | Strong isolation; tenant data never co-mingled |
| ULID for all IDs | Lexicographically sortable, globally unique, no UUID collision risk |
| JSONB `props` column | Flexible properties without schema migrations for non-indexed data |
| Saga framework | Compensating transactions across gRPC boundaries without 2PC |
| No HTTP in backbone | gRPC is the only interface — protocol enforced at gateway |

---

## 9. Pharmacy Proxy Pattern

9.1 Backbone proxies gateway pharmacy calls to `wellmed-pharmacy` via `PHARMACY_GRPC_ADDRESS`. No direct gateway→pharmacy routing exists (ADR-010 §2.2).

9.2 Backbone enriches item catalog data before publishing saga triggers to pharmacy — the `EnrichPrescriptionItems` local step embeds catalog data into the saga context; pharmacy reads from the trigger payload and never calls `ItemService` (ADR-006).

9.3 **New saga types registered in backbone for pharmacy:**

| Saga Type | Saga Builder | Steps |
|-----------|-------------|-------|
| `prescription.create` | `PrescriptionCreateSagaBuilder` | 1. Local `EnrichPrescriptionItems` → 2. Remote `RecordPrescriptionInPharmacy` |
| `pharmacy_sale` | `PharmacySaleSagaBuilder` | existing steps + Remote `Process` (pharmacy) + Remote `ProcessSaleTransaction` (POS stub) |
| `pharmacist_visit` | `AtomicPharmacistVisitSagaBuilder` | 1. Local `CreateCanonicalPharmacistVisit` → 2. Remote `atomic.create` (pharmacy, always succeeds) |

9.4 **DispenseRequiresPayment:** Tenant config boolean embedded in `pharmacy_sale.process` trigger payload. Backbone reads from `tenant.dispense_requires_payment`; pharmacy reads from payload. TODO: step order swap in `PharmacySaleSagaBuilder` when `dispense_requires_payment=true`.

9.5 **Auth:** `BACKBONE_API_KEY_PHARMACY` validates inbound pharmacy→backbone gRPC calls (ADR-007).

---

## 10. Interrelations with Consultation

10.1 Backbone and `wellmed-consultation` have a carefully bounded relationship. The interactions are non-obvious — this section exists to prevent accidental ADR-006 violations.

**The only permitted runtime calls:**

| Direction | Call | When | ADR |
|-----------|------|------|-----|
| Backbone → Consultation | RabbitMQ trigger (`wellmed.saga` exchange) | Each saga step Consultation must execute | ADR-005 |
| Consultation → Backbone | gRPC `CanonicalVisitService.WriteVisitRecord` | Doctor sign-off only | ADR-006 §2.4 |

No other calls are permitted. Backbone never calls Consultation's gRPC services. Consultation never calls Backbone's patient/employee/item services directly.

10.2 **How data flows from Backbone into Consultation:** Backbone composes the full step payload into the RabbitMQ trigger. Consultation reads from that payload and never fetches from Backbone at runtime. If a step needs `patient_name`, Backbone includes it in the trigger.

10.3 **How `visit_id` flows after visit creation:** Consultation creates the visit (owns the ULID), calls back with `{visit_id}` in the saga callback payload. Backbone stores `visit_id` in `SagaContext` and threads it into subsequent step payloads (POS billing, pharmacy, sign-off).

10.4 **Consultation source code still in Backbone during transition:** Modules `visit_patient`, `visit_examination`, `assessment`, `treatment`, `frontline`, `referral`, `family_relationship` exist in both repos. Backbone's copy is used only by the `patient` saga builder (new-patient registration flow creates a patient + first visit in one saga). These will be removed from Backbone once the `patient` saga builder is refactored to use a Remote `create_visit` step. See `application.go` lines 13–17.

10.5 **Wiring status as of 05 Mar 2026:**

| Item | Status |
|------|--------|
| `canonical_visit.proto` — Go generated | ✅ `canonicalvisitpb/*.pb.go` exist |
| `CanonicalVisitService` — handler registered | ✅ registered (stub — returns Unimplemented) |
| `saga_callback.proto` — Go generated | ❌ `sagacallbackpb/` is empty |
| `saga_status.proto` — Go generated | ❌ `sagastatuspb/` is empty |
| `SagaCallbackService` — handler registered | ❌ waiting on protoc |
| `SagaStatusService` — handler registered | ❌ waiting on protoc |
| `CreateVisit` saga builder | ❌ not yet written |

Full wiring plan: `kalpa-docs/plans/visit-saga-wiring/visit-saga-wiring-PLAN.md`

---

## 11. Service-Specific Documentation

| Document | Location |
|----------|---------|
| Directory structure | `wellmed-backbone/docs/backbone-structure.md` |
| Domain modules | `wellmed-backbone/docs/backbone-modules.md` |
| Infrastructure (saga, DB, Redis) | `wellmed-backbone/docs/backbone-infrastructure.md` |
| Middleware & auth | `wellmed-backbone/docs/backbone-middleware.md` |
| Test baseline | `wellmed-backbone/docs/backbone-test-baseline.md` |
| Service docs | `wellmed-backbone/docs/services/` |
| Saga framework | `wellmed-backbone/internal/saga/README.md` |

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 01 Mar 2026 | Alex + Claude | Initial stub |
| 1.0 | 02 Mar 2026 | Alex + Claude | Full service doc — all sections complete |
| 1.1 | 05 Mar 2026 | Alex + Claude | BB-1 through BB-4: Updated Backbone role to three-role model (canonical model owner, saga orchestrator, auth/config); updated §3 responsibilities to remove stale sync/business-logic framing; split §4 gRPC catalog into three subsections (staying, extracting to consultation, new saga services); replaced §6 saga framework with ADR-005-aligned async-only description including callback envelope, REJECTED/FAILED distinction, and RabbitMQ convention. |
| 1.2 | 05 Mar 2026 | Alex + Claude | §4.2 updated: 8 consultation services extracted to wellmed-consultation (ADR-002 Phase 1 complete). §4.1 count updated 22→14. |
| 1.3 | 05 Mar 2026 | Alex + Claude | Added §9 Interrelations with Consultation — boundary rules, data flow patterns, visit_id threading, transition-period duplicate source note, wiring status table. References visit-saga-wiring plan. |
| 1.4 | 06 Mar 2026 | Alex + Claude | Added §9 Pharmacy Proxy Pattern — new saga types (prescription.create, pharmacy_sale updates, pharmacist_visit), dispense_requires_payment, PHARMACY_GRPC_ADDRESS, ADR-010 auth. Added §4.3 pharmacy proxy services table. Updated §4.2 to 10 consultation services. Renumbered §9–§11. |
