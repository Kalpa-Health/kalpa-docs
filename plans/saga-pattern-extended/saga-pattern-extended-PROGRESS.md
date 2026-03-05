# Progress Log: Saga Package Rework

## Session: 2026-03-05T00:00:00Z

### Phase 0: Pre-flight
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **Paths verified**:
  - ✅ `kalpa-docs/adrs/ADR-005-saga-orchestration.md`
  - ✅ `kalpa-docs/adrs/ADR-006-domain-service-boundaries.md`
  - ✅ `kalpa-docs/development/saga-pattern-extended-PRD.md`
  - ✅ `wellmed-backbone/internal/saga/` (all framework files)
  - ✅ `wellmed-consultation/internal/clients/saga_callback_client.go`
  - ✅ `wellmed-consultation/internal/queue/consumer.go`
  - ⚠️ `kalpa-docs/architecture/wellmed-system-architecture.md` — actual path is `kalpa-docs/wellmed-system-architecture.md` (plan updated reference)
  - ✅ `kalpa-docs/services/backbone.md`
- **Issues**: System architecture doc is at `kalpa-docs/wellmed-system-architecture.md` (no subdirectory). Phase 7 tasks reference correct path.

---

## Session: 2026-03-05T (resumed)

### Task 2.5.1: Fix missing persistSagaTerminal() in continuator
- **Status**: ✅ DONE
- **What was done**: Added `persistSagaTerminal(ctx, saga, resultPayload)` method to `continuator.go`. Writes to PostgreSQL first (source of truth), then updates Redis cache. Called by `handleCompleted` and `handleFailed` for all terminal transitions.
- **Files modified**: `wellmed-backbone/internal/saga/continuator.go`
- **Issues**: None

### Task 2.6: Replace InMemoryReplyStore with RedisReplyStore
- **Status**: ✅ DONE
- **What was done**: Rewrote `infrastructure/reply_store.go` as `RedisReplyStore` using `github.com/go-redis/redis/v8`. Implements full `ReplyStore` interface: `CacheSagaStatus` (TTL 1h in-flight / 24h terminal), `GetCachedSagaStatus` (returns nil,nil if not found for PG fallback), `DeleteCachedStatus`, `StoreIdempotencyKey`, `CheckIdempotencyKey`. Redis key patterns: `saga:{sagaID}:status`, `saga:idem:{key}`.
- **Files modified**: `wellmed-backbone/internal/saga/infrastructure/reply_store.go`
- **Issues**: None

### Task 2.7: Update event_consumer.go
- **Status**: ✅ DONE
- **What was done**: Removed hardcoded legacy queues from `SetupRabbitMQ` (backbone no longer owns those queues). Updated `handleMessage` to parse both new `StepCallback` format and legacy `StepReply` (via `stepReplyToCallback` adapter). Removed `replyStore` dependency from consumer. Exchange is `wellmed.saga` (via `DefaultSagaExchange` constant in setup.go).
- **Files modified**: `wellmed-backbone/internal/saga/infrastructure/event_consumer.go`
- **Issues**: None

### Task 2.8: Update rabbitmq_publisher.go
- **Status**: ✅ DONE (exchange is now passed via DefaultSagaExchange = "wellmed.saga" in setup.go; publisher itself unchanged as it accepts exchange at construction)
- **Files modified**: None (publisher already parameterized)
- **Issues**: None

### Task 2.9: Write callback_handler.go
- **Status**: ✅ DONE
- **What was done**: Created `saga/handler/callback_handler.go` with `CallbackHandler`. Implements idempotency check, tenant context injection, proto→StepCallback conversion, continuator delegation, idempotency key storage. Uses local request/response types (proto wiring deferred to Phase 1 HUMAN task after protoc runs).
- **Files modified**: `wellmed-backbone/internal/saga/handler/callback_handler.go` (NEW)
- **Issues**: None

### Task 2.10: Write status_handler.go
- **Status**: ✅ DONE
- **What was done**: Created `saga/handler/status_handler.go` with `StatusHandler`. Redis fast path → PostgreSQL fallback pattern. Returns saga status, result payload (on completion), error fields.
- **Files modified**: `wellmed-backbone/internal/saga/handler/status_handler.go` (NEW)
- **Issues**: None

### Task 2.11: Update setup.go
- **Status**: ✅ DONE
- **What was done**: Rewrote `setup.go`. `Initialize()` now accepts `*redis.Client` instead of config struct. Creates `RedisReplyStore`, `PostgresStepRepository`, `PostgresRepository`, `RabbitMQPublisher`. Passes step repository to both orchestrator and continuator. `DefaultSagaExchange = "wellmed.saga"`. Removed `RegisterAsyncSaga` → replaced with `RegisterSagaBuilder`.
- **Files modified**: `wellmed-backbone/internal/saga/infrastructure/setup.go`
- **Issues**: None

### Task 3.1: DB migration for saga_steps
- **Status**: ✅ DONE
- **What was done**: Created `002_create_saga_steps_table.sql`. Columns: id, saga_id (FK → sagas), step_name, status, started_at, completed_at, error_code, result (JSONB), retry_count, created_at, updated_at. UNIQUE constraint on (saga_id, step_name). Indexes on saga_id, status, and composite (saga_id, step_name).
- **Files modified**: `wellmed-backbone/internal/saga/infrastructure/migrations/002_create_saga_steps_table.sql` (NEW)
- **Issues**: None

