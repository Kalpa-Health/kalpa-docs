# Plan: Visit Saga Wiring — Consultation Phase 2

**Version:** 1.0
**Date:** 05 Mar 2026
**Author:** Alex + Claude
**Status:** Ready to execute
**ADR:** `kalpa-docs/adrs/ADR-002-consultation-service-extraction.md` (Phase 2)
**ADR:** `kalpa-docs/adrs/ADR-005-saga-orchestration.md`
**PRD:** `kalpa-docs/development/saga-pattern-extended-PRD.md`

## Related Docs

- `kalpa-docs/adrs/ADR-002-consultation-service-extraction.md`
- `kalpa-docs/adrs/ADR-005-saga-orchestration.md`
- `kalpa-docs/adrs/ADR-006-domain-service-boundaries.md`
- `kalpa-docs/development/saga-pattern-extended-PRD.md`
- `wellmed-backbone/internal/saga/` — refactored framework (complete)
- `wellmed-backbone/proto/saga_callback.proto` — written, **Go not yet generated**
- `wellmed-backbone/proto/saga_status.proto` — written, **Go not yet generated**
- `wellmed-backbone/proto/canonical_visit.proto` — written and generated
- `wellmed-consultation/internal/clients/saga_callback_client.go` — stub
- `wellmed-consultation/internal/clients/canonical_visit_client.go` — stub
- `wellmed-consultation/internal/queue/consumer.go` — stub handlers

---

## Context

The saga framework rework (saga-pattern-extended plan) is complete. Consultation extraction Phase 1 is code-complete. This plan wires the two together:

1. `SagaCallbackService.ReportStepResult` gRPC endpoint on Backbone — so Consultation can report saga step results
2. `CreateVisit` saga builder on Backbone — so visit creation becomes a saga-orchestrated operation
3. Consultation step handlers for `create_visit` — so Consultation processes the RabbitMQ trigger and calls back
4. `SagaStatusService.GetStatus` gRPC endpoint on Backbone — so Gateway can poll saga status
5. `DoctorSignOff` saga builder (follow-on) — chains visit sign-off through POS billing and SATU SEHAT

**The visit wiring sequence:**

```
Backbone publishes                Consultation handles            Backbone receives callback
──────────────────               ──────────────────────          ──────────────────────────
CreateVisit saga
  Execute()
  ├── Local steps (patient lookup, pre-checks)
  └── Remote: publish
        saga.create_visit
        .record_visit_in_consultation
        .trigger
                              →  consumer.go StepConsumer
                                 routes to CreateVisitHandler
                                 creates visit_patient record
                                 returns visit_id
                              →  SagaCallbackClient.ReportStepResult(
                                   saga_id, step_id, COMPLETED,
                                   {visit_id: "..."})
                                                                  SagaCallbackService
                                                                  CallbackHandler.HandleStepResult()
                                                                  continuator.HandleCallback()
                                                                  → advance to next step OR
                                                                    mark saga completed
                                                                  → update Redis + PostgreSQL
```

**Current stubs blocking this flow:**
- `sagacallbackpb/` is empty — `saga_callback.proto` not yet compiled (HUMAN task)
- `sagastatuspb/` is empty — `saga_status.proto` not yet compiled (HUMAN task)
- `CallbackHandler` and `StatusHandler` in Backbone use local types, not registered in gRPC server
- `SagaCallbackClient` in Consultation returns `ErrNotImplemented`
- `CanonicalVisitClient` in Consultation returns `ErrNotImplemented`
- `consumer.go` handlers are all `TODO: Phase 2`
- No `CreateVisit` saga builder exists in Backbone

---

## Phase 1: Proto Generation (HUMAN — Abdul)

### Task 1.1: Generate sagacallbackpb
- **Type**: HUMAN
- **Input**: `wellmed-backbone/proto/saga_callback.proto` (already written)
- **Action**: Run protoc from the backbone repo root:
  ```bash
  protoc --go_out=proto/sagacallbackpb --go_opt=paths=source_relative \
         --go-grpc_out=proto/sagacallbackpb --go-grpc_opt=paths=source_relative \
         proto/saga_callback.proto
  ```
