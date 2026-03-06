# ADR-010: Pharmacy Microservice Extraction

**Status:** Accepted
**Date:** 2026-03-06
**Author:** Alex
**Reviewers:** Alex

---

## 1. Context

1.1 WellMed's pharmacy domain (prescription tracking, medication requests, dispense lifecycle, pharmacy sales) initially lived inside `wellmed-backbone`. As the system grew, this created tight coupling between the orchestration layer and pharmacy business logic, making independent deployment, testing, and scaling impossible.

1.2 The consultation extraction (ADR-002) demonstrated that domain services can be cleanly separated from backbone while backbone remains the sole saga orchestrator. Pharmacy is the second candidate for this pattern.

1.3 Key forces driving the decision:
- Pharmacy has its own database schema, deployment cadence, and team scope
- Pharmacy sale, dispense, and prescription flows span multiple services — backbone must orchestrate these without owning the business logic
- The item catalog (prices, dosage instructions) is canonical in backbone; pharmacy must receive this data in step payloads rather than fetching it mid-request (ADR-006)
- Indonesian clinics have variable payment workflows: some require payment before dispense (`dispense_requires_payment`), others dispense first

---

## 2. Decision

2.1 Extract pharmacy domain to a standalone microservice `wellmed-pharmacy` at port `:50054`, following the consultation extraction pattern (ADR-002).

2.2 **Proxy pattern:** Gateway calls Backbone only. Backbone proxies pharmacy-bound gRPC calls to `wellmed-pharmacy` via `PHARMACY_GRPC_ADDRESS`. No direct gateway→pharmacy routing.

2.3 **Saga participant model:** Pharmacy is a saga participant per ADR-005. Backbone publishes step triggers to the `pharmacy.steps` RabbitMQ queue; pharmacy processes steps and calls back via `SagaCallbackService`. Pharmacy never initiates a saga.

2.4 **Item catalog delivery:** Backbone embeds enriched item catalog data (name, price, dosage instructions) into the saga trigger payload via a local `EnrichPrescriptionItems` step before publishing to pharmacy. Pharmacy reads from the payload; it never calls Backbone's `ItemService` directly (ADR-006).

2.5 **DispenseRequiresPayment:** A boolean tenant config field (`dispense_requires_payment`) on the `Tenant` entity in backbone controls saga step ordering for pharmacy sales. Backbone embeds this value in the `pharmacy_sale.process` trigger payload — pharmacy reads from payload, not from backbone at runtime.

2.6 **Atomic pharmacist visit:** When a pharmacist confirms dispense, an always-succeeds saga (`pharmacist_visit`) writes a canonical visit record with `signed_off_by_role: PHARMACIST`. This saga has no compensation path — the canonical visit record is written locally in backbone (step 1), then pharmacy is notified (step 2, always succeeds). This creates an audit trail of pharmacist-confirmed dispenses in the canonical archive.

2.7 **Shared proto contract:** `medication_request.proto` and `prescription.proto` are owned in backbone's `proto/` and distributed to both pharmacy and consultation. Both services implement these interfaces independently — consultation for doctor-side prescription creation, pharmacy for pharmacist-side tracking.

2.8 **FinalizePrescription placement:** The `FinalizePrescription` RPC lives on `FrontlineService` in consultation (not a new `PrescriptionService` proto), since the doctor finalises the prescription in the frontline/consultation UX context. It delegates to an injected `PrescriptionService` internally.

2.9 **Per-service auth:** Backbone validates pharmacy's inbound calls using `BACKBONE_API_KEY_PHARMACY` per ADR-007. This is separate from the gateway and consultation keys.

---

## 3. Consequences

### 3.1 Positive

- Independent deployment and scaling for pharmacy domain
- Clear domain ownership: pharmacy owns prescription tracking, dispense, and pharmacy sales; backbone owns orchestration and item catalog
- Tenant payment workflow flexibility via `dispense_requires_payment` without code changes
- Canonical pharmacist visit record enables SATU SEHAT reporting and audit without pharmacy being coupled to the canonical archive directly
- Parallel implementation pattern for `medication_request` reduces cross-service coupling

### 3.2 Negative

