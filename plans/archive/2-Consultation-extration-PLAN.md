# Plan: wellmed-consultation Service Extraction

**Version:** 2.1
**Date:** 04 March 2026
**Author:** Alex
**ADR:** [`ADR-002`](../../adrs/ADR-002-consultation-service-extraction.md) |
         [`ADR-005`](../../adrs/ADR-005-saga-orchestration.md) |
         [`ADR-006`](../../adrs/ADR-006-domain-service-boundaries.md)
**Status:** Ready to execute — updated to incorporate saga architecture decisions

> **v2.0 Changes:** Step 1.3 updated (no saga copy — see ADR-005). Step 1.5 updated
> (no gRPC clients for cross-module deps — see ADR-006). Phase 2 scope rewritten
> (Consultation as participant, not orchestrator). Testing section added. Consultation
> domain ownership documented. data_sync module flagged for audit.
>
> **v2.1 Changes (post-audit):** Module count 13 → 12: `data_sync` stays in Backbone
> (contains saga builder — ADR-005 gate). `service_price` stays in Backbone (Pricing
> package boundary — future extraction target). `item_rent` stays in Backbone (shared
> with backbone-only `patient` module). `unicode` resolution: Consultation carries a
> read-only repository copy (direct tenant DB read, no gRPC); Phase 2 saga payloads
> include pre-composed unicode data from Backbone.

---

## Overview

Extract 12 clinical consultation modules from `wellmed-backbone` into a new standalone microservice `wellmed-consultation`. After Phase 1, both services run independently. Phase 2 (saga wiring) is planned separately using Phase 1 as operational context.

**Repos involved:**
- `wellmed-backbone` — source of modules, will be modified
- `wellmed-gateway-go` — routing update required
- `wellmed-consultation` — new repo (created in this plan)
- `kalpa-docs` — documentation updates

**Local paths:**
- Backbone: `/Users/alexknecht/Projects/WellMed/wellmed-backbone`
- Gateway: `/Users/alexknecht/Projects/WellMed/wellmed-gateway-go`
- Docs: `/Users/alexknecht/Projects/WellMed/kalpa-docs`

---

## Modules Moving to wellmed-consultation (12)

`visit_patient`, `visit_registration`, `visit_registration_referral`, `visit_examination`, `assessment`, `treatment`, `examination_treatment`, `medical_service_treatment`, `referral`, `practitioner_evaluation`, `frontline`, `family_relationship`

## Modules Staying in Backbone

`patient`, `tenant`, `user`, `employee`, `role`, `permission`, `menu`, `unicode`, `autolist`, `address`, `reference_service`, `consumer`, `card_identity`, `data_sync`, `item_rent`, `service_price`

> **`data_sync`**: Contains a saga builder (`saga/builder.go`) — saga ownership stays in Backbone per ADR-005.
>
> **`service_price`**: Pricing package — manages service prices, consumables, admin fees. Intentionally isolated within Backbone as a future extraction target (Payer/Invoice MS). Consultation references item IDs only; never embeds price logic or price data.
>
> **`item_rent`**: Shared with `patient` (backbone-only module). Cannot move to Consultation without violating ADR-006. Consultation modules that reference item rentals receive item IDs only; full item data arrives via Backbone saga payload in Phase 2.
>
> **`unicode`**: Reference/master data (configurable enums: patient type codes, family role codes, treatment type codes, etc.). Backbone owns and manages it. Consultation carries a **read-only copy** of `UnicodeRepository` for direct tenant DB lookups (no gRPC call — same underlying tenant DB). In Phase 2 saga paths, Backbone pre-composes unicode data in step payloads so Consultation receives it already resolved.

---

## Phase 1: Structural Extraction

### 1.1 Pre-extraction Audit

- [ ] **Audit cross-module imports in the 12 Consultation modules.** For each module being moved, grep for any imports of Backbone-only modules (e.g., `patient`, `tenant`, `user`). Document every discovered dependency. Any module that calls into a non-Consultation module needs a gRPC client in Consultation to reach it after extraction.
  ```
  grep -rn "backbone\|/patient\|/tenant\|/user" internal/domain/visit_patient/ --include="*.go"
  ```
  Repeat for each of the 12 modules. Log findings before proceeding.