- **Output**: `proto/sagacallbackpb/saga_callback.pb.go`, `proto/sagacallbackpb/saga_callback_grpc.pb.go`
- **Acceptance**: Both files exist. `sagacallbackpb.SagaCallbackServiceServer` interface visible.

### Task 1.2: Generate sagastatuspb
- **Type**: HUMAN
- **Input**: `wellmed-backbone/proto/saga_status.proto` (already written)
- **Action**: Run protoc:
  ```bash
  protoc --go_out=proto/sagastatuspb --go_opt=paths=source_relative \
         --go-grpc_out=proto/sagastatuspb --go-grpc_opt=paths=source_relative \
         proto/saga_status.proto
  ```
- **Output**: `proto/sagastatuspb/saga_status.pb.go`, `proto/sagastatuspb/saga_status_grpc.pb.go`
- **Acceptance**: Both files exist. `sagastatuspb.SagaStatusServiceServer` interface visible.

### Task 1.3: Verify canonical_visit.proto Go is current
- **Type**: HUMAN
- **Action**: Confirm `proto/canonicalvisitpb/canonical_visit.pb.go` matches current `canonical_visit.proto` fields. Re-run protoc if the proto was updated after generation.
- **Acceptance**: Fields in Go struct match proto: `VisitId`, `PatientId`, `Tenant`, `DoctorId`, `DiagnosisCodes`, `TreatmentSummary`, `SignOffAt`.

---
### 🔲 CHECKPOINT: Proto Review
**Review**: All three proto pb directories have `.pb.go` and `_grpc.pb.go` files. Fields match what consultation stubs expect (`SagaID`, `StepID`, `TenantID`, etc.). Run `go build ./...` in backbone — must compile.
**Resume**: "continue the visit-saga-wiring plan"
---

## Phase 2: Wire Backbone gRPC Handlers

### Task 2.1: Wire SagaCallbackService into application.go
- **Type**: AI
- **Input**: `wellmed-backbone/internal/app/application.go` §9 TODO comment, `wellmed-backbone/internal/saga/handler/callback_handler.go`
- **Action**: Replace the TODO comment block in application.go §9 with real wiring:
  - Import `sagacallbackpb "github.com/kalpa-health/wellmed-backbone/proto/sagacallbackpb"` and `sagahandler "github.com/kalpa-health/wellmed-backbone/internal/saga/handler"`
  - Create `callbackHandler := sagahandler.NewCallbackHandler(sagaContinuator, sagaReplyStore)`
  - Add `sagacallbackpb.RegisterSagaCallbackServiceServer(a.grpcServer, callbackHandler)`
  - The `CallbackHandler.HandleStepResult` method must implement `SagaCallbackServiceServer` interface — adapt method signature to accept `*sagacallbackpb.StepResultRequest` and return `*sagacallbackpb.StepResultResponse`
- **Output**: Modified `application.go`, modified `callback_handler.go`
- **Acceptance**: `go build ./...` passes. `SagaCallbackService` visible in `grpc_reflection` listing.

### Task 2.2: Wire SagaStatusService into application.go
- **Type**: AI
- **Input**: `wellmed-backbone/internal/app/application.go`, `wellmed-backbone/internal/saga/handler/status_handler.go`
- **Action**: After wiring callback handler, add:
  - Import `sagastatuspb "github.com/kalpa-health/wellmed-backbone/proto/sagastatuspb"`
  - Create `statusHandler := sagahandler.NewStatusHandler(sagaReplyStore, sagaRepository)`
  - Add `sagastatuspb.RegisterSagaStatusServiceServer(a.grpcServer, statusHandler)`
  - Adapt `StatusHandler.GetStatus` signature to match `SagaStatusServiceServer` interface
- **Output**: Modified `application.go`, modified `status_handler.go`
- **Acceptance**: `go build ./...` passes. Gateway's `GET /api/v1/saga/:saga_id/status` can call this.