- Two more services to deploy, monitor, and maintain
- `FinalizePrescription` on `FrontlineService` is a slightly awkward placement — consultation's `PrescriptionService` is internal-only, not a first-class gRPC contract
- Step-order swap logic for `dispense_requires_payment` is a TODO in `PharmacySaleSagaBuilder` — ordering is hardcoded for Phase 1

### 3.3 Risks

- **Item catalog staleness:** Backbone enriches item data at saga time from its own cache. If a pharmacist holds a transaction open while prices change, the enriched payload reflects the price at saga start. Acceptable for Phase 1 — to be addressed in a future ADR on saga payload versioning.
- **Atomic pharmacist visit failure:** The canonical visit step is local-only (no external call). Failure would leave a canonical visit without a pharmacy confirmation notification. Mitigation: the `atomic.create` remote step always succeeds by design; pharmacy ignores failures and relies on the canonical record.

---

## 4. Alternatives Considered

| Alternative | Pros | Cons | Why Not Chosen |
|-------------|------|------|----------------|
| Keep pharmacy in backbone | No new service | Defeats domain separation; backbone grows indefinitely | Rejected — same reasoning as ADR-002 |
| Direct gateway→pharmacy routing | Lower latency | Breaks proxy pattern; requires gateway config changes per service | Rejected — proxy pattern avoids gateway change per extracted service |
| Pharmacy fetches item catalog at runtime | Always current pricing | Violates ADR-006; cross-service call mid-request; tight coupling | Rejected — ADR-006 |
| Separate `PrescriptionService` proto for consultation | Clean separation | Two separate protos for the same domain split; doubles proto maintenance | Rejected — `FrontlineService.FinalizePrescription` is simpler for Phase 1 |

---

## 5. Implementation Notes

5.1 **New service:** `github.com/kalpa-health/wellmed-pharmacy`, port `:50054`. Mirrors consultation's internal structure: `ConnectionManager`, `TransactionManager`, multi-tenant DB pattern, RabbitMQ `StepConsumer`.

5.2 **Backbone changes:**
- `internal/domain/prescription/saga/builder.go` — `PrescriptionCreateSagaBuilder` (registered as `"prescription.create"`)
- `internal/domain/pharmacy_sale/saga/builder.go` — updated with `Process` and `ProcessSaleTransaction` steps
- `internal/domain/pharmacist_visit/saga/builder.go` — `AtomicPharmacistVisitSagaBuilder` (registered as `"pharmacist_visit"`)
- `internal/schema/entity/tenant.go` — added `DispenseRequiresPayment bool`
- `proto/frontline.proto` — added `FinalizePrescription` RPC + `FrontlineFinalizeRequest` message
- `ssm/parameters/backbone.json` — added `BACKBONE_API_KEY_PHARMACY`, `PHARMACY_GRPC_ADDRESS`

5.3 **Consultation changes:**
- Added `PrescriptionService` and `MedicationRequestService` domains
- Added `FrontlineServer.FinalizePrescription` method
- Updated `application.go` to register `PrescriptionServiceServer` and `MedicationRequestServiceServer` (10 total gRPC services)
- Updated `frontlinepb` from recompiled proto

5.4 **Infrastructure:**
- `wellmed-infrastructure/ssm/parameters/pharmacy.json` — all env vars with placeholder values
- GitHub repo `kalpa-health/wellmed-pharmacy` created; CI workflows on `main`, `develop`, `staging` branches

5.5 **Phase 1 stubs:** Dispense and pharmacy sale service methods are stubs returning `Unimplemented`. Only `CreatePrescriptionHandler` writes real DB records. `PharmacySaleHandler` and `AtomicPharmacistVisitHandler` are shells.

---

## 6. References

- ADR-002: Consultation service extraction (reference pattern)
- ADR-005: Saga orchestration (backbone-only, RabbitMQ triggers, gRPC callbacks)
- ADR-006: Domain service boundaries (no cross-service calls mid-request)
- ADR-007: Per-service auth tokens
- ADR-009: Canonical archive pattern (pharmacist visit writes canonical record)
- `kalpa-docs/plans/pharmacy-extraction/pharmacy-extraction-PLAN.md`

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-06 | Alex + Claude | Initial ADR — pharmacy extraction Phase 1 complete |
