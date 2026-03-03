# ADR-002: Consultation Service Extraction from Backbone

**Status:** Accepted
**Date:** 03 March 2026
**Author:** Alex
**Reviewers:** Hamzah (CTO), Abdul (Lead Backend)

---

## 1. Context

1.1 Backbone is a structured monolith with 37 domain modules in one binary. Each module follows the correct layered pattern (handler → service → repository → dto) and the PostgreSQL schema is already partitioned per service (emr, cashier, pharmacy, backbone schemas). The data separation exists. The deployment separation does not.

1.2 Fourteen of those 37 modules belong to the clinical consultation domain — the visit lifecycle from registration through doctor sign-off. These are listed in the product specification as a distinct Lite-tier service ("EMR") but physically live inside Backbone.

1.3 The decomposition decision was deferred until the CI/CD and documentation infrastructure was established (completed March 2026). Two pressures made the decision urgent:
- The lead backend developer is actively completing the Consultation→Frontend connection — architecture must be decided before this work ships.
- The codebase is at risk of accumulating new clinical logic inside Backbone with no explicit boundary to prevent drift.

1.4 The visit creation saga is the most complex event in the system. It crosses multiple service domains: initial service line items push immediately to POS (Cashier), a pharmacy hook fires for prescriptions, and further services can be added mid-visit — each updating billing and inventory. Cross-service saga compensation (e.g., cancellation of a visit that has already pushed to POS) is a real requirement. This complexity is acknowledged and is handled in Phase 2 of the extraction plan. Phase 1 delivers the structural separation; Phase 2 delivers full saga wiring.

1.5 Three options were evaluated. See §4.

---

## 2. Decision

2.1 Extract Consultation from Backbone as a standalone microservice (`wellmed-consultation`). This is a complete extraction — no temporary co-location in Backbone. The boundary is defined and sealed now, before new clinical features are added.

2.2 **Consultation owns these 13 modules** (moved from Backbone):

| Module | Responsibility |
|--------|---------------|
| `visit_patient` | Visit record, status lifecycle |
| `visit_registration` | Patient registration for a visit |
| `visit_registration_referral` | Referral at registration |
| `visit_examination` | Examination record |
| `assessment` | Clinical assessment (diagnosis) |
| `treatment` | Treatment record |
| `examination_treatment` | Examination–treatment linkage |
| `medical_service_treatment` | Medical services during treatment |
| `referral` | Referral out |
| `practitioner_evaluation` | Doctor evaluation and sign-off |
| `frontline` | Frontline staff workflow |
| `data_sync` | Visit data synchronisation utilities |
| `family_relationship` | Patient family/relationship record |

2.3 **Backbone retains these modules** (unchanged):

`patient` (demographics/identity), `tenant`, `user`, `employee`, `role`, `permission`, `menu`, `unicode`, `autolist`, `address`, `reference_service`, `consumer`, `card_identity`

The `patient` module stays in Backbone because it is a shared identity reference — Cashier, Pharmacy, BPJS, and reporting all need patient demographics. It is not a clinical domain concern.

2.4 **Canonical visit record (the one-way door).** On doctor final sign-off, Consultation writes a FHIR-shaped canonical record to the Backbone schema. This record contains outcomes only: confirmed diagnosis, treatment summary, prescriptions dispensed, attending doctor. It is the record used for SATU SEHAT sync, closed insurance invoices, and audit trails. In-progress visit state (draft assessments, partial treatments, form data) lives exclusively in Consultation's schema and is not visible to Backbone until sign-off.

This gives Backbone a clear and legitimate role: system of canonical medical record. It is not clinical logic. It is a structured write that other services consume downstream.

2.5 **Phase 1 / Phase 2 split.** Phase 1 delivers the extraction: separate binary, all 13 modules moved, gRPC server running, Gateway routing updated. Phase 2 delivers full cross-service saga coordination (see §5.3 and the Phase 2 plan reference).

---

## 3. Consequences

### 3.1 Positive

- **Deployment independence.** Clinical features can be released without touching Backbone's auth, tenant, and config code.
- **Explicit domain ownership.** Clinical logic cannot drift into Backbone. Every new visit-related feature has an obvious home.
- **Defensible boundary for the team.** The lead backend developer has a document to reference when direction conflicts arise.
- **Backbone shrinks.** From 37 modules to ~15, focused on tenant/auth/config/reference data and canonical records.
- **Independent scaling.** Consultation is the highest-traffic service during active clinic hours. It can scale independently of Backbone's auth/config workload.
- **Simpler downstream consumers.** SATU SEHAT integration, billing, and reporting read from the clean canonical record — not from a mix of in-progress visit state and draft assessments.

### 3.2 Negative