- [ ] **Audit saga steps that touch Consultation modules.** In `internal/saga/`, identify every saga that references a Consultation module. Document: step name, what it does, what compensation it requires. This is the input for Phase 2.

- [ ] **Note any shared utilities used by Consultation modules** (DB connection manager, Redis client, config loader). These patterns need to be replicated in the new service.

- [ ] ~~**Special audit: `data_sync` module placement.**~~ **RESOLVED (v2.1):** `data_sync`
  stays in Backbone. `internal/domain/data_sync/saga/builder.go` confirmed present — saga
  builder found. ADR-005 gate triggered. Module count is 12.

> **HUMAN CHECKPOINT:** Review audit findings. Confirm no blocking dependencies were missed before creating the new repo and starting the move.

---

### 1.2 Bootstrap wellmed-consultation Repo

- [ ] **Run bootstrap script** from `wellmed-infrastructure`:
  ```bash
  cd ~/Projects/WellMed/infrastructure/scripts
  ./bootstrap-repo.sh Kalpa-health/wellmed-consultation \
    --binary-name wellmed-consultation \
    --cmd-path ./cmd
  ```

- [ ] **Update go.mod** module path: `github.com/Kalpa-Health/wellmed-consultation`

  > Note: Backbone's module path uses `github.com/ingarso/wellmed-backbone` (legacy org name).
  > The new Consultation repo uses `github.com/Kalpa-Health/wellmed-consultation`. This is
  > intentional — `Kalpa-Health` is the active GitHub org. Migration of Backbone's path is
  > a separate backlog item.

- [ ] **Add ANTHROPIC_API_KEY secret** to the new repo:
  ```bash
  gh secret set ANTHROPIC_API_KEY -r Kalpa-health/wellmed-consultation
  ```

- [ ] **Update `.github/workflows/pr-review.yml`** — fill in the `TODO:ARCH` section with a description of the Consultation service.

---

### 1.3 Scaffold the Consultation Service

- [ ] **Copy the service skeleton** from Backbone: `cmd/`, `internal/app/`, `config/`, `.env.example`. Rename binary references from `wellmed-backbone` to `wellmed-consultation`.

- [ ] **Copy DB connection manager** (`internal/db/` or equivalent) verbatim. Consultation needs its own `ConnectionManager` for multi-tenant PostgreSQL access. Do not import Backbone as a dependency.

- [ ] **Copy Redis client setup** verbatim.

- [ ] **Do NOT copy `internal/saga/`.** Per ADR-005, Backbone is the sole saga
  orchestrator. Consultation receives saga step triggers and responds — it does not
  orchestrate. Do not replicate saga orchestration logic in Consultation.

- [ ] **Scaffold RabbitMQ consumer framework** (stub only) at
  `internal/queue/consumer.go`. This infrastructure enables Consultation to receive
  saga step triggers from Backbone in Phase 2. In Phase 1, all handlers return
  `TODO: Phase 2 — implement step handler`.

  Structure mirrors Backbone's event consumer. Required interface:
  ```go
  type StepHandler interface {
      Handle(ctx context.Context, payload []byte) error
      Compensate(ctx context.Context, payload []byte) error
  }
  ```

- [ ] **Add `SagaCallbackService` gRPC client stub** at
  `internal/clients/saga_callback_client.go`. This client is how Consultation
  reports saga step results back to Backbone after processing a step trigger.

  Interface: `ReportStepResult(ctx, StepResultRequest) (*StepResultResponse, error)`

  Proto source: `wellmed-backbone/proto/saga_callback.proto` (from saga PRD Phase 2).
  If this proto is not yet merged to Backbone at time of Phase 1 execution, create
  a local placeholder in `proto/saga_callback_stub.proto` marked:
  ```
  // TODO: Replace with generated client from Kalpa-Health/wellmed-backbone when
  // saga-pattern-extended-PRD Phase 2 merges. Tracking issue: [link]
  ```

- [ ] ~~**Copy `pkg/apiclient/`**~~ **RESOLVED (v2.1):** No consultation module uses
  `internal/apiclient/`. Skip.