### Task 3.2: PostgresStepRepository
- **Status**: ✅ DONE
- **What was done**: Added `PostgresStepRepository` to `postgres_repository.go` implementing all 5 methods of `StepRepository` interface. Also added `TenantAwareStepRepository` to `tenant_repository.go`. Also fixed `postgres_repository.go` to remove `FindPendingAsync` (used removed state constants) and add `FindByIDForUpdate` (SELECT ... FOR UPDATE). Updated `TenantAwareSagaRepository` to remove `FindPendingAsync` delegation, add `FindByIDForUpdate`.
- **Files modified**: `postgres_repository.go`, `tenant_repository.go`
- **Issues**: None

### Task 4.1–4.3: Migrate existing saga builders
- **Status**: ✅ DONE
- **What was done**:
  - `pharmacy_sale/saga/builder.go`: Removed `BuildSync()`, renamed `BuildAsync()` → `Build()`. Step 5 (CreatePOSTransaction) changed from `GRPC()/Event(...)` to `Remote().WithTriggerPayload().WithCompensatePayload().WithRetryCategory(RetryExternal)`. Added `buildPOSCompensateEvent` payload builder. Updated `pharmacy_sale/service/pharmacy_sale.go`: removed `executionMode`, removed `executeSync()`, `executeAsync()` → single path calling `Execute()`. Updated `wires.go` to remove `SagaExecutionSync` param.
  - `data_sync/saga/builder.go`: Renamed `BuildAsync()` → `Build()`. Replaced all `.Event(...)` chains with `.Remote().WithTriggerPayload()`.
  - `patient/saga/builder.go`: Renamed `BuildAsync(payload)` → `Build(payload)`, updated `BuildFromSaga`. Replaced all `.Event(...)` chains with `.Remote().WithTriggerPayload().WithRetryCategory(RetryExternal)`.
  - Updated `patient/service/patient.go`: `Build(payload)` + `Execute()`.
  - Updated `application.go`: New saga wiring with `RedisReplyStore`, `TenantAwareStepRepository`, updated constructors, `DefaultSagaExchange`, saga type key `"patient"` (lowercase).
- **Files modified**: 7 files
- **Issues**: None — `go build ./...` passes cleanly

### Task 5.1–5.2: Unit tests for saga framework
- **Status**: ✅ DONE
- **What was done**: Created `orchestrator_test.go` and `continuator_test.go` in `internal/saga/`. Tests cover all core paths: local step execution, local compensation, remote step trigger, REJECTED retry, terminal idempotency, backoff calculation, routing key format, state machine helpers, `SagaContext` get/set/save, `findNextRemoteStep`. Fixed `seedSaga` helper (was accidentally dropped during rewrite — added to `orchestrator_test.go`). All 15 tests pass.
- **Files modified**: `wellmed-backbone/internal/saga/orchestrator_test.go` (NEW), `wellmed-backbone/internal/saga/continuator_test.go` (NEW)
- **Issues**: None — `go test ./internal/saga/...` passes cleanly

---

## Session: 2026-03-05T (Phase 7 — Doc Updates)

### Task 7.1: Apply SA-1–SA-6 to wellmed-system-architecture.md
- **Status**: ✅ DONE (was already applied in a prior session — doc is at v1.2 with all 6 changes)
- **Files modified**: None — already complete
- **Issues**: None

### Task 7.2: Apply BB-1–BB-4 to services/backbone.md
- **Status**: ✅ DONE
- **What was done**: Updated `services/backbone.md` from v1.0 → v1.1. BB-1: §1 overview updated to three-role model (canonical owner, saga orchestrator, auth/config). BB-2: §3 responsibilities rewritten — removed stale SYNC/business-logic framing, added canonical visit record reception, POS pass-through clarification. BB-3: §4 gRPC catalog split into three subsections (staying in Backbone, pending extraction to consultation, new saga services with ADR references). BB-4: §6 saga framework replaced with async-only description including callback envelope table, REJECTED/FAILED distinction, RabbitMQ convention, and per-step state machine.
- **Files modified**: `kalpa-docs/services/backbone.md`
- **Issues**: None

### Task 7.3: Close open TODOs in ADR-005
- **Status**: ✅ DONE
- **What was done**: All four open TODO items in `ADR-005-saga-orchestration.md` marked resolved with concrete resolution references. §3.3 retry TODO → resolved by PRD §3.2.2 (three retry profiles). §3.3 timeout TODO → resolved by PRD §3.2.3. §5.5 naming convention → resolved by PRD §3.2.4 (wellmed.saga exchange, routing key pattern). §5.6 gRPC callback contract → resolved by PRD §3.2.5 and §3.3.1 (SagaCallbackService.ReportStepResult). ADR version bumped to 1.0.
- **Files modified**: `kalpa-docs/adrs/ADR-005-saga-orchestration.md`
- **Issues**: None
