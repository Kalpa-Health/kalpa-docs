# Plan: Saga Package Rework

**Version:** 1.1
**Date:** 05 Mar 2026
**Author:** Alex + Claude
**Status:** Ready to execute
**PRD:** `kalpa-docs/development/saga-pattern-extended-PRD.md`

## Related Docs
- `kalpa-docs/adrs/ADR-005-saga-orchestration.md`
- `kalpa-docs/adrs/ADR-006-domain-service-boundaries.md`
- `kalpa-docs/development/saga-pattern-extended-PRD.md`
- `wellmed-backbone/internal/saga/README.md`
- `wellmed-consultation/internal/clients/saga_callback_client.go`
- `wellmed-consultation/internal/queue/consumer.go`

---

## Context

The backbone saga framework (`internal/saga/`) uses dual-mode (ExecuteSync/ExecuteAsync) with blocking gRPC step strategy — both rejected by ADR-005. The consultation service already expects `wellmed.saga` exchange and has stub clients for `SagaCallbackService` and `CanonicalVisitService` awaiting Phase 2 wiring.

**Key consultation repo findings:**
- Exchange `wellmed.saga` confirmed (already set in consultation)
- `canonical_visit.proto` must use structured fields (not `bytes fhir_payload`) — matches consultation stub
- StepConsumer in consultation routes by routing key → handlers
- Consultation's saga entity already uses lowercase state values

---

## Phase 1: Proto Definitions

### Task 1.1: Write saga_callback.proto
- **Type**: AI
- **Input**: `kalpa-docs/development/saga-pattern-extended-PRD.md` §3.3.1
- **Action**: Write proto file at `wellmed-backbone/proto/saga_callback.proto`. Service: `SagaCallbackService.ReportStepResult`. Include `StepStatus` enum (UNSPECIFIED/COMPLETED/FAILED/REJECTED). Fields: saga_id, step_id, status, payload(bytes), error_code, error_message, idempotency_key, tenant.
- **Output**: `wellmed-backbone/proto/saga_callback.proto`
- **Acceptance**: File exists with correct package, go_package option, enum, and message definitions.
- **Notes**: `go_package = "sagacallbackpb/"` to match codebase convention. Package name `saga.callback.v1`.

### Task 1.2: Write canonical_visit.proto
- **Type**: AI
- **Input**: PRD §3.3.2 + `wellmed-consultation/internal/clients/canonical_visit_client.go`
- **Action**: Write proto at `wellmed-backbone/proto/canonical_visit.proto`. Use STRUCTURED fields (not bytes fhir_payload) based on consultation stub: visit_id, patient_id, tenant, doctor_id, diagnosis_codes(repeated string), treatment_summary, sign_off_at. Add canonical_record_id to response.
- **Output**: `wellmed-backbone/proto/canonical_visit.proto`
- **Acceptance**: File exists with correct service, request/response messages using structured fields.
- **Notes**: `go_package = "canonicalvisitpb/"`. Package name `canonical.visit.v1`. This OVERRIDES PRD's fhir_payload approach based on consultation stub evidence.

### Task 1.3: Write saga_status.proto
- **Type**: AI
- **Input**: PRD §3.3.3
- **Action**: Write proto at `wellmed-backbone/proto/saga_status.proto`. Service: `SagaStatusService.GetStatus`. Request: saga_id, tenant. Response: saga_id, status, result_payload(bytes), error_code, error_message.
- **Output**: `wellmed-backbone/proto/saga_status.proto`
- **Acceptance**: File exists with correct service definition.
- **Notes**: `go_package = "sagastatuspb/"`. Package name `saga.status.v1`.

---
### 🔲 CHECKPOINT: Proto Review
**Review**: Check all 3 proto files at `wellmed-backbone/proto/`. Verify field names match consultation stubs. Confirm structured fields in canonical_visit.proto.
**Resume**: "continue the saga-pattern-extended plan" or "run the saga plan Phase 2"
---

## Phase 2: Framework Core