- [ ] **Copy read-only `unicode` repository** into Consultation at
  `internal/domain/unicode/repository/unicode.go`. Copy the interface and implementation
  verbatim from Backbone. Read methods only (`FindById`, `FindByFlag`, `FindByName`).
  Do NOT copy the service layer or wires — Consultation is a consumer, not an owner.

  Also copy `internal/schema/entity/unicode.go` into Consultation's schema entity package.

  **Rationale:** 6 of the 12 modules use `unicode` for write-time ID validation. Both
  services connect to the same per-tenant DB via `ConnectionManager` — this is a direct
  DB read, not a cross-service call. ADR-006 is not violated. In Phase 2 saga paths,
  Backbone pre-composes unicode fields in step payloads so no lookup is needed.

- [ ] **Set gRPC port.** Consultation runs on `:50052`. Update `cmd/main.go` and `.env.example`. Document in `services/consultation.md` after Phase 1.

- [ ] **Verify the service compiles with no modules yet:**
  ```bash
  cd ~/Projects/WellMed/wellmed-consultation
  go build ./...
  ```

---

### 1.4 Define Consultation Proto Interface

- [ ] **Create proto file** `proto/consultation/consultation.proto`. Define gRPC services for all 12 modules. Mirror the method signatures that currently exist in Backbone's proto for these modules. Example structure:
  ```protobuf
  service VisitPatientService {
    rpc GetAll(VisitPatientRequest) returns (VisitPatientListResponse);
    rpc GetById(VisitPatientByIdRequest) returns (VisitPatientResponse);
    rpc Store(VisitPatientStoreRequest) returns (VisitPatientResponse);
    rpc UpdateStatus(VisitPatientStatusRequest) returns (VisitPatientResponse);
  }
  // ... repeat for all 12 services
  ```

- [ ] **Do NOT define `CanonicalVisitService` as a proto service in Consultation.**
  Backbone is the SERVER. Consultation is the CLIENT. The service definition lives at
  `wellmed-backbone/proto/canonical_visit.proto` (saga PRD §3.3.2).

  Add the canonical visit CLIENT to Consultation:
  Location: `internal/clients/canonical_visit_client.go`

  Interface: `WriteVisitRecord(ctx, WriteVisitRecordRequest) (*WriteVisitRecordResponse, error)`

  This client is called on doctor final sign-off. In Phase 1 it is a stub — it documents
  the direction of the call and blocks on a `TODO: Phase 2`.

  Proto source: `wellmed-backbone/proto/canonical_visit.proto`. If not yet merged,
  use a local stub marked the same way as the SagaCallbackService stub above.

  Note on Consultation's visit ownership: `visit_id` is generated and owned by
  Consultation. The canonical record sent on sign-off contains OUTCOMES only —
  confirmed diagnosis, treatment summary, prescriptions dispensed, attending doctor.
  All in-progress clinical state (form fill, assessments, approvals, service info)
  lives exclusively in Consultation and is never written to Backbone until sign-off.

> **HUMAN CHECKPOINT:** Review proto interface definitions. Confirm all method signatures are complete and match what the Gateway currently calls on Backbone. This is the contract — approve before generating stubs.

- [ ] **Generate gRPC stubs** from proto:
  ```bash
  protoc --go_out=. --go-grpc_out=. proto/consultation/consultation.proto
  ```

---

### 1.5 Move the 12 Modules

- [ ] **Copy each module** from `wellmed-backbone/internal/domain/{module}/` to `wellmed-consultation/internal/domain/{module}/`. Do this one module at a time and verify compilation after each move.

  Order (least cross-dependencies first, per audit):
  1. `family_relationship`
  2. `referral`
  3. `assessment`
  4. `treatment`
  5. `examination_treatment`
  6. `medical_service_treatment`
  7. `practitioner_evaluation`
  8. `visit_examination`
  9. `frontline`
  10. `visit_registration_referral`
  11. `visit_registration`
  12. `visit_patient`

- [ ] **Update import paths** in each moved module from `github.com/Kalpa-Health/wellmed-backbone/...` to `github.com/Kalpa-Health/wellmed-consultation/...`.