### Task 2.3: Wire CanonicalVisitService handler
- **Type**: AI
- **Input**: `wellmed-backbone/internal/domain/canonical_visit/handler/canonical_visit_handler.go`
- **Action**: Implement `WriteVisitRecord` — currently returns `Unimplemented`. Should:
  1. Validate `visit_id`, `patient_id`, `tenant`, `sign_off_at` are non-empty
  2. Inject tenant context from `req.Tenant`
  3. Persist canonical record to backbone's `canonical_visits` table (create migration if needed)
  4. Return `canonical_record_id` (new ULID)
  5. Publish async RabbitMQ message to trigger SATU SEHAT sync (routing key: `backbone.canonical_visit.created`)
- **Output**: Implemented `canonical_visit_handler.go`, new migration `003_create_canonical_visits_table.sql`
- **Acceptance**: Calling `WriteVisitRecord` with valid fields creates a DB record and returns a canonical_record_id.
- **Notes**: Keep handler thin — create a `canonical_visit` service layer. Do not put business logic in handler.

---
### 🔲 CHECKPOINT: Backbone Handler Review
**Review**: Three new gRPC handlers registered: SagaCallbackService, SagaStatusService, CanonicalVisitService. `go build ./...` passes. `go test ./internal/saga/...` still passes. Verify in grpc_cli or Postman that all three services respond (even if stub data).
**Resume**: "continue the visit-saga-wiring plan"
---

## Phase 3: Wire Consultation Clients

### Task 3.1: Replace SagaCallbackClient stub with real gRPC client
- **Type**: AI
- **Input**: `wellmed-consultation/internal/clients/saga_callback_client.go`, generated `sagacallbackpb`
- **Action**: Replace `ErrNotImplemented` stub with real gRPC client:
  - Add `BACKBONE_GRPC_ADDRESS` env var read (same address as existing backbone client)
  - Create `grpc.Dial` connection to backbone
  - Replace `ReportStepResult` stub body with `pb.NewSagaCallbackServiceClient(conn).ReportStepResult(...)`
  - Map local `StepResultRequest` fields → `sagacallbackpb.StepResultRequest`
  - Map `StepStatus` enum values to proto enum
- **Output**: Real gRPC implementation in `saga_callback_client.go`
- **Acceptance**: Calling `ReportStepResult` reaches backbone. Can be verified with integration test.

### Task 3.2: Replace CanonicalVisitClient stub with real gRPC client
- **Type**: AI
- **Input**: `wellmed-consultation/internal/clients/canonical_visit_client.go`, generated `canonicalvisitpb`
- **Action**: Same pattern as Task 3.1 but for `CanonicalVisitService.WriteVisitRecord`.
  - Map `WriteVisitRecordRequest` fields to `canonicalvisitpb.WriteVisitRecordRequest`
  - Called only from doctor sign-off handler (one call site)
- **Output**: Real gRPC implementation in `canonical_visit_client.go`
- **Acceptance**: Calling `WriteVisitRecord` creates a canonical record in backbone.
- **Notes**: This is the ONLY Consultation → Backbone call permitted by ADR-006. Do not add others.

---
### 🔲 CHECKPOINT: Client Wiring Review
**Review**: Both stubs are replaced. Build passes in both repos. Manually test: start backbone + consultation locally, trigger a sign-off, verify canonical record appears in backbone DB.
**Resume**: "continue the visit-saga-wiring plan"
---

## Phase 4: CreateVisit Saga Builder + Consultation Step Handler

### Task 4.1: CreateVisit saga builder in Backbone
- **Type**: AI
- **Input**: `wellmed-backbone/internal/domain/patient/saga/builder.go` (reference for pattern), ADR-005, ADR-006
- **Action**: Create `wellmed-backbone/internal/domain/visit/saga/builder.go` with `CreateVisitSagaBuilder`:
  - Step 1 (Local): ValidateVisitPrerequisites — check patient exists, employee exists, no duplicate active visit
  - Step 2 (Remote): RecordVisitInConsultation — publishes to `saga.create_visit.record_visit_in_consultation.trigger`. Trigger payload: `{patient_id, employee_id, service_ids[], visit_date, tenant}`. Compensation: `saga.create_visit.record_visit_in_consultation.compensate`.
  - Step 3 (Remote, optional): CreatePOSVisitLine — publishes to `saga.create_visit.create_pos_visit_line.trigger`. Trigger payload uses `visit_id` from Step 2 result (stored in SagaContext). Compensation publishes to `.compensate`.
  - `BuildFromSaga(s *saga.Saga) ([]*saga.SagaStep, error)` adapter method for Continuator registration.