### Task 2.1: Update model.go
- **Type**: AI
- **Input**: `wellmed-backbone/internal/saga/model.go`, PRD §3.4.6
- **Action**: Replace uppercase states (INITIATED, IN_PROGRESS, etc.) with lowercase (pending, in_flight, completed, failed, compensated). Remove LOCAL_STEPS_COMPLETED and WAITING_REMOTE_RESPONSE. Remove SagaExecutionMode type and constants. Add step-level state constants (StepStatePending, StepStateInFlight, etc.).
- **Output**: Modified `wellmed-backbone/internal/saga/model.go`
- **Acceptance**: No uppercase state constants remain. No SagaExecutionMode. Step states defined.

### Task 2.2: Update step.go
- **Type**: AI
- **Input**: `wellmed-backbone/internal/saga/step.go`
- **Action**: Remove StepStrategyGRPC constant and GRPC() builder method. Rename StepStrategyEvent → StepStrategyRemote. Rename Event() builder to Remote() with routing keys derived from convention (no explicit topic params). Keep EventPayload as TriggerPayload. Remove ReplyTopic field (callbacks now come via gRPC, not RabbitMQ). Add ParallelStepGroup struct. Add Parallel() step builder method.
- **Output**: Modified `wellmed-backbone/internal/saga/step.go`
- **Acceptance**: No StepStrategyGRPC. Remote() method exists. ParallelStepGroup defined. No blocking logic.

### Task 2.3: Update orchestrator.go
- **Type**: AI
- **Input**: `wellmed-backbone/internal/saga/orchestrator.go`, `model.go` (updated), `step.go` (updated)
- **Action**: Remove ExecuteSync() and ExecuteAsync(). Add single Execute() that: (1) creates saga in 'pending' state, (2) transitions to 'in_flight', (3) runs all Local steps sequentially, (4) publishes first Remote/Parallel step trigger(s) to RabbitMQ using routing key `saga.{saga_type}.{step_name}.trigger`, (5) returns saga_id immediately. Update publishStepEvent to use new routing key format. Update compensateSteps to use new routing keys for Remote compensation.
- **Output**: Modified `wellmed-backbone/internal/saga/orchestrator.go`
- **Acceptance**: No ExecuteSync/ExecuteAsync. Single Execute() returning (sagaID, error). Routing keys use new convention.

### Task 2.4: Update continuator.go
- **Type**: AI
- **Input**: `wellmed-backbone/internal/saga/continuator.go`, `model.go` (updated)
- **Action**: Add STEP_STATUS_REJECTED handling: on REJECTED, increment retry_count and re-publish the step trigger (retry) instead of compensating. Add retry backoff logic (exponential: base 1s, max 5 retries for internal-fast category). Update state references to use new lowercase constants. HandleReply() becomes HandleCallback() to reflect gRPC callback source.
- **Output**: Modified `wellmed-backbone/internal/saga/continuator.go`
- **Acceptance**: REJECTED path retries without compensation. FAILED path compensates. State constants updated.

### Task 2.5: Update repository.go — add SagaStep model and StepRepository
- **Type**: AI
- **Input**: `wellmed-backbone/internal/saga/repository.go`
- **Action**: Add SagaStep struct with fields: ID, SagaID, StepName, Status, StartedAt, CompletedAt, ErrorCode, Result(json.RawMessage), RetryCount, CreatedAt, UpdatedAt. Add StepRepository interface: CreateStep, UpdateStep, FindStepsBySagaID, FindStep. Update Repository interface: remove FindPendingAsync (replaced by step-level tracking), add FindByIDForUpdate.
- **Output**: Modified `wellmed-backbone/internal/saga/repository.go`
- **Acceptance**: SagaStep struct defined. StepRepository interface defined.

### Task 2.6: Replace InMemoryReplyStore with RedisReplyStore
- **Type**: AI
- **Input**: `wellmed-backbone/internal/saga/infrastructure/reply_store.go`, `wellmed-backbone/internal/config/redis.go`
- **Action**: Replace InMemoryReplyStore with RedisReplyStore using `github.com/go-redis/redis/v8`. Remove WaitForReply() from ReplyStore interface (no blocking). Add CacheSagaStatus(ctx, sagaID, status string, resultJSON json.RawMessage, ttl time.Duration) error. Add GetCachedSagaStatus(ctx, sagaID string) (*CachedSagaStatus, error). Add DeleteCachedStatus(ctx, sagaID string) error. Keep Store/Get/Delete for step-level results. Redis key format: `saga:{sagaID}:status` (TTL 1 hour).
- **Output**: Modified `wellmed-backbone/internal/saga/infrastructure/reply_store.go`
- **Acceptance**: InMemoryReplyStore removed. RedisReplyStore compiles. No WaitForReply anywhere.