- [ ] **For any module that calls a Backbone-only module** (found in audit 1.1):
  DO NOT add a gRPC client. Per ADR-006 §2.4, the only permitted Consultation →
  Backbone call is the `CanonicalVisitService` sign-off client added in Step 1.3.

  Resolution per dependency type:

  **`unicode` (patient type, family role, treatment/examination/referral type codes):**
  Consultation carries a read-only `UnicodeRepository` (copied in Step 1.3). No change
  needed to the service layer calls — they continue to call `unicodeRepo.FindById(ctx, id)`
  using Consultation's own `ConnectionManager`. In Phase 2 saga paths, Backbone
  pre-composes unicode fields in payloads so no lookup is needed.

  **`patient` (patient_id reference):**
  ```go
  // TODO: Phase 2 — patient_id received via Backbone saga payload (create_visit_record step).
  // See ADR-006. Do not call PatientService directly.
  ```

  **`card_identity`, `reference_service` (visit_patient):**
  ```go
  // TODO: Phase 2 — received via Backbone saga payload.
  // See ADR-006. Backbone composes this field in the create_visit saga step trigger.
  ```

  **`item_rent` (referral, visit_registration_referral, visit_patient):**
  ```go
  // TODO: Phase 2 — item IDs referenced only. Full item data via Backbone saga payload.
  // service_price stays in Backbone (Pricing package). See audit findings 2026-03-04.
  ```

  Exception: `CanonicalVisitService` client is the one permitted cross-boundary call
  (already scaffolded in Step 1.3).

- [ ] **Register all 12 gRPC services** in `wellmed-consultation/internal/app/application.go`.

- [ ] **Verify Consultation compiles and all 12 services are registered:**
  ```bash
  go build ./...
  go test ./...
  ```

> **HUMAN CHECKPOINT:** Verify `wellmed-consultation` compiles cleanly. Start the service locally and confirm gRPC server registers all 12 services. Basic smoke test: one GetAll call per module via grpcurl or a test client.

---

### 1.6 Update Backbone

- [ ] **Delete the 12 module implementations** from `wellmed-backbone/internal/domain/`. Remove: `visit_patient`, `visit_registration`, `visit_registration_referral`, `visit_examination`, `assessment`, `treatment`, `examination_treatment`, `medical_service_treatment`, `referral`, `practitioner_evaluation`, `frontline`, `family_relationship`.

  **Do NOT delete:** `data_sync` (saga builder — stays in Backbone per ADR-005), `unicode`, `item_rent`, `service_price`.

- [ ] **Remove their registrations** from `wellmed-backbone/internal/app/application.go`.

- [ ] **Register `CanonicalVisitService` gRPC server stub in Backbone.**
  Backbone is the SERVER — it receives the final visit record from Consultation.
  Consultation is the client (scaffolded in Step 1.3).

  Add stub handler: `internal/domain/canonical_visit/handler/canonical_visit_handler.go`
  Implements: `WriteVisitRecord` — returns `TODO: Phase 2` in Phase 1.

  Proto: `wellmed-backbone/proto/canonical_visit.proto` — add this file if not yet
  present. See saga PRD §3.3.2 for the stub definition.

  Register the server in `internal/app/application.go`.

- [ ] **Add `CONSULTATION_GRPC_ADDRESS` env var** to Backbone's config and `.env.example` (for the sign-off write in Phase 2).

- [ ] **Verify Backbone compiles** after module removal:
  ```bash
  go build ./...
  go test ./...
  ```

> **HUMAN CHECKPOINT:** Verify Backbone compiles and its remaining tests pass. Confirm the 12 modules are gone from Backbone's gRPC service catalog.

---

### 1.7 Update Gateway

- [ ] **Add `CONSULTATION_GRPC_ADDRESS`** to Gateway's config, `.env.example`, and `README.md` env var section.

- [ ] **Add a new RPC client struct** for Consultation in Gateway (alongside the existing Backbone client). Follow the same pattern as the Backbone client.

- [ ] **Re-point routing** for all 12 module handlers: change the RPC client target from Backbone to Consultation. The handler code itself does not change — only the client it calls.

  Modules to re-route: `visit_patient`, `visit_registration`, `visit_registration_referral`, `visit_examination`, `assessment`, `treatment`, `examination_treatment`, `medical_service_treatment`, `referral`, `practitioner_evaluation`, `frontline`, `family_relationship`

  **`data_sync` stays routed to Backbone.** Its gRPC handler is not moved.

- [ ] **Verify Gateway compiles:**
  ```bash
  go build ./...
  go test ./...
  ```

