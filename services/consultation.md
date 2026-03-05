# Service: Consultation

**Version:** 1.0
**Date:** 05 March 2026
**Status:** Active — extracted from Backbone (ADR-002)
**Maintained by:** Alex (Kalpa Inovasi Digital)

---

## 1. Overview

1.1 Consultation owns the full clinical visit lifecycle for WellMed. It is the largest domain service, housing all clinical records from visit registration through doctor sign-off. It is a participant in Backbone sagas per ADR-005 — it receives RabbitMQ step triggers from Backbone and reports results via `SagaCallbackService`. The only permitted Consultation→Backbone call is `CanonicalVisitService` on doctor sign-off (ADR-006 §2.4).

1.2 The gateway (`wellmed-gateway-go`) routes clinical visit reads and single-service writes directly to Consultation. Cross-service write operations that span domain boundaries route to Backbone for saga orchestration.

1.3 For the full system context, see [`wellmed-system-architecture.md §2.1`](../wellmed-system-architecture.md).

---

## 2. Repository

| Item | Value |
|------|-------|
| **Repo** | `wellmed-consultation` |
| **Language** | Go 1.25+ |
| **gRPC address** | `CONSULTATION_GRPC_ADDRESS` (env on gateway) |
| **gRPC port** | `:50052` |
| **Module** | `github.com/Kalpa-Health/wellmed-consultation` |

---

## 3. Key Responsibilities

- **Clinical visit lifecycle ownership** — `visit_id` generated here, all clinical records owned here. No module duplicates these models (ADR-006 §2.1).
- **8 gRPC services covering full visit workflow** — from frontline registration through assessment, treatment, examination, referral, and doctor sign-off.
- **Saga participant** — receives step triggers from Backbone via RabbitMQ (Phase 2), reports results via `SagaCallbackService`. Does not orchestrate; Backbone is the sole orchestrator (ADR-005).
- **CanonicalVisitService client** — writes final visit record (diagnosis codes, treatment summary, sign_off_at) to Backbone on doctor sign-off. This is the only permitted Consultation→Backbone call (ADR-006 §2.4).
- **Multi-tenant database access** — one PostgreSQL database per tenant, per-service schema isolation (same `ConnectionManager` pattern as Backbone).

---

## 4. gRPC Interface Catalog

4.1 8 services registered at `:50052`:

### 4.1 gRPC Services

| # | Service | Key Methods |
|---|---------|-------------|
| 1 | AssessmentService | Index |
| 2 | FrontlineService | Store, Index |
| 3 | ReferralService | Store, Index, Show |
| 4 | TreatmentService | Store, Index |
| 5 | VisitExaminationService | Store, Index, Show |
| 6 | VisitPatientService | Index, Show, Store |
| 7 | VisitRegistrationService | Index, ShowDetail |
| 8 | VisitRegistrationReferralService | Store, Show, Index |

### 4.2 Sub-modules (no standalone gRPC — used internally)

| Sub-module | Purpose |
|------------|---------|
| `family_relationship` | Family relationship records for patient visits |
| `examination_treatment` | Links examinations to treatment records |
| `medical_service_treatment` | Medical service items on treatment plans |
| `practitioner_evaluation` | Doctor evaluation records on sign-off |

---

## 5. Multi-Tenant Design

5.1 Each tenant (clinic) has a dedicated PostgreSQL database. Data is further partitioned by year-based schema (`schema_2026`, `schema_2027`, etc.). This is identical to the Backbone pattern.

5.2 The `ConnectionManager` manages lazy connection pooling per database. On first use, a connection is opened and cached.

5.3 Tenant identity flows from JWT claims (`tenant_db`, `tenant_schema`) → `ConnectionManager` → `SET LOCAL search_path` on each query session.

5.4 DB and schema identifiers are validated before use as PostgreSQL identifiers to prevent SQL injection.

---

## 6. Saga Participation

6.1 **Role: PARTICIPANT** — Consultation is not an orchestrator. Backbone is the sole saga orchestrator (ADR-005). Consultation receives step triggers and reports results.

6.2 **Phase 1 (current):** RabbitMQ consumer scaffold is in `internal/queue/consumer.go`. All step handlers return `TODO` — the infrastructure is wired but not yet implemented.

6.3 **Phase 2 (pending):** Implement step handlers for the following saga steps:

| Step | Handler | Action |
|------|---------|--------|
| `create_visit_record` | `HandleCreateVisitRecord` | Create visit_patient record, generate visit_id (ULID) |
| `add_visit_examination` | `HandleAddVisitExamination` | Store visit_examination record |
| `store_assessment` | `HandleStoreAssessment` | Store assessment and ICD codes |
| `store_treatment` | `HandleStoreTreatment` | Store treatment records and prescribed items |
| `doctor_sign_off` | `HandleDoctorSignOff` | Store practitioner_evaluation, trigger CanonicalVisitService |

6.4 **SagaCallbackService client stub** at `internal/clients/saga_callback_client.go` — Phase 2. Used to call `Backbone.SagaCallbackService.ReportStepResult` after each step completes or fails.

---

## 7. CanonicalVisitService (Sign-off)

7.1 **Direction:** Consultation is CLIENT, Backbone is SERVER.

7.2 Called at doctor final sign-off only. Sends outcome data only — diagnosis codes, treatment summary, `sign_off_at`. Not a full record sync.

7.3 **Client stub** at `internal/clients/canonical_visit_client.go` — Phase 2.

7.4 ADR-006 §2.4: this is the **ONLY** permitted Consultation→Backbone call. No other direct gRPC calls from Consultation to Backbone are allowed.

---

## 8. Data Ownership

| Saga Step | Backbone Provides IN | Consultation Creates and Owns | Returns via Callback |
|-----------|---------------------|-------------------------------|---------------------|
| `create_visit_record` | `patient_id`, `employee_id`, `tenant` | `visit_id` (ULID), `visit_patient` record | `{visit_id}` |
| `visit_examination` | `visit_id`, exam type | `visit_examination` record | `{visit_examination_id}` |
| `assessment` | `visit_id`, clinical context | `assessment` record, ICD codes | `{assessment_id}` |
| `treatment` | `visit_id`, prescribed items | `treatment` records | `{treatment_id}` |
| `sign_off` | `visit_id` | final `practitioner_evaluation`, sign_off timestamp | triggers `WriteVisitRecord` to Backbone |

---

## 9. Auth Architecture

9.1 JWT propagated from Gateway via gRPC metadata — same flow as Backbone. Token carries full tenant context: `tenant_db`, `tenant_schema`, `tenant_id`, `user_id`, `employee_id`.

9.2 `BACKBONE_SERVICE_KEY` used for service-to-service auth when Consultation calls Backbone (`CanonicalVisitService`).

9.3 Whitelist (no auth required): none — all Consultation endpoints require authentication.

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 05 Mar 2026 | Alex + Claude | Initial service doc — extracted from Backbone per ADR-002 |