- **Output**: `wellmed-backbone/internal/domain/visit/saga/builder.go`
- **Acceptance**: Builder compiles. Steps reference correct routing keys. SagaContext correctly threads `visit_id` from Step 2 into Step 3 payload.
- **Notes**: Step 2 result payload from Consultation: `{"visit_id": "01JXXX..."}`. Backbone stores it in `SagaContext` key `visit_id` for Step 3.

### Task 4.2: Wire CreateVisit builder in application.go
- **Type**: AI
- **Input**: `wellmed-backbone/internal/app/application.go`, Task 4.1 output
- **Action**:
  - Create `visitSagaBuilder := visitSaga.NewCreateVisitSagaBuilder(...)` (pass required repos)
  - Register: `sagaContinuator.RegisterSagaBuilder("create_visit", visitSagaBuilder.BuildFromSaga)`
  - Add `Execute` call site in the visit service (or create a thin `CreateVisitService` that calls `sagaOrchestrator.Execute(ctx, "create_visit", payload, steps)`)
- **Output**: Modified `application.go`, new visit service/wires
- **Acceptance**: Calling visit creation returns `saga_id`. Saga record created in DB. RabbitMQ trigger published to correct routing key.

### Task 4.3: Consultation CreateVisit step handler
- **Type**: AI
- **Input**: `wellmed-consultation/internal/queue/consumer.go`, Task 4.1 routing keys
- **Action**: Replace the `TODO: Phase 2` stub for `saga.create_visit.record_visit_in_consultation.trigger` with a real handler:
  - Parse trigger payload: `patient_id`, `employee_id`, `service_ids[]`, `visit_date`, `tenant`
  - Call `visit_patient` service to create visit → returns `visit_id`
  - Call `saga_callback_client.ReportStepResult(COMPLETED, {visit_id: visit_id})`
  - Compensation handler: receive `saga.create_visit.record_visit_in_consultation.compensate`, delete the visit record, call `ReportStepResult(COMPLETED)`
- **Output**: Real handler registered in `consumer.go`
- **Acceptance**: Sending a `create_visit` trigger message to RabbitMQ creates a visit in consultation DB and triggers a callback to backbone.

---
### 🔲 CHECKPOINT: CreateVisit E2E
**Review**: Full flow test:
1. POST create visit → Backbone returns `{saga_id}`
2. Backbone publishes trigger to `saga.create_visit.record_visit_in_consultation.trigger`
3. Consultation creates visit, calls `SagaCallbackService.ReportStepResult(COMPLETED, {visit_id})`
4. Backbone updates saga state to completed (or triggers next step if POS line)
5. GET `/api/v1/saga/{saga_id}/status` returns `{status: "completed", result_payload: {visit_id}}`
**Resume**: "continue the visit-saga-wiring plan"
---

## Phase 5: DoctorSignOff Saga Builder

### Task 5.1: DoctorSignOff saga builder in Backbone
- **Type**: HUMAN
- **Notes**: Abdul implements. Chain: Backbone → Consultation (finalize SOAP) → POS (trigger billing) → Backbone CanonicalVisitService (WriteVisitRecord) → RabbitMQ SATU SEHAT async.
  - Step 1 (Local): ValidateSignOff — check visit in correct state
  - Step 2 (Remote): FinalizeConsultationRecord — trigger `saga.doctor_sign_off.finalize_consultation_record.trigger`; Consultation closes SOAP, calls back with `{diagnosis_codes, treatment_summary}`
  - Step 3 (Remote): CreateBillingInPOS — trigger `saga.doctor_sign_off.create_billing_in_pos.trigger`; POS creates invoice lines, calls back with `{invoice_id}`
  - Step 4 (Local): WriteCanonicalRecord — calls `CanonicalVisitService.WriteVisitRecord` (this is a local step because Backbone IS the server; no RabbitMQ hop needed)
  - Step 5 (Local): PublishSatuSehatSync — publishes async RabbitMQ message to `backbone.canonical_visit.created` for SATU SEHAT integration service to consume