> **HUMAN CHECKPOINT:** Verify Gateway compiles. Confirm: (a) Backbone client routes only Backbone modules, (b) Consultation client routes only Consultation modules. Check for any handler accidentally left pointing to the wrong service.

---

### 1.8 AWS Infrastructure

- [ ] **Open port 50052** (Consultation gRPC) in AWS Security Groups — same pattern as port 50051 for Backbone:
  - Production: Gateway fe SG → Consultation app SG, port 50052 inbound
  - Dev/Staging: Gateway fe SG → Consultation dev-app SG, port 50052 inbound

- [ ] **Add `CONSULTATION_GRPC_ADDRESS`** to the appropriate SSM Parameter Store path:
  `/wellmed/{env}/go/gateway/CONSULTATION_GRPC_ADDRESS`

- [ ] **Create ECS task definition** for `wellmed-consultation` (clone from Backbone task definition, update image and env vars).

---

### 1.9 Deploy and Smoke Test

- [ ] **Deploy to dev environment.** Both services must be running simultaneously.

- [ ] **End-to-end smoke test:**
  - [ ] User login (Backbone — unchanged)
  - [ ] Patient lookup (Backbone — unchanged)
  - [ ] Visit registration (Consultation — new routing)
  - [ ] Visit examination entry (Consultation — new routing)
  - [ ] Assessment store (Consultation — new routing)
  - [ ] Treatment store (Consultation — new routing)
  - [ ] Frontline workflow (Consultation — new routing)

> **HUMAN CHECKPOINT:** Full smoke test on dev. Confirm all 12 Consultation flows work through the new service. Confirm all Backbone flows still work. Sign off before proceeding to doc updates.

---

### 1.9.1 Testing Gate (Required Before Phase 1 Complete)

> **Policy:** Per `wellmed-testing-architecture.md §7.3`, 80% service-layer coverage
> is required. This gate must pass before Phase 1 PRs are merged to develop.

- [ ] **Migrate existing tests.** For each of the 12 modules, copy all test files
  from `wellmed-backbone/internal/domain/{module}/*_test.go` into Consultation.
  Update import paths. Run tests — all must pass.

  ```bash
  go test ./internal/domain/... -v -cover
  ```

- [ ] **Achieve 80% service-layer coverage.** For any module that did not have tests
  in Backbone (or whose coverage was below 80%), write tests in Consultation before
  the PR is opened. No exceptions — this is not deferred to Phase 2.

  Coverage check:
  ```bash
  go test ./internal/domain/.../service/... -coverprofile=coverage.out
  go tool cover -func=coverage.out | grep -v "100.0"
  ```

- [ ] **Test new Phase 1 infrastructure code:**
  - `internal/clients/saga_callback_client.go` — unit test the client stub
    (mock gRPC call, verify correct method signature and error handling)
  - `internal/clients/canonical_visit_client.go` — same pattern
  - `internal/queue/consumer.go` — unit test consumer start/stop/registration

- [ ] **CI pipeline passes.** The new `wellmed-consultation` repo's CI must be green:
  lint (`golangci-lint run`), unit tests, build. Update the `pr-review.yml`
  description (§1.2) with Consultation service context before CI runs.

---

### 1.10 Documentation Updates

- [ ] **Create `kalpa-docs/services/consultation.md`** — service doc following the
  same structure as `services/backbone.md`. Required sections:
  - Overview (largest microservice; owns full visit lifecycle; participant in Backbone sagas)
  - Repository (wellmed-consultation, gRPC port :50052, module path)
  - Key Responsibilities (visit lifecycle, clinical domain, saga participant role)
  - gRPC Interface Catalog (12 services)
  - Multi-Tenant Design (same ConnectionManager pattern as Backbone)
  - Saga Participation (RabbitMQ consumer stub, SagaCallbackService client — Phase 2 stub)
  - CanonicalVisitService (sign-off write to Backbone — Phase 2 stub)
  - Data Ownership Boundary (reference the Phase 2 ownership table in this plan)
  - Auth Architecture (JWT propagated from Gateway via gRPC metadata — same as Backbone)

- [ ] **Update `kalpa-docs/services/backbone.md`:**
  - Remove the 12 moved modules from §4 gRPC Interface Catalog
  - Update §3 Key Responsibilities (add CanonicalVisitService receiver, remove clinical logic)
  - Update module count
  - Note: coordinate with saga PRD BB-1 through BB-4 changes to avoid conflicts