### Task 2.7: Update event_consumer.go
- **Type**: AI
- **Input**: `wellmed-backbone/internal/saga/infrastructure/event_consumer.go`
- **Action**: Update SetupRabbitMQ to use new exchange name `wellmed.saga`, DLX `dlx.wellmed.saga`. Remove old hardcoded queue declarations (backbone.transaction.reply etc.). Add function to declare queues dynamically based on registered saga step routing keys. The consumer no longer processes reply messages from modules (callbacks come via gRPC) — simplify to handle DLQ/retry scenarios only.
- **Output**: Modified `wellmed-backbone/internal/saga/infrastructure/event_consumer.go`
- **Acceptance**: Exchange is `wellmed.saga`. Old queue names removed. SetupRabbitMQ accepts queue config dynamically.

### Task 2.8: Update rabbitmq_publisher.go
- **Type**: AI
- **Input**: `wellmed-backbone/internal/saga/infrastructure/rabbitmq_publisher.go`
- **Action**: Update default exchange to `wellmed.saga`. Add PublishSagaStep(ctx, sagaType, stepName, direction string, payload interface{}) error as a typed saga-aware publish method that constructs routing key `saga.{sagaType}.{stepName}.{direction}`. Keep Publish() and PublishWithKey() for backward compat.
- **Output**: Modified `wellmed-backbone/internal/saga/infrastructure/rabbitmq_publisher.go`
- **Acceptance**: Default exchange is wellmed.saga. PublishSagaStep method exists.

### Task 2.9: Write callback_handler.go
- **Type**: AI
- **Input**: PRD §3.4.2, `wellmed-backbone/internal/saga/continuator.go` (updated)
- **Action**: Create `wellmed-backbone/internal/saga/handler/callback_handler.go`. Define local types: CallbackRequest (SagaID, StepID, Status int32, Payload []byte, ErrorCode, ErrorMessage, IdempotencyKey, Tenant string), CallbackResponse (Accepted bool). Implement CallbackHandler struct with HandleCallback(ctx, req) method that: validates idempotency key in Redis, updates saga_steps table, calls continuator.HandleCallback(), updates Redis saga status cache. NOTE: gRPC server registration requires protoc — wire in application.go after running protoc.
- **Output**: `wellmed-backbone/internal/saga/handler/callback_handler.go`
- **Acceptance**: File compiles. HandleCallback logic validates idempotency, updates step state, calls continuator.

### Task 2.10: Write status_handler.go
- **Type**: AI
- **Input**: PRD §3.4.4, `wellmed-backbone/internal/saga/infrastructure/reply_store.go` (updated)
- **Action**: Create `wellmed-backbone/internal/saga/handler/status_handler.go`. Define StatusRequest (SagaID, Tenant string), StatusResponse (SagaID, Status, ResultPayload json.RawMessage, ErrorCode, ErrorMessage string). Implement StatusHandler with GetStatus(ctx, req) that: tries Redis cache first (fast path <50ms), falls back to PostgreSQL. Returns status, result_payload, error fields.
- **Output**: `wellmed-backbone/internal/saga/handler/status_handler.go`
- **Acceptance**: File compiles. GetStatus uses Redis fast path with PG fallback.

### Task 2.11: Update setup.go
- **Type**: AI
- **Input**: `wellmed-backbone/internal/saga/infrastructure/setup.go`
- **Action**: Update Config struct to include RedisClient *redis.Client. Update Initialize() to accept redis.Client and use RedisReplyStore. Remove InMemoryReplyStore usage. Update Start() to only start necessary consumers. Add RegisterSagaBuilder convenience method. Remove RegisterAsyncSaga (saga type now has single Build() method).
- **Output**: Modified `wellmed-backbone/internal/saga/infrastructure/setup.go`
- **Acceptance**: Setup compiles with Redis client. No InMemoryReplyStore references.

---
### 🔲 CHECKPOINT: Framework Review
**Review**: Review all 11 updated/created framework files. Check that: (1) no ExecuteSync/ExecuteAsync refs, (2) no StepStrategyGRPC refs, (3) lowercase state constants throughout, (4) RedisReplyStore compiles, (5) callback_handler.go and status_handler.go compile.
**Resume**: "continue the saga-pattern-extended plan"
---

