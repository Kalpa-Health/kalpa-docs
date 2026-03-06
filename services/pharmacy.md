# Service: Pharmacy

**Version:** 1.0
**Date:** 2026-03-06
**Status:** Active — extracted from Backbone (ADR-010)
**Maintained by:** Alex (Kalpa Inovasi Digital)

---

## 1. Overview

1.1 Pharmacy owns the pharmacy domain lifecycle for WellMed: prescription tracking, medication request coordination, dispense lifecycle, and pharmacy sale recording. It is a participant in Backbone sagas per ADR-005 — it receives RabbitMQ step triggers from Backbone and reports results via `SagaCallbackService`. Pharmacy never initiates a saga.

1.2 Gateway calls Backbone only. Backbone proxies pharmacy-bound gRPC calls to this service. There is no direct gateway→pharmacy routing.

1.3 Item catalog data (prices, dosage instructions) arrives embedded in saga trigger payloads from Backbone — pharmacy never calls Backbone's `ItemService` directly (ADR-006).

---

## 2. Repository

| Item | Value |
|------|-------|
| **Repo** | `wellmed-pharmacy` |
| **Language** | Go 1.25+ |
| **gRPC address** | `PHARMACY_GRPC_ADDRESS` (env on backbone) |
| **gRPC port** | `:50054` |
| **Module** | `github.com/kalpa-health/wellmed-pharmacy` |

---

## 3. Key Responsibilities

- **Prescription tracking** — records prescriptions as received from consultation via saga trigger, stores enriched item data (catalog embedded by backbone)
- **Medication request coordination** — stubs for Phase 1; coordinates `CONSULTATION_TO_PHARMACY` and `PHARMACY_TO_CONSULTATION` request directions
- **Dispense lifecycle** — stub for Phase 1; confirms physical dispensing of prescription items
- **Pharmacy sale recording** — stub for Phase 1; records sale transactions after dispense
- **Saga participant** — receives step triggers from Backbone, reports results via `SagaCallbackService`. Does not orchestrate (ADR-005)
- **Multi-tenant database access** — one PostgreSQL database per tenant, per-service schema isolation

---

## 4. gRPC Interface Catalog

4.1 4 services registered at `:50054`:

| # | Service | Key Methods | Phase 1 Status |
|---|---------|-------------|----------------|
| 1 | PrescriptionService | GetAll, GetById, Create, Update, Finalize | Stubs only (saga handler writes records) |
| 2 | MedicationRequestService | GetAll, GetById, Create, Approve, Reject | Create stub |
| 3 | DispenseService | GetAll, GetById, Create, Confirm | All stubs |
| 4 | PharmacySaleService | GetAll, GetById, ProcessSale | All stubs |

4.2 All business writes flow through saga step handlers, not direct gRPC calls. Phase 1 gRPC handlers are scaffolded stubs.

---

## 5. Multi-Tenant Design

5.1 Identical to Backbone and Consultation patterns. Each tenant has a dedicated PostgreSQL database; per-year schema partitioning.

5.2 Tenant identity flows from JWT claims (`tenant_db`, `tenant_schema`) → `ConnectionManager` → `SET LOCAL search_path`.

---

## 6. Saga Participation

6.1 **Role: PARTICIPANT** — Backbone is the sole orchestrator (ADR-005). Pharmacy receives step triggers via the `pharmacy.steps` RabbitMQ queue.

6.2 **Routing keys handled:**

| Routing Key | Handler | Phase 1 Status |
|-------------|---------|----------------|
| `saga.prescription.create.recordprescriptioninpharmacy.trigger` | `CreatePrescriptionHandler.Handle()` | Wired — writes to DB + callbacks |
| `saga.prescription.create.recordprescriptioninpharmacy.compensate` | `CreatePrescriptionHandler.Compensate()` | Stub |
| `saga.pharmacy_sale.process.trigger` | `PharmacySaleHandler.Handle()` | Stub — logs `dispense_requires_payment` |
| `saga.pharmacy_sale.process.compensate` | `PharmacySaleHandler.Compensate()` | Stub |
| `saga.pharmacist_visit.atomic.create.trigger` | `AtomicPharmacistVisitHandler.Handle()` | No-op (always succeeds) |
| `saga.pharmacist_visit.atomic.create.compensate` | `AtomicPharmacistVisitHandler.Compensate()` | No-op |

6.3 **`AtomicPharmacistVisit` design:** Always-succeeds saga. Backbone writes the canonical pharmacist visit record locally (step 1), then notifies pharmacy (step 2). Pharmacy's handler is intentionally a no-op — it acknowledges receipt only.

6.4 **`DispenseRequiresPayment`:** Boolean embedded in the `pharmacy_sale.process` trigger payload by backbone. Controls whether pharmacy waits for POS payment confirmation before dispensing. Pharmacy reads from payload — never calls backbone for this value.

---

## 7. Data Ownership

| Domain | Pharmacy Creates and Owns | Backbone Provides IN Trigger |
|--------|--------------------------|------------------------------|
| Prescription | `pharmacy_prescriptions` rows | `visit_examination_id`, `patient_id`, enriched items (catalog) |
| Medication Request | `pharmacy_medication_requests` rows | direction, reference IDs |
| Dispense | `pharmacy_dispenses` rows | prescription IDs, dispense instructions |
| Pharmacy Sale | `pharmacy_sales` rows | `dispense_requires_payment`, consumer ID |

---

## 8. Auth Architecture

8.1 JWT propagated from Gateway via Backbone via gRPC metadata. Token carries full tenant context.

8.2 Service-to-service auth: `BACKBONE_SERVICE_KEY` must match `BACKBONE_API_KEY_PHARMACY` on backbone (ADR-007).

8.3 For saga callbacks: pharmacy uses `BACKBONE_GRPC_ADDRESS` to call backbone's `SagaCallbackService`.

---

## 9. Database Schema

9.1 Tables (all in tenant schema):

| Table | Purpose |
|-------|---------|
| `pharmacy_prescriptions` | Pharmacy-side prescription records (received from consultation) |
| `pharmacy_medication_requests` | Medication request tracking |
| `pharmacy_dispenses` | Dispense lifecycle records |
| `pharmacy_sales` | Pharmacy sale records |

9.2 Migration: `internal/db/migration/sql/001_pharmacy_tables.sql`

---

## 10. What Is and Isn't Implemented (Phase 1)

| Feature | Status |
|---------|--------|
| Service scaffold + gRPC server | ✅ Complete |
| Multi-tenant DB pattern | ✅ Complete |
| Saga step consumer | ✅ Wired |
| `CreatePrescriptionHandler` | ✅ Wired — writes DB, calls saga callback |
| `PharmacySaleHandler` | Stub — logs payload |
| `AtomicPharmacistVisitHandler` | No-op by design |
| Dispense confirmation logic | Stub |
| `ConsumentPatientClient` | Stub (Phase 2) |
| Direct gRPC service handlers | Stubs |

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-06 | Alex + Claude | Initial service doc — pharmacy extraction Phase 1 complete (ADR-010) |