### Task 5.2: Consultation FinalizeConsultationRecord step handler
- **Type**: HUMAN
- **Notes**: Abdul implements. Handler receives sign-off trigger, calls consultation service to lock SOAP, calls `ReportStepResult(COMPLETED, {diagnosis_codes, treatment_summary})`. Compensation: unlock SOAP / restore draft state.

---
### 🔲 CHECKPOINT: DoctorSignOff E2E
**Review**: Full sign-off flow test. Visit transitions from in-progress → signed-off. Invoice created in POS. Canonical record in backbone. SATU SEHAT sync message published.
**Resume**: N/A — plan complete after this checkpoint.
---

## 6. Key Interrelations for Navigation

```
DEPENDENCY CHAIN
────────────────
saga-pattern-extended plan (COMPLETE)
  └── Framework core works: orchestrator, continuator, RedisReplyStore
  └── Builders migrated: patient, pharmacy_sale, data_sync
  └── Proto files written: saga_callback.proto, saga_status.proto, canonical_visit.proto
  └── canonical_visit.proto GENERATED: canonicalvisitpb/*.pb.go exist
  └── saga_callback.proto NOT GENERATED: sagacallbackpb/ is empty ← BLOCKER

consultation-extraction plan Phase 1 (code complete, ops pending)
  └── 12 modules moved to wellmed-consultation
  └── Gateway routes consultation calls to :50052
  └── Stubs in place: saga_callback_client, canonical_visit_client, consumer.go handlers

THIS PLAN (visit-saga-wiring)
  └── Phase 1: generate sagacallbackpb + sagastatuspb (Abdul HUMAN)
  └── Phase 2: wire handlers into backbone gRPC server (AI)
  └── Phase 3: replace consultation stubs with real gRPC calls (AI)
  └── Phase 4: CreateVisit saga builder + consultation step handler (AI + manual test)
  └── Phase 5: DoctorSignOff saga builder (Abdul HUMAN)
```

**ADR-006 boundary rules enforced by this plan:**
- Backbone NEVER calls Consultation directly — only publishes RabbitMQ triggers
- Consultation NEVER calls Backbone except `CanonicalVisitService.WriteVisitRecord` at sign-off
- Backbone mediates all data flow — Consultation receives full payload in trigger, never fetches from Backbone
- Visit state stays in Consultation until sign-off; Backbone only sees outcomes

**Where visit_id comes from:**
- Backbone triggers `record_visit_in_consultation` step
- Consultation creates visit, generates ULID `visit_id`
- Consultation calls back with `{visit_id}` in payload
- Backbone's `continuator.HandleCallback` stores `visit_id` in `SagaContext` via `SagaContext.Set("visit_id", ...)`
- Subsequent steps (POS billing, sign-off) receive `visit_id` in their trigger payload via `WithTriggerPayload` builder func reading from `SagaContext`

**Patient saga vs CreateVisit saga:**
- `patient` saga (existing) is registration + first-visit-creation in one — used for NEW patients
- `create_visit` saga (this plan) is pure visit creation — used for RETURNING patients
- Both are orchestrated from Backbone; both thread `visit_id` into subsequent POS/pharmacy steps

---

## 7. Acceptance Criteria

| Phase | Criteria |
|-------|---------|
| Proto generation | `sagacallbackpb/*.pb.go` and `sagastatuspb/*.pb.go` exist. `go build ./...` passes. |
| Backbone handlers | SagaCallbackService, SagaStatusService, CanonicalVisitService all registered and responding. |
| Consultation clients | `ReportStepResult` and `WriteVisitRecord` make real gRPC calls to backbone. |
| CreateVisit E2E | POST create-visit → `saga_id` returned → Consultation creates visit → backbone marks saga completed → GET status returns `{status: completed, result: {visit_id}}`. |
| DoctorSignOff E2E | Sign-off → SOAP finalized → POS invoice created → canonical record in backbone → SATU SEHAT sync message queued. |

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 05 Mar 2026 | Alex + Claude | Initial plan — captures all visit wiring work as concrete tasks. Identified by user as missing from existing TODOs. Prerequisite: saga-pattern-extended plan complete (✅) and consultation-extraction Phase 1 code complete (✅). |