## Phase 3: Database & Gateway

### Task 3.1: Write saga_steps DB migration
- **Type**: AI
- **Input**: PRD §3.4.5
- **Action**: Create `wellmed-backbone/internal/saga/infrastructure/migrations/002_create_saga_steps_table.sql`. Schema: id (TEXT PK), saga_id (TEXT NOT NULL), step_name (TEXT NOT NULL), status (TEXT DEFAULT 'pending'), started_at (TIMESTAMPTZ), completed_at (TIMESTAMPTZ), error_code (TEXT), result (JSONB), retry_count (INT DEFAULT 0), created_at/updated_at (TIMESTAMPTZ). Add indexes on saga_id and status. Add comment on table about tenant database requirement. Update migrate_all_tenants.sh to run new migration.
- **Output**: `wellmed-backbone/internal/saga/infrastructure/migrations/002_create_saga_steps_table.sql`
- **Acceptance**: Valid SQL. Indexes created. Consistent with existing 001 migration style.

### Task 3.2: Add PostgresStepRepository
- **Type**: AI
- **Input**: `wellmed-backbone/internal/saga/infrastructure/postgres_repository.go`, StepRepository interface from Task 2.5
- **Action**: Add PostgresStepRepository struct implementing StepRepository interface: CreateStep, UpdateStep, FindStepsBySagaID, FindStep (by sagaID + stepName). Add to Initialize() in setup.go.
- **Output**: Modified `wellmed-backbone/internal/saga/infrastructure/postgres_repository.go`
- **Acceptance**: All StepRepository interface methods implemented.

### Task 3.3: Add gateway saga status endpoint
- **Type**: AI
- **Input**: `wellmed-gateway-go/internal/route/routes.go`, `wellmed-gateway-go/internal/route/api.go`, PRD §3.4.4
- **Action**: Create `wellmed-gateway-go/internal/domain/saga/handler/saga.go` with GetSagaStatus handler. Create `wellmed-gateway-go/internal/domain/saga/service/saga.go` as HTTP client to backbone's SagaStatusService (stub for now — calls backbone via gRPC after protoc, or via HTTP fallback). Create `wellmed-gateway-go/internal/domain/saga/wires.go`. Register route `GET /api/v1/saga/:saga_id/status` in routes.go under authorized group. Wire module in api.go.
- **Output**: 3 new files + 2 modified routes files
- **Acceptance**: Gateway route exists. Handler returns JSON with saga_id, status, result_payload, error fields.

---
### 🔲 CHECKPOINT: DB + Gateway Review
**Review**: Check migration SQL is valid. Check gateway handler compiles and route is registered. Test `GET /api/v1/saga/:saga_id/status` returns expected JSON shape.
**Resume**: "continue the saga-pattern-extended plan"
---

## Phase 4: Migrate Existing Builders

### Task 4.1: Migrate pharmacy_sale saga builder
- **Type**: AI
- **Input**: `wellmed-backbone/internal/domain/pharmacy_sale/saga/builder.go`
- **Action**: Replace BuildSync() and BuildAsync() with single Build() method. Steps 1-4 remain Local. Step 5 (CreatePOSTransaction) changes from GRPC to Remote — remove transactionRpc dependency, use Remote() builder with TriggerPayload using buildPOSTransactionEvent. Routing key: `saga.pharmacy_sale.create_pos_transaction.trigger`. Compensate key: `saga.pharmacy_sale.create_pos_transaction.compensate`. Remove grpc client import. Update NewPharmacySaleSagaBuilder signature (remove transactionRpc param).
- **Output**: Modified `wellmed-backbone/internal/domain/pharmacy_sale/saga/builder.go`
- **Acceptance**: Single Build() method. No GRPC strategy. No transactionRpc import. Routing keys follow convention.

