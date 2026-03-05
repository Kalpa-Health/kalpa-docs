# Progress Log: Visit Saga Wiring — Consultation Phase 2

## Session: 2026-03-05T00:00:00Z

### Phase 0: Pre-flight
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **Paths verified**:
  - `kalpa-docs/adrs/ADR-002-consultation-service-extraction.md` ✅
  - `kalpa-docs/adrs/ADR-005-saga-orchestration.md` ✅
  - `kalpa-docs/adrs/ADR-006-domain-service-boundaries.md` ✅
  - `kalpa-docs/development/saga-pattern-extended-PRD.md` ✅
  - `wellmed-backbone/proto/saga_callback.proto` ✅
  - `wellmed-backbone/proto/saga_status.proto` ✅
  - `wellmed-backbone/proto/canonical_visit.proto` ✅
  - `wellmed-backbone/internal/saga/` ✅
  - `wellmed-consultation/internal/clients/saga_callback_client.go` ✅
  - `wellmed-consultation/internal/clients/canonical_visit_client.go` ✅
  - `wellmed-consultation/internal/queue/consumer.go` ✅
- **Plan status**: Ready to execute ✅
- **Issues**: None

### Task 1.1: Generate sagacallbackpb
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**: Ran `protoc --go_out=proto --go-grpc_out=proto proto/saga_callback.proto`. Installed `protoc-gen-go` and `protoc-gen-go-grpc` plugins first.
- **Files modified**: `proto/sagacallbackpb/saga_callback.pb.go`, `proto/sagacallbackpb/saga_callback_grpc.pb.go`
- **Issues**: None

### Task 1.2: Generate sagastatuspb
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**: Ran `protoc --go_out=proto --go-grpc_out=proto proto/saga_status.proto`.
- **Files modified**: `proto/sagastatuspb/saga_status.pb.go`, `proto/sagastatuspb/saga_status_grpc.pb.go`
- **Issues**: None

### Task 1.3: Verify canonical_visit.proto Go is current
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**: `proto/canonicalvisitpb/canonical_visit.pb.go` and `canonical_visit_grpc.pb.go` exist and match current proto. `go build ./...` clean.
- **Files modified**: None
- **Issues**: None

### Task 2.1: Wire SagaCallbackService into application.go
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**: Rewrote `callback_handler.go` to implement `sagacallbackpb.SagaCallbackServiceServer` (embed `UnimplementedSagaCallbackServiceServer`, `ReportStepResult` with proto types, tenant parsing from "db_name:schema" format). Removed old local stub types. Imported `sagacallbackpb` and `sagahandler` in `application.go`. Registered `SagaCallbackService` in gRPC server at §9.
- **Files modified**: `internal/saga/handler/callback_handler.go`, `internal/app/application.go`
- **Issues**: None

### Task 2.2: Wire SagaStatusService into application.go
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**: Rewrote `status_handler.go` to implement `sagastatuspb.SagaStatusServiceServer` (embed `UnimplementedSagaStatusServiceServer`, `GetStatus` with proto types). Imported `sagastatuspb` in `application.go`. Registered `SagaStatusService` in gRPC server at §9.
- **Files modified**: `internal/saga/handler/status_handler.go`, `internal/app/application.go`
- **Issues**: None

### Task 2.3: Wire CanonicalVisitService handler
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**: Implemented `WriteVisitRecord` — validates fields, parses RFC3339 sign_off_at, injects tenant context, persists to `canonical_visits` table, fires SATU SEHAT sync event via `wellmed.events` exchange (fire-and-forget; logs warning if exchange not yet declared). Created repository using `ConnectionManager.WithTenantTable`. Passed `connectionManager` and `sagaPublisher` into handler constructor. `go build ./...` and `go test ./internal/saga/...` pass.
- **Files modified**: `internal/domain/canonical_visit/handler/canonical_visit_handler.go`, `internal/domain/canonical_visit/repository/canonical_visit_repository.go` (new), `internal/domain/canonical_visit/migrations/001_create_canonical_visits_table.sql` (new), `internal/app/application.go`
- **Issues**: `wellmed.events` exchange not yet declared at startup — publish will warn but not fail. Needs exchange setup when SATU SEHAT service is ready.