- [ ] **Update `kalpa-docs/wellmed-system-architecture.md`:**
  - Add `wellmed-consultation` to the service topology diagram in §2.1
  - Update the Internal Microservices subgraph
  - Update §1.2 product tier table — rename "EMR" → "Consultation" in Lite tier
  - Note: coordinate with saga PRD SA-1 through SA-6 changes. These affect the same
    file. The consultation doc update should be merged AFTER or ALONGSIDE the saga
    PRD doc updates, or conflicts will occur. Confirm order with Hamzah.

- [ ] **Update `kalpa-docs/README.md`** — add `services/consultation.md` to navigation.
- [ ] **Update `kalpa-docs/repo-governance.md`** — add `wellmed-consultation` repo entry.
- [ ] **Update `MEMORY.md`** (Claude's memory file) — add `wellmed-consultation` to
  Kalpa-Health org repos, note ADR-002 is Accepted, note gRPC port :50052.

> **HUMAN CHECKPOINT:** Review all doc updates for accuracy before committing.

---

### 1.11 Commit and PR

- [ ] **wellmed-consultation repo:** Initial commit — "feat: bootstrap wellmed-consultation service (Phase 1 extraction)"
- [ ] **wellmed-backbone repo:** PR to develop — "refactor: extract consultation modules to wellmed-consultation service"
- [ ] **wellmed-gateway-go repo:** PR to develop — "feat: add consultation gRPC client and re-route consultation module handlers"
- [ ] **kalpa-docs repo:** PR to main — "docs: add consultation service doc and update architecture for Phase 1 extraction"

---

## Phase 2: Saga Wiring — Consultation as Participant (Scope Only — Separate Plan)

Phase 2 begins after Phase 1 is in production. The Phase 2 plan is written after
the team has direct experience with the two-service architecture and after the
saga-pattern-extended-PRD's Backbone framework rework is underway.

---

### Consultation's Role in the Visit Saga

Consultation is the **largest microservice** in the application. It owns the complete
clinical visit lifecycle. The visit is initiated by Backbone's saga, but Backbone
is a coordinator — Consultation is the domain owner.

**Key architectural boundary:**

| What Backbone provides | What Consultation creates and owns |
|---|---|
| `patient_id` (ref to backbone.patient) | `visit_id` — Consultation's canonical visit identifier |
| `employee_id` (attending doctor/staff ref) | All clinical records: `visit_patient`, `visit_registration`, `visit_examination`, `assessment`, `treatment`, `referral`, `practitioner_evaluation`, `frontline` |
| Tenant context (`tenant_db`, `tenant_schema`) | Form fill data, decision audits, approval records |
| Frontend-submitted inputs (service selections, appointment ref) | Department assignments, service level records |
| `visit_id` (returned from first step) in subsequent saga payloads | Doctor sign-off status, timestamps, workflow state |

**Sign-off boundary:** At doctor final sign-off, Consultation calls
`CanonicalVisitService.WriteVisitRecord` with outcomes only. Everything else
stays in Consultation permanently.

---

### Consultation Data Ownership — Per Saga Event

| Saga Event | Backbone Sends IN | Consultation Creates / Owns | Returns via Callback |
|---|---|---|---|
| `create_visit_record` (visit start) | `patient_id`, `employee_id`, tenant context, service selections | `visit_id` (ULID), `visit_patient` record, initial status | `{ visit_id }` → Backbone uses this in ALL subsequent step payloads |
| `add_visit_examination` | `visit_id`, examination type, tenant context | `visit_examination` record, examination details | `{ visit_examination_id, status }` |
| `store_assessment` | `visit_id`, clinical context | `assessment` record, ICD codes, diagnosis | `{ assessment_id }` |
| `store_treatment` | `visit_id`, prescribed items | `treatment`, `examination_treatment`, `medical_service_treatment` records | `{ treatment_id, service_line_items }` — Backbone uses for POS step |
| `doctor_sign_off` | `visit_id` | Final `practitioner_evaluation`, sign-off timestamp | Triggers `WriteVisitRecord` to Backbone + SATU SEHAT RabbitMQ event |

> **Note:** The saga events above are illustrative. The exact step names and payloads
> are defined in the Phase 2 plan after Phase 1 is live. The ownership boundaries
> above are canonical — they will not change.

---

### Phase 2 Consultation Work Items

- [ ] Implement RabbitMQ consumer handler: `create_visit_record` step
- [ ] Implement RabbitMQ consumer handler: each additional visit step Backbone triggers
- [ ] Implement compensation handlers: visit cancellation (reverse created records)
- [ ] Wire `SagaCallbackService` client calls on step completion/failure
- [ ] Implement `CanonicalVisitService` client call on doctor sign-off
- [ ] Publish SATU SEHAT sync event to RabbitMQ on sign-off

### Forward Saga — Visit Creation

- [ ] Backbone step: `create_visit_record` → Consultation → returns `visit_id`
- [ ] Backbone uses `visit_id` to compose next step: POS transaction trigger (→ POS module)
- [ ] Backbone uses `visit_id` in pharmacy hook step if prescriptions exist (→ Pharmacy)
- [ ] Additional services added mid-visit: separate smaller saga triggered by Backbone

### Backward Saga — Visit Cancellation

- [ ] Compensation: void in-progress visit records in Consultation
- [ ] Compensation: triggers from Backbone to POS (void transaction), Pharmacy (release hold)
- [ ] Trigger: user-initiated (wrong doctor, department error) — all compensations via Backbone

### Sign-off One-Way Doors (Non-Compensatable)

- [ ] Consultation → Backbone: `WriteVisitRecord` (canonical FHIR-shaped outcome record)
- [ ] Consultation → RabbitMQ: publish SATU SEHAT sync event
- [ ] These are final — doctor sign-off cannot be undone

### Removed from Phase 2 Scope (Superseded by ADR-005)

- ~~Consultation pushes directly to POS~~ → POS triggered by Backbone as separate saga step
- ~~Consultation pharmacy hook~~ → Pharmacy triggered by Backbone
- ~~Shared saga framework extraction~~ → Resolved: saga stays in Backbone (ADR-005 §2.1)
- ~~Nested saga for mid-visit additions~~ → Backbone-owned smaller sagas, not Consultation-initiated
- ~~Sign-off compensation~~ → Sign-off is a one-way door; there is no compensation path

---

## Reference Documents

| Document | Location |
|----------|----------|
| ADR-002 | `kalpa-docs/adrs/ADR-002-consultation-service-extraction.md` |
| ADR-005 | `kalpa-docs/adrs/ADR-005-saga-orchestration.md` |
| ADR-006 | `kalpa-docs/adrs/ADR-006-domain-service-boundaries.md` |
| Saga Pattern Extended PRD | `kalpa-docs/development/saga-pattern-extended-PRD.md` |
| Indonesian team brief | `kalpa-docs/meetings/consultation-decomp-team-brief-id.md` |
| Backbone service doc | `kalpa-docs/services/backbone.md` |
| System architecture | `kalpa-docs/wellmed-system-architecture.md` |
| Backbone source | `/Users/alexknecht/Projects/WellMed/wellmed-backbone` |
| Gateway source | `/Users/alexknecht/Projects/WellMed/wellmed-gateway-go` |

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 03 Mar 2026 | Alex + Claude | Initial plan — Phase 1 and Phase 2 scope. |
| 2.0 | 04 Mar 2026 | Alex + Claude | Incorporated ADR-005/006 and saga architecture decisions: removed saga framework copy, replaced with RabbitMQ consumer + SagaCallbackService client stubs; clarified CanonicalVisitService direction (Consultation is client, Backbone is server); changed cross-module dep resolution to document-only; rewrote Phase 2 scope (Consultation as participant); added Consultation data ownership table; added Phase 1 testing gate (80% service-layer coverage); expanded doc update coordination notes. |
| 2.1 | 04 Mar 2026 | Alex + Claude | Post-audit decisions: module count 13→12 (data_sync stays in Backbone — saga builder found); service_price stays in Backbone (Pricing package boundary, future extraction target); item_rent stays in Backbone (shared with patient module); unicode resolution — Consultation carries read-only UnicodeRepository (direct tenant DB read), Phase 2 saga payloads include pre-composed unicode data. |