### Task 4.2: Migrate patient saga builder
- **Type**: AI
- **Input**: `wellmed-backbone/internal/domain/patient/saga/builder.go`
- **Action**: Replace BuildSync() and BuildAsync() with single Build() method. Review all steps — convert any GRPC steps to Remote, keep Local steps as Local. Update routing keys to `saga.patient.{step_name}.trigger` convention. Ensure compensation steps use `saga.patient.{step_name}.compensate`.
- **Output**: Modified `wellmed-backbone/internal/domain/patient/saga/builder.go`
- **Acceptance**: Single Build() method. Routing keys follow new convention. No GRPC strategy.

### Task 4.3: Migrate data_sync saga builder
- **Type**: AI
- **Input**: `wellmed-backbone/internal/domain/data_sync/saga/builder.go`
- **Action**: Replace BuildAsync() with Build(). Update routing keys from old convention (pos.data.sync etc.) to new convention (saga.data_sync.{step_name}.trigger). Remove old-style topic params from Event() calls — use Remote() instead.
- **Output**: Modified `wellmed-backbone/internal/domain/data_sync/saga/builder.go`
- **Acceptance**: Single Build() method. All routing keys follow new convention.

---
### 🔲 CHECKPOINT: Builder Migration Review
**Review**: Verify all 3 builders have single Build() method. Search codebase for any remaining ExecuteSync/ExecuteAsync/BuildSync/BuildAsync/StepStrategyGRPC calls. Check that builders reference correct routing keys.
**Resume**: "continue the saga-pattern-extended plan"
---

## Phase 5: Tests

### Task 5.1: Unit tests for framework core
- **Type**: HUMAN
- **Notes**: Abdul writes unit tests for refactored saga framework. Target: 80%+ coverage for orchestrator.go, continuator.go, model.go, step.go. Use testify mocks for Repository, MessagePublisher, ReplyStore. Test REJECTED retry path, FAILED compensation path, parallel step group triggering.

---
### 🔲 CHECKPOINT: Test Coverage
**Review**: Run `go test ./internal/saga/... -cover`. Verify ≥80% coverage. All existing tests pass. No regressions.
**Resume**: "continue the saga-pattern-extended plan Phase 6"
---

## Phase 6: New Saga Builders (post-framework-stable)

### Task 6.1: CreateVisit saga builder
- **Type**: HUMAN
- **Notes**: Abdul implements CreateVisit saga builder. Backbone publishes to `saga.create_visit.record_visit_in_consultation.trigger`. Consultation's StepConsumer must register handler for this routing key. On completion, consultation calls SagaCallbackService.ReportStepResult. Requires protoc generation of proto files from Phase 1 and consultation Phase 2 wiring.

### Task 6.2: DoctorSignOff saga builder
- **Type**: HUMAN
- **Notes**: Abdul implements DoctorSignOff. Chain: backbone → consultation (finalize SOAP) → pos (trigger billing) → satu-sehat.

---
### 🔲 CHECKPOINT: Phase 6 Ready Gate
**Review**: Framework must be stable (Phase 2-4 complete), proto generated, saga_steps table migrated. Only begin Phase 6 when E2E tests from Phase 5 pass.
**Resume**: "continue the saga-pattern-extended plan Phase 6"
---

## Phase 7: Documentation Updates

### Task 7.1: Update wellmed-system-architecture.md
- **Type**: AI
- **Input**: `kalpa-docs/architecture/wellmed-system-architecture.md`, PRD §3.5.1 (SA-1 through SA-6)
- **Action**: Apply all 6 doc changes from PRD §3.5.1 to wellmed-system-architecture.md. Update version to 1.2.
- **Output**: Modified `kalpa-docs/architecture/wellmed-system-architecture.md`
- **Acceptance**: All SA-1 through SA-6 changes applied. No stale references to sync execution mode.

### Task 7.2: Update services/backbone.md
- **Type**: AI
- **Input**: `kalpa-docs/services/backbone.md`, PRD §3.5.2 (BB-1 through BB-4)
- **Action**: Apply all 4 doc changes from PRD §3.5.2. Update version to 1.1.
- **Output**: Modified `kalpa-docs/services/backbone.md`
- **Acceptance**: All BB-1 through BB-4 changes applied. Version updated.

---
### 🔲 CHECKPOINT: Doc Review (CTO)
**Review**: Hamzah reviews updated wellmed-system-architecture.md and services/backbone.md. Verify ADR-002/005/006 accurately reflected. No stale sync mode or shared /pkg/saga references.
**Resume**: N/A — plan complete after CTO approval.
---