### Task 3.1: Replace SagaCallbackClient stub with real gRPC client
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**: Copied `saga_callback.proto` from backbone into `wellmed-consultation/proto/`, ran protoc to generate `proto/sagacallbackpb/`. Replaced stub with real gRPC client using `sagacallbackpb.NewSagaCallbackServiceClient`. Maps `StepResultRequest` fields to proto fields. Added `sagaCallbackNoOp` fallback for connection failures. `BACKBONE_GRPC_ADDRESS` env var driven.
- **Files modified**: `internal/clients/saga_callback_client.go`, `proto/saga_callback.proto` (new), `proto/sagacallbackpb/saga_callback.pb.go` (new), `proto/sagacallbackpb/saga_callback_grpc.pb.go` (new)
- **Issues**: None

### Task 3.2: Replace CanonicalVisitClient stub with real gRPC client
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**: Copied `canonical_visit.proto` from backbone into `wellmed-consultation/proto/`, ran protoc to generate `proto/canonicalvisitpb/`. Replaced stub with real gRPC client using `canonicalvisitpb.NewCanonicalVisitServiceClient`. Maps `WriteVisitRecordRequest` fields including `DiagnosisCodes []string`. Added `canonicalVisitNoOp` fallback. `BACKBONE_GRPC_ADDRESS` env var driven.
- **Files modified**: `internal/clients/canonical_visit_client.go`, `proto/canonical_visit.proto` (new), `proto/canonicalvisitpb/canonical_visit.pb.go` (new), `proto/canonicalvisitpb/canonical_visit_grpc.pb.go` (new)
- **Issues**: None

### Task 4.1: CreateVisit saga builder in Backbone
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**: Created `internal/domain/visit/dto/create_visit.go` (CreateVisitRequest DTO) and `internal/domain/visit/saga/builder.go` (CreateVisitSagaBuilder). 2-step saga: Step 1 (Local) ValidateVisitPrerequisites checks patient + employee exist; Step 2 (Remote) RecordVisitInConsultation sends CreateVisitRequest as trigger payload, RetryExternal, timeout 60s. Routing keys: `saga.create_visit.recordvisitinconsultation.{trigger|compensate}`.
- **Files modified**: `internal/domain/visit/dto/create_visit.go` (new), `internal/domain/visit/saga/builder.go` (new)
- **Issues**: None

### Task 4.2: Wire CreateVisit builder in backbone application.go
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**: Added imports for `visitSaga`, `patientRepo`, `employeeRepo`, `employeeDomain`. Renamed `employee.NewModule` → `employeeDomain.NewModule`. Created builder with real repo instances. Registered `sagaContinuator.RegisterSagaBuilder("create_visit", createVisitBuilder.BuildFromSaga)`. `go build ./...` clean.
- **Files modified**: `internal/app/application.go`
- **Issues**: None

### Task 4.3: Consultation CreateVisit step handler
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**: Created `internal/queue/steps/create_visit_handler.go`. `Handle()` parses stepEnvelope, maps CreateVisitRequest → VisitPatientRequest, injects tenant context, calls `visitPatientService.Create`, reports COMPLETED with `{visit_id}` via SagaCallbackClient. `Compensate()` reads visit_id from sagaCtx, calls `visitPatientRepo.Delete`, reports COMPLETED. Registered both routing keys in `internal/app/application.go`. Updated `internal/clients/clients_test.go` to Phase 2 contract (removed Phase 1 `ErrNotImplemented` references). `go build ./...` and `go test ./...` pass.
- **Files modified**: `internal/queue/steps/create_visit_handler.go` (new), `internal/app/application.go`, `internal/clients/clients_test.go`
- **Issues**: None