- **gRPC proto definitions required** for all 13 Consultation modules. Estimated 3–5 days of interface definition and stub generation work.
- **Gateway update required.** A new `CONSULTATION_GRPC_ADDRESS` env var and a new RPC client struct must be added. The 13 modules' handler routes must be re-pointed.
- **Saga compensation incomplete at Phase 1.** Cross-service saga wiring (visit cancellation unwinding POS writes, pharmacy holds) is not delivered in Phase 1.
- **Shared packages.** The saga framework (`internal/saga/`) and the API client (`pkg/apiclient/`) currently live inside the Backbone repository. Consultation needs its own copies in Phase 1. A shared module extraction is deferred to Phase 2 or later.

### 3.3 Risks

- **Incomplete saga at Phase 1 limits go-live scope.** Multi-service operations (visit + billing + pharmacy) that require compensation are not production-safe until Phase 2. Mitigation: define all saga step interfaces and stub the compensation functions in Phase 1 so Phase 2 is purely implementation, not design.
- **Missed module dependencies surface at runtime.** Any undiscovered cross-module call between a Consultation module and a Backbone module (or vice versa) that isn't accounted for in the extraction will fail at runtime, not at compile time. Mitigation: grep all cross-module imports before finalising the move; log any discovered dependencies in the Phase 2 plan.
- **Patient data boundary.** When Cashier or reporting needs patient name/contact on an invoice or report, they will need to call Backbone's `PatientService` gRPC rather than reading the emr schema directly. This call path doesn't exist today. Mitigation: document as a Phase 2 task; don't pre-optimize before the extraction is validated.

---

## 4. Alternatives Considered

| Alternative | Pros | Cons | Why Not Chosen |
|-------------|------|------|----------------|
| **Option A — Keep EMR in Backbone permanently** | Saga coordination trivially simple (same process). No gRPC/proto cost. Fastest near-term. | Backbone becomes a God service (37 → 40+ modules with no end). No deployment independence. No defensible team boundary. Architectural entropy accelerates as new features are added without an explicit boundary to enforce. | Rejected. The short-term simplicity cost compounds into a long-term structural debt that cannot be unwound incrementally. The boundary must be drawn before new features are added, not after. |
| **Option C — Backbone as thin saga coordinator (hybrid)** | Cleaner than A. Consultation owns clinical logic; Backbone coordinates cross-service workflows. | The request flow has an unsolvable dependency direction: if Gateway calls Consultation first, and Consultation triggers Backbone for saga coordination, Consultation is calling upstream — inverting the dependency. If Gateway calls Backbone first, Backbone is the handler for all clinical requests and is not thin. The pattern creeps without a hard rule. | Rejected. The logical flow breaks regardless of how coordination is framed. The right answer is that Consultation owns its own saga coordination for the visit lifecycle, calling downstream services directly (Cashier, Pharmacy). |

---

## 5. Implementation Notes

5.1 **Bootstrap:** Create `wellmed-consultation` using `bootstrap-repo.sh` per the `wellmed-infrastructure` standard. Set module path to `github.com/Kalpa-Health/wellmed-consultation`.

5.2 **Saga framework:** Copy `internal/saga/` from Backbone into Consultation for Phase 1. Do not import Backbone as a Go module dependency. The two copies evolve independently until a shared module extraction is planned.

5.3 **Saga Phase 2 scope** (not in this ADR, documented in `plans/consultation-extraction/PLAN.md`):
- Forward steps: visit creation → POS service line items push → pharmacy prescription hook → future linked services
- Backward compensation: visit cancellation (wrong doctor/department) → void POS transaction → release pharmacy hold
- Sign-off one-way doors: SATU SEHAT async publish, closed invoice creation, canonical record write to Backbone
- Nested saga pattern: post-creation service additions (add treatment mid-visit) are treated as new, smaller sagas — not nested inside the original visit saga

5.4 **Canonical record write direction:** Consultation calls Backbone's `CanonicalVisitService` gRPC on sign-off. This is the one legitimate Consultation → Backbone call. All other dependencies flow downstream (Consultation → Cashier, Consultation → Pharmacy, Consultation → RabbitMQ).

5.5 **Gateway routing:** The 13 Consultation modules currently route to `BACKBONE_GRPC_ADDRESS`. After extraction they route to `CONSULTATION_GRPC_ADDRESS`. The gateway handler files for these modules do not change — only the RPC client target changes.

5.6 **Execution plan:** Full step-by-step extraction plan for use with the task-runner: `kalpa-docs/plans/consultation-extraction/PLAN.md`.

---

## 6. References

- [`wellmed-system-architecture.md §2.3.2`](../wellmed-system-architecture.md) — Gateway routing current vs. target state
- [`wellmed-system-architecture.md §3.2.3`](../wellmed-system-architecture.md) — SAGA orchestrator pattern
- [`services/backbone.md`](../services/backbone.md) — Backbone service documentation
- [`meetings/team-meeting-brief-march-2026.md §7`](../meetings/team-meeting-brief-march-2026.md) — Decomposition discussion and module mapping
- [`plans/consultation-extraction/PLAN.md`](../plans/consultation-extraction/PLAN.md) — Phase 1 and Phase 2 execution plan

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 03 Mar 2026 | Alex + Claude | Initial ADR — accepted. Three options evaluated; Option B (full extraction) chosen. |
