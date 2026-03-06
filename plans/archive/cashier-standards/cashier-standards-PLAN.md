# Cashier MS Standards Alignment Plan

**Version:** 1.0
**Date:** 06 March 2026
**Previous Version:** N/A
**Maintained by:** Alex
**Author:** Alex
**Status:** Ready to execute

### Key Changes v1.0
- Initial version — full standards alignment for `wellmed-cashier`

---

## Related Docs

- `kalpa-docs/adrs/ADR-005-saga-orchestration.md` — saga callback must be gRPC, not RabbitMQ
- `kalpa-docs/adrs/ADR-006-domain-service-boundaries.md` — no direct cross-service calls
- `kalpa-docs/adrs/ADR-007-per-service-auth-tokens.md` — per-service key pattern
- `kalpa-docs/adrs/ADR-009-canonical-archive-pattern.md` — cashier triggers CanonicalTransactionService at invoice settlement
- `kalpa-docs/development/conventions.md` — module naming, logging, IDs
- `kalpa-docs/services/backbone.md` — POS pass-through pattern; cashier is a backbone-proxied service
- `kalpa-docs/services/pharmacy.md` — reference for port sequencing and saga participant structure

---

## Background: Evaluation Findings

The `wellmed-cashier` repo was evaluated on 06 March 2026. It is a fork of the old `wellmed-pos` service
(`github.com/ingarso/pos`) that was renamed on GitHub but never updated internally. It has functional
scaffolding but does not conform to the standards set by `wellmed-consultation` and `wellmed-pharmacy`.

### Critical gaps identified

- Module path `github.com/ingarso/pos` — all imports reference the old repo under the wrong org
- Saga callback uses RabbitMQ reply (`backbone.transaction.reply`) instead of gRPC `SagaCallbackService.ReportStepResult` (ADR-005 §5.6 violation)
- RabbitMQ exchange `backbone.saga.events` and queue names `pos.transaction.*` do not follow ADR-005 §5.5 convention
- Default gRPC port `:50052` conflicts with `wellmed-consultation`; cashier port is `:50053`
- Logger is `logrus` — standard is `uber-go/zap` with JSON output
- GORM `AutoMigrate` on startup — not acceptable in production; must use versioned SQL migration files
- `godotenv.Load()` called inside `getEnv()` on every invocation — should be called once at startup
- No CI/CD workflows (`.github/` directory absent)
- No `CLAUDE.md`, no SSM manifest, no `kalpa-docs/services/cashier.md`
- No backbone gRPC client for saga callbacks
- No per-service auth token (ADR-007)
- Zero tests
- Service still identifies itself as "POS" in logs, binary name, schema cluster name
- `go.mod` declares `go 1.25.6` (non-existent version)
- Debug `fmt.Println` left in production code
- Two disconnected entity types for the same domain concept (`Consument` in schema/entity vs `Consumer` in domain/consumer/entity)
- Dead code in `pkg/transaction.go` (unused utility, superseded by `TransactionManager`)

### What is good and carries forward

- Multi-tenant `ConnectionManager` pattern (structurally sound, mirrors consultation)
- ULID for all entity IDs
- Graceful shutdown correctly implemented
- GORM `WithTx` repository pattern for transaction override
- RabbitMQ auto-reconnect with context cancellation
- Multi-stage Dockerfile structure
- Four-domain structure (transaction, billing, invoice, payment_summary) is a reasonable starting point

---

## Port Assignment

Per pharmacy extraction plan §1.3 (the definitive port sequence):

| Service | Port |
|---------|------|
| `wellmed-backbone` | `:50051` |
| `wellmed-consultation` | `:50052` |
| `wellmed-cashier` | `:50053` (this plan) |
| `wellmed-pharmacy` | `:50054` |

---

## Plan Structure

**Phase 1 — Standards Alignment** (this document; execute first)

Complete overhaul of the repo to meet the standards set by consultation and pharmacy. No new features.
Every file is touched. At the end of Phase 1, the service builds cleanly, passes tests, follows all ADRs,
and is ready for CI bootstrap and merge to `main`.

**Phase 2 — Feature Completion** (separate document; follows Phase 1)

Discussion and implementation of: full billing domain, ADR-009 canonical transaction integration,
backbone proxy wiring, saga types for invoice/payment flows, BPJS integration, Xendit. Phase 2 begins
after Phase 1 is merged to `main` and CI is bootstrapped.

---

## 1. Phase 1 — Go Module Rename

**Objective:** Replace all `github.com/ingarso/pos` import paths with `github.com/kalpa-health/wellmed-cashier`.
Replace all `ingarso` org references. This is a mechanical find-and-replace across every `.go` file, `go.mod`,
`Dockerfile`, and any string that references the old identity.

1.0.1 Acceptance criteria: `go build ./...` passes in `wellmed-cashier` with zero references to `ingarso` or
old module path remaining. `grep -r "ingarso" .` returns nothing (excluding `.git/`).

  [ ] 1.1.1 Update `go.mod` line 1: `module github.com/ingarso/pos` → `module github.com/kalpa-health/wellmed-cashier`

  [ ] 1.1.2 Update `go.mod` Go version: `go 1.25.6` → `go 1.23.0` (align with other services)

  [ ] 1.1.3 Run global import path replacement across all `.go` files:
  ```bash
  find . -name "*.go" | xargs sed -i '' 's|github.com/ingarso/pos|github.com/kalpa-health/wellmed-cashier|g'
  ```
  Then verify: `grep -r "ingarso" . --include="*.go"` returns empty.

  [ ] 1.1.4 Run `go mod tidy` to clean up `go.sum` after module path change.

  [ ] 1.1.5 Run `go build ./...` — must pass with new module path.

---

## 2. Phase 2 — Port + Service Identity

**Objective:** Update the gRPC port from `:50052` to `:50053`. Rename all non-method-name references from
"POS" / "pos" to "cashier". The binary, log output, Dockerfile, and schema cluster name must reflect the
correct identity. Method/function names that contain `POS` (e.g., existing protobuf-generated methods)
are left unchanged.

2.0.1 Acceptance criteria: Service logs `"Starting Cashier Microservice..."` on boot. Dockerfile exposes
`:50053`. Schema cluster is named `cashier`. No non-method references to "POS" or "pos" remain in
application code or configuration.

  [ ] 2.1.1 Update `internal/config/env.go`:
  - `GRPCPort` default: `"50052"` → `"50053"`

  [ ] 2.1.2 Update `internal/app/app.go`:
  - Port fallback: `"50052"` → `"50053"`

  [ ] 2.1.3 Update `Dockerfile`:
  - `EXPOSE 50052` → `EXPOSE 50053`
  - Binary output: `-o /app/pos` → `-o /app/cashier`
  - CMD: `["./pos"]` → `["./cashier"]`
  - Build working directory stays `/app/cmd`

  [ ] 2.1.4 Update `cmd/main.go`:
  - Log message: `"Starting POS Microservice..."` → `"Starting Cashier Microservice..."`

  [ ] 2.1.5 Update `internal/config/schema_registry.go`:
  - Cluster name: `"pos"` → `"cashier"`
  - Schema names will become `cashier_2025`, `cashier_2026` etc.
  - Note: if any existing data lives in `pos_*` schemas, a schema rename migration is required.
    Add a comment flagging this for Phase 2 discussion.

  [ ] 2.1.6 Update all log messages referencing "POS" in non-method-name contexts:
  - `internal/app/app.go`: "Connected to master DB" log — fine as-is
  - `internal/messaging/consumer.go`: any log strings referencing "pos" → "cashier"
  - `internal/config/connection_manager.go`: schema-related log strings

  [ ] 2.1.7 Run `grep -rn '"pos' . --include="*.go"` and `grep -rn '"POS' . --include="*.go"` to find
  any remaining identity strings (excluding method/function/type names starting with POS in generated pb code).

  [ ] 2.1.8 Run `go build ./...` — must pass.

---

## 3. Phase 3 — Logger Migration (logrus → zap)

**Objective:** Replace `github.com/sirupsen/logrus` with `go.uber.org/zap` throughout. All log output must
be structured JSON (CloudWatch-compatible). Follow conventions §4.4: log at service boundary, never inside
loops, never `fmt.Printf` in production code.

3.0.1 Acceptance criteria: `grep -r "logrus" . --include="*.go"` returns empty. `grep -r "fmt.Println" . --include="*.go"`
returns empty. All log calls use `zap.L()` or a passed `*zap.Logger`. Service starts and logs a structured
JSON line on boot.

  [ ] 3.1.1 Add `go.uber.org/zap` to `go.mod`: `go get go.uber.org/zap`

  [ ] 3.1.2 Remove `github.com/sirupsen/logrus` from `go.mod`: `go mod tidy` after replacement.

  [ ] 3.1.3 Create `internal/logger/logger.go` — initialise a production zap logger with JSON encoding.
  Mirror the pattern from `wellmed-consultation/internal/logger/logger.go` if it exists, otherwise:
  ```go
  // Initialise production JSON logger. Call once at startup.
  func New() (*zap.Logger, error) {
      return zap.NewProduction()
  }
  ```

  [ ] 3.1.4 Update `cmd/main.go`:
  - Replace `log.SetLevel` / `log.SetFormatter` with `logger.New()`
  - Pass `*zap.Logger` into `app.New(cfg, logger)`
  - Replace all `log.Info*` / `log.Fatal*` calls with `logger.Info(...)` / `logger.Fatal(...)`

  [ ] 3.1.5 Update `internal/app/app.go`:
  - Accept `*zap.Logger` parameter in `New()` and `Application` struct
  - Replace all `log.Info*` / `log.Warn*` / `log.Error*` calls with zap equivalents
  - Structured fields: `zap.String("port", port)`, `zap.Error(err)` etc.

  [ ] 3.1.6 Update `internal/messaging/consumer.go`:
  - Accept `*zap.Logger` in `SagaEventConsumer` struct
  - Replace all logrus calls

  [ ] 3.1.7 Update `internal/grpc/server/*.go` — replace all logrus calls with zap.

  [ ] 3.1.8 Update `internal/interceptor/*.go` — replace all logrus calls with zap.

  [ ] 3.1.9 Update `internal/domain/*/service/*.go` and `repository/*.go` — replace all logrus calls with zap.

  [ ] 3.1.10 Update `internal/config/connection_manager.go` — replace logrus calls.

  [ ] 3.1.11 Remove the debug `fmt.Println` in `internal/domain/transaction/repository/transaction.go:131`:
  ```go
  fmt.Println("DB instance pool transaction: ", db.Statement.ConnPool) // DELETE THIS
  ```

  [ ] 3.1.12 Run `go build ./...` — must pass. Run linter: `golangci-lint run`.

---

## 4. Phase 4 — Env + Config Cleanup

**Objective:** Fix `godotenv.Load()` being called on every `getEnv()` invocation. Align env var names with
the standard used across other services. Add missing env vars (backbone address, service key, exchange name).

4.0.1 Acceptance criteria: `godotenv.Load()` called exactly once at process start. All required env vars
documented in `env.example`. Config struct contains all vars needed for Phase 1 operation.

  [ ] 4.1.1 Refactor `internal/config/env.go`:
  - Move `godotenv.Load()` call to a one-time `init()` or call it at the top of `Initialize()` (not inside `getEnv()`)
  - `getEnv()` should only call `os.Getenv()`

  [ ] 4.1.2 Add missing env vars to `Config` struct and `Initialize()`:

  | Variable | Purpose |
  |----------|---------|
  | `BACKBONE_GRPC_ADDRESS` | gRPC address for saga callback client |
  | `BACKBONE_API_KEY_CASHIER` | Per-service auth key (ADR-007) |
  | `RABBITMQ_URL` | RabbitMQ connection URL |
  | `RABBITMQ_EXCHANGE` | Override exchange name (default: `wellmed.saga`) |
  | `CASHIER_GRPC_ADDRESS` | Self-address (for SSM / health checks) |
  | `APP_ENV` | `development` / `staging` / `production` |

  [ ] 4.1.3 Remove `ListeningPort` (HTTP port) from Config — cashier is gRPC only, no HTTP listener.

  [ ] 4.1.4 Remove `JWTSecretKey` and `LifeTime` from Config — cashier validates the service key from
  gRPC metadata (ADR-007), not JWT. Auth is backbone's concern.

  [ ] 4.1.5 Remove `BaseUrlImage` from Config — not relevant to cashier domain.

  [ ] 4.1.6 Create `env.example` at repo root — document every variable with a placeholder value and
  a one-line comment explaining its purpose.

  [ ] 4.1.7 Run `go build ./...` — must pass.

---

## 5. Phase 5 — Saga Infrastructure (gRPC Callback Client + RabbitMQ Naming)

**Objective:** Replace the RabbitMQ reply mechanism with the correct gRPC callback to backbone's
`SagaCallbackService.ReportStepResult` (ADR-005 §5.6). Update exchange and routing key names to match
ADR-005 §5.5. Add the backbone gRPC client used for callbacks.

This is the most architecturally significant change in Phase 1.

5.0.1 Acceptance criteria: `internal/messaging/consumer.go` has no `publishReply` function. All saga
step results are reported via `SagaCallbackClient.ReportStepResult(...)`. Exchange is `wellmed.saga`.
Routing keys follow `saga.{saga_type}.{step_name}.{trigger|compensate}` pattern.

  [ ] 5.1.1 Copy `sagacallbackpb/` compiled protobuf package from `wellmed-backbone/proto/sagacallbackpb/`
  into `wellmed-cashier/proto/sagacallbackpb/`. This is the gRPC client interface cashier calls.

  [ ] 5.1.2 Create `internal/clients/saga_callback_client.go`:
  ```go
  // SagaCallbackClient wraps the gRPC connection to backbone's SagaCallbackService.
  // Cashier uses this to report step results after processing a saga trigger.
  type SagaCallbackClient struct {
      client sagacallbackpb.SagaCallbackServiceClient
  }

  func NewSagaCallbackClient(backboneAddr string) (*SagaCallbackClient, error) { ... }

  func (c *SagaCallbackClient) ReportStepResult(ctx context.Context, req *sagacallbackpb.StepResultRequest) error { ... }
  ```
  Mirror the implementation in `wellmed-consultation/internal/clients/saga_callback_client.go`.

  [ ] 5.1.3 Update `internal/messaging/consumer.go`:

  - **Remove** `publishReply()` method entirely — RabbitMQ reply to backbone is not the correct pattern
  - **Remove** `StepReply` struct — replaced by the proto message
  - Inject `*clients.SagaCallbackClient` into `SagaEventConsumer`
  - After each step handler completes, call `SagaCallbackClient.ReportStepResult` with:
    - `status`: `COMPLETED` on success, `FAILED` on business error, `REJECTED` on malformed payload
    - `payload`: JSON-encoded step result (e.g., `{"transaction_id": "..."}`)
    - `error_code`: module-specific code on failure (e.g., `CASHIER_TRANSACTION_CREATE_FAILED`)
    - `idempotency_key`: use `event.CommandID`
    - `tenant`: pass through from trigger

  [ ] 5.1.4 Update RabbitMQ exchange name in `setupQueues()` and `app.go`:
  - `"backbone.saga.events"` → `"wellmed.saga"` (ADR-005 §5.5)
  - Make configurable via `RABBITMQ_EXCHANGE` env var with default `"wellmed.saga"`

  [ ] 5.1.5 Update queue/routing key configuration to ADR-005 §5.5 convention.

  The routing keys are **confirmed from backbone's patient saga builder** (`internal/domain/patient/saga/builder.go`).
  Backbone derives routing keys as `saga.{saga_type}.{step_name_lowercase}.{trigger|compensate}`.
  The patient saga type is `patient`; the four POS/cashier steps are:

  | Step Name (backbone) | Trigger Routing Key | Compensate Routing Key |
  |---|---|---|
  | `POSTransaction_Consument` | `saga.patient.postransaction_consument.trigger` | `saga.patient.postransaction_consument.compensate` |
  | `POSTransaction_VisitPatient` | `saga.patient.postransaction_visitpatient.trigger` | `saga.patient.postransaction_visitpatient.compensate` |
  | `POSTransaction_VisitRegistration` | `saga.patient.postransaction_visitregistration.trigger` | `saga.patient.postransaction_visitregistration.compensate` |
  | `POSTransaction_VisitExamination` | `saga.patient.postransaction_visitexamination.trigger` | `saga.patient.postransaction_visitexamination.compensate` |

  Cashier's `SagaEventConsumer` must consume all four trigger keys and all four compensate keys.
  Replace the current two queues (`pos.transaction.create`, `pos.transaction.compensate`) with
  four trigger queues and four compensate queues bound to these routing keys on exchange `wellmed.saga`.

  Note: the step names (`POSTransaction_*`) are Go identifiers in backbone's builder and ARE the
  naming convention for these steps. They are not renamed to `cashier` — the step name is a
  functional description. The `POS` prefix here is a method name context (per the user convention:
  POS in method names is fine).

  [ ] 5.1.6 Update compensation step key map in `handleCompensateEvent` to use the confirmed step names:
  ```go
  stepKeyMap := map[string]string{
      "POSTransaction_Consument":         "pos_tx_consument_id",
      "POSTransaction_VisitPatient":      "pos_tx_visit_patient_id",
      "POSTransaction_VisitRegistration": "pos_tx_visit_reg_id",
      "POSTransaction_VisitExamination":  "pos_tx_visit_exam_id",
  }
  ```
  These keys are already correct in the existing code — keep them. Only the exchange name and
  queue binding mechanism change.

  [ ] 5.1.7 Update `internal/app/app.go`:
  - Pass `SagaCallbackClient` to `messaging.NewSagaEventConsumer(...)`
  - Initialise `SagaCallbackClient` using `BACKBONE_GRPC_ADDRESS` from config

  [ ] 5.1.8 Run `go build ./...` — must pass.

---

## 6. Phase 6 — Auth Interceptor (ADR-007)

**Objective:** Add per-service auth key validation to the gRPC server interceptors. Backbone validates
cashier's inbound gRPC calls using `BACKBONE_API_KEY_CASHIER`. Cashier validates backbone's inbound
gRPC calls using the same key.

6.0.1 Acceptance criteria: Unauthenticated gRPC calls to cashier return `codes.Unauthenticated`.
Calls with correct `BACKBONE_API_KEY_CASHIER` in metadata pass through.

  [ ] 6.1.1 Create `internal/interceptor/auth_interceptor.go` — validate `x-service-key` gRPC metadata
  against `BACKBONE_API_KEY_CASHIER` env var. Exempt `grpc.reflection.*` method paths.
  Mirror the implementation from `wellmed-consultation/internal/interceptor/auth_interceptor.go` or
  `wellmed-pharmacy/internal/interceptor/auth_interceptor.go`.

  [ ] 6.1.2 Register `AuthInterceptor` in the gRPC server interceptor chain in `internal/app/app.go`
  (before `TenantInterceptor`):
  ```go
  grpc.ChainUnaryInterceptor(
      interceptor.RecoveryInterceptor,
      interceptor.AuthInterceptor,      // NEW
      interceptor.ErrorInterceptor,
      interceptor.TenantInterceptor,
      interceptor.LoggingInterceptor,
  )
  ```

  [ ] 6.1.3 Update `internal/interceptor/tenant_interceptor.go`:
  - Remove the "legacy format" / "new format" dual-path logic — cashier receives `db_name` and `schema`
    from backbone metadata (the established pattern). The tenant_id fallback path was a workaround and
    should not exist in the standardised service.
  - Keep only: extract `db_name` and `schema` from metadata, inject into context.
  - If `db_name` or `schema` is missing, return `codes.InvalidArgument`.

  [ ] 6.1.4 Run `go build ./...` — must pass.

---

## 7. Phase 7 — Entity Cleanup

**Objective:** Resolve the `Consumer` vs `Consument` entity duplication. Remove dead code in `pkg/`.
Fix context value key to use a typed unexported key (not a plain string).

7.0.1 Acceptance criteria: One canonical entity for the consumer/patient concept in cashier's domain.
`pkg/transaction.go` removed. Context keys use typed unexported keys. `grep -r '"user_id"' . --include="*.go"` returns empty.

  [ ] 7.1.1 Audit `internal/schema/entity/consument.go` vs `internal/domain/consumer/entity/consumer.go`:
  - Determine which is actively used (GORM cluster has `Consument`; domain has `Consumer`)
  - `Consument` is the registered GORM entity and is referenced in `Transaction` foreign key — this is
    the canonical entity
  - `Consumer` in `internal/domain/consumer/entity/consumer.go` appears to be a legacy/unused struct.
    Verify no repository or service imports it, then delete it.
  - Consolidate: cashier's domain vocabulary uses `Consument` (the Indonesian/Dutch term consistent
    with existing DB schema). Document this in CLAUDE.md.

  [ ] 7.1.2 Remove `pkg/transaction.go` — the `TransactionSafe` generic utility is unused (the
  `config.TransactionManager` is used instead). Confirm no imports remain, then delete the file.

  [ ] 7.1.3 Create `internal/tenant/keys.go` — typed unexported context key:
  ```go
  type contextKey string
  const userIDKey contextKey = "user_id"
  ```
  Update `interceptor/tenant_interceptor.go` and `grpc/server/transaction.go` to use the typed key.

  [ ] 7.1.4 Run `go build ./...` — must pass.

---

## 8. Phase 8 — Migration Files (Replace AutoMigrate)

**Objective:** Replace GORM `AutoMigrate` with explicit versioned SQL migration files. `AutoMigrate`
in production code is unacceptable — it silently drops columns and cannot be reviewed or rolled back.

8.0.1 Acceptance criteria: `connection_manager.go` contains no `AutoMigrate` call. SQL migration files
exist for all entities registered in `schema_registry.go`. Migration runner is called explicitly from
a CLI command or startup flag, not on every connection initialisation.

  [ ] 8.1.1 Create `internal/database/migrations/` directory (rename/reorganise existing `internal/database/`).

  [ ] 8.1.2 Create `internal/database/migrations/001_cashier_tables.sql` — `CREATE TABLE IF NOT EXISTS`
  statements for all entities in the cashier schema cluster:
  - `activities`, `activity_statuses`, `billings`, `invoices`, `payment_details`
  - `payment_has_models`, `payment_summaries`, `refunds`, `split_payments`
  - `transactions`, `transaction_has_consuments`, `transaction_items`, `wallet_transactions`
  - `consuments`
  - All tables created in schema `cashier_{year}` (e.g., `cashier_2026`)
  - Include foreign key constraints, indexes, and ULID `char(26)` primary keys

  [ ] 8.1.3 Remove `autoMigrateCluster()` and its call from `internal/config/connection_manager.go`.

  [ ] 8.1.4 Remove the schema auto-create loop from `GetMasterConnection()`:
  ```go
  // DELETE this block:
  for _, cluster := range GetAllClusters() {
      schemaName := ...
      db.Exec(fmt.Sprintf("CREATE SCHEMA IF NOT EXISTS %s", schemaName))
      if err := cm.autoMigrateCluster(db, schemaName, cluster); err != nil { ... }
  }
  ```

  [ ] 8.1.5 Update `internal/database/migration.go` (existing file) to use the new SQL migration files.
  Reference the pattern from `wellmed-consultation` or `wellmed-pharmacy` for how SQL files are
  applied (likely using `golang-migrate/migrate` or reading SQL files directly).

  [ ] 8.1.6 Update `makefile` to add a `migrate` target that runs migrations explicitly.

  [ ] 8.1.7 Run `go build ./...` — must pass.

---

## 9. Phase 9 — Tests

**Objective:** Add unit tests for the service layer targeting 80% coverage. Follow the table-driven
test pattern established in consultation and pharmacy. Tests must not touch real DB — use mocks for
repositories. Handler tests (30% target) are optional for Phase 1; service tests are mandatory.

9.0.1 Acceptance criteria: `go test ./...` passes. Service layer coverage ≥ 80% for
`transaction/service`, `billing/service`, `invoice/service`, `payment_summary/service`.

  [ ] 9.1.1 Create `internal/domain/transaction/service/transaction_test.go`:
  - Table-driven tests for `Store` (happy path, missing reference_id, DB error)
  - Table-driven tests for `Delete` (happy path, transaction not found)
  - Table-driven tests for `Show` (happy path, not found)
  - Mock `TransactionRepository` and `PaymentSummaryRepository`

  [ ] 9.1.2 Create `internal/domain/billing/service/billing_test.go`:
  - Table-driven tests for all public service methods
  - Mock `BillingRepository`

  [ ] 9.1.3 Create `internal/domain/invoice/service/invoice_test.go`:
  - Table-driven tests for all public service methods
  - Mock `InvoiceRepository`

  [ ] 9.1.4 Create `internal/domain/payment_summary/service/payment_summary_test.go`:
  - Table-driven tests for all public service methods
  - Mock `PaymentSummaryRepository`

  [ ] 9.1.5 Create `internal/messaging/consumer_test.go`:
  - Test `handleCreateEvent`: valid payload → `SagaCallbackClient.ReportStepResult(COMPLETED)` called
  - Test `handleCreateEvent`: service error → `SagaCallbackClient.ReportStepResult(FAILED)` called
  - Test `handleCompensateEvent`: valid transaction_id → delete called → callback success
  - Mock `TransactionService` and `SagaCallbackClient`

  [ ] 9.1.6 Run `go test ./... -cover` and verify service layer coverage ≥ 80%.

---

## 10. Phase 10 — CI Workflow Files

**Objective:** Add GitHub Actions CI workflows matching the standard set by `wellmed-consultation` and
`wellmed-pharmacy`. These files will be committed but CI will be bootstrapped by Alex after Phase 1
is merged to `main`.

10.0.1 Acceptance criteria: `.github/workflows/ci.yml`, `main-approval-check.yml`, and `pr-review.yml`
exist in the repo and are syntactically valid YAML. No secrets are hardcoded.

  [ ] 10.1.1 Copy `.github/workflows/ci.yml` from `wellmed-pharmacy` — update service name references
  (`pharmacy` → `cashier`), binary name (`pharmacy` → `cashier`), and gRPC port (`50054` → `50053`).

  [ ] 10.1.2 Copy `.github/workflows/main-approval-check.yml` from `wellmed-pharmacy` — update service
  name references.

  [ ] 10.1.3 Copy `.github/workflows/pr-review.yml` from `wellmed-pharmacy` — update service name
  references.

  [ ] 10.1.4 Validate YAML syntax: `python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))"`.

---

## 11. Phase 11 — Documentation

**Objective:** Create all required documentation: `CLAUDE.md` in the repo, `services/cashier.md` in
kalpa-docs, SSM manifest in wellmed-infrastructure. Update `backbone.md` to correctly describe cashier
as a backbone-proxied service.

11.0.1 Acceptance criteria: All docs created and committed. `backbone.md` §4.1 correctly describes
cashier's three services as proxy pass-throughs. `kalpa-docs/services/cashier.md` exists with full
port, module, domain, and saga routing key documentation.

  [ ] 11.1.1 Create `wellmed-cashier/.claude/CLAUDE.md`:
  - Service identity: port `:50053`, module `github.com/kalpa-health/wellmed-cashier`
  - Domain vocabulary: Cashier owns `Transaction`, `Billing`, `Invoice`, `PaymentSummary`, `Consument`
  - Saga participant role (ADR-005): backbone triggers via `wellmed.saga` exchange; cashier responds
    via `SagaCallbackClient.ReportStepResult` gRPC
  - ADR references: ADR-005, ADR-006, ADR-007, ADR-009
  - Phase 1 stub status: saga callback wired; billing/invoice domain is real CRUD (no saga triggers yet)
  - Phase 2 scope preview: canonical transaction integration (ADR-009), invoice settlement sagas
  - What does NOT belong here: saga orchestration logic (backbone only), canonical model ownership
    (backbone only), patient data (reference ID only — never fetch)
  - `Consument` naming: cashier's domain term for the billing party. Not renamed to `Patient` or
    `Consumer` — matches existing DB schema.

  [ ] 11.1.2 Create `kalpa-docs/services/cashier.md`:
  - Mirrors the format of `services/consultation.md` and `services/pharmacy.md`
  - Sections: Overview, Repository (port, module, gRPC address env), Key Responsibilities,
    gRPC Interface Catalog, Multi-tenant Design, Saga Participation, Domain Entities,
    Backbone Proxy Pattern, Phase 1 Status, References

  [ ] 11.1.3 Create `wellmed-infrastructure/ssm/parameters/cashier.json`:
  - All env vars from Phase 4 (§4.1.2 table) with placeholder values
  - Match the structure of `wellmed-infrastructure/ssm/parameters/pharmacy.json`

  [ ] 11.1.4 Update `kalpa-docs/services/backbone.md` §4.1:
  - `TransactionService`, `BillingService`, `InvoiceService` rows: change description from
    `→ POS pass-through` to `→ Cashier pass-through` with note `wellmed-cashier :50053`
  - Add `CASHIER_GRPC_ADDRESS` to the backbone env var list (mirror `PHARMACY_GRPC_ADDRESS`)
  - Add `BACKBONE_API_KEY_CASHIER` to the service key list (ADR-007)

  [ ] 11.1.5 Update `MEMORY.md` — add cashier entry:
  - Port, module path, saga routing keys, Phase 1 complete status
  - Note schema cluster name: `cashier` (schemas: `cashier_2025`, `cashier_2026`)

  [ ] 11.1.6 Update this plan's Status to `Complete` and move to `kalpa-docs/plans/archive/` after
  Phase 1 is merged.

---

## 12. Phase 12 — Final Validation

**Objective:** End-to-end validation pass before handing off to Alex for push + CI bootstrap.

12.0.1 Acceptance criteria: All checks below pass. Zero `ingarso` references. Zero `logrus` references.
Zero `fmt.Println` in application code. `go build ./...` and `go test ./...` pass clean.

  [ ] 12.1.1 Run `grep -r "ingarso" . --include="*.go" --include="*.mod"` — must return empty.

  [ ] 12.1.2 Run `grep -r "logrus" . --include="*.go"` — must return empty.

  [ ] 12.1.3 Run `grep -r "fmt.Println" . --include="*.go"` — must return empty.

  [ ] 12.1.4 Run `grep -r "50052" . --include="*.go" --include="Dockerfile"` — must return empty.

  [ ] 12.1.5 Run `go build ./...` — must pass clean.

  [ ] 12.1.6 Run `go test ./... -race` — must pass with no data races.

  [ ] 12.1.7 Run `golangci-lint run` — must pass (zero errors, zero warnings that aren't excluded).

  [ ] 12.1.8 Run `go vet ./...` — must pass.

  [ ] 12.1.9 Verify `env.example` exists and contains all required variables.

  [ ] 12.1.10 Verify `.github/workflows/` contains `ci.yml`, `main-approval-check.yml`, `pr-review.yml`.

  [ ] 12.1.11 Verify `wellmed-cashier/.claude/CLAUDE.md` exists.

  [ ] 12.1.12 Verify `kalpa-docs/services/cashier.md` exists.

  [ ] 12.1.13 Verify `wellmed-infrastructure/ssm/parameters/cashier.json` exists.

  [ ] 12.1.14 Alex: push to `main`, run bootstrap script, confirm CI passes on GitHub Actions.

---

## 13. Phase 2 Preview (Separate Document)

Phase 2 is a feature and integration plan. It begins after Phase 1 is merged to `main` and CI is green.
The following are discussion topics, not tasks — they require backbone alignment before they can be
specified as concrete work items.

**13.1 Backbone proxy wiring for cashier**

Backbone's `TransactionService`, `BillingService`, and `InvoiceService` are currently described as
"POS pass-through" in `backbone.md §4.1`. This means backbone already expects to proxy these calls
to cashier via `CASHIER_GRPC_ADDRESS`. Phase 2 must confirm that backbone's gRPC proxy client for
cashier exists, and if not, create it (mirrors the `PHARMACY_GRPC_ADDRESS` proxy pattern).

**13.2 Saga types for cashier**

The current `handleCreateEvent` and `handleCompensateEvent` correspond to a `cashier_transaction.create`
saga initiated by backbone when a visit generates a bill. The full saga participation contract needs
to be defined: what sagas does cashier participate in, what steps does it own, what does it return?
This requires reviewing backbone's existing saga builders for POS/cashier steps.

**13.3 ADR-009 — Canonical Transaction integration**

Per ADR-009 §6.2: when an invoice transitions to `SETTLED` or `WRITTEN_OFF`, cashier must trigger
`CanonicalTransactionService`. This requires a `CanonicalTransactionClient` gRPC client (or a saga
callback pattern) to backbone. Phase 2 must define: when exactly this happens, what the payload looks
like, and whether it is a direct gRPC call or a saga step.

**13.4 Invoice domain — billing lifecycle**

The current `Billing`, `Invoice`, `PaymentSummary` domains are CRUD-only with no business lifecycle.
Phase 2 must define: how a visit generates an invoice, how payments are recorded, how the OPEN →
PARTIAL → SETTLED status machine is driven, and which saga steps cashier owns.

**13.5 Payment integrations**

Xendit (card/transfer payments) and insurance/corporate payer flows are not yet scoped. Phase 2 will
include an ADR or plan for payment gateway integration once the invoice lifecycle is defined.

**13.6 BPJS integration**

The system architecture shows BPJS integration as a separate service that reads from cashier's domain.
The handoff point (what cashier emits, when, on what event) needs to be defined.

---

## Open Questions

OQ-1: **Schema rename for existing data.** Backbone uses schema cluster name `pos` (entities in
`internal/schema/entity/pos/`; migration in `internal/db/migration/pos/migration.go`). Cashier's
schema registry also uses `pos`. Renaming cashier's cluster to `cashier` requires a coordinated
change in backbone too. Decision: **rename both in Phase 1** — backbone's `pos/` entity folder and
migration folder get renamed to `cashier/` as part of this plan. If any dev/staging data lives in
`pos_2025`/`pos_2026` schemas, a one-time `ALTER SCHEMA pos_2025 RENAME TO cashier_2025` is needed.
Confirm with Hamzah whether any dev DB has live POS data before executing Phase 8.

~~OQ-2~~: **CLOSED.** Routing keys confirmed from backbone's patient saga builder. See §5.1.5.

OQ-3: **`Consumer` entity — safe to delete?** `internal/domain/consumer/entity/consumer.go` contains
a `Consumer` struct not in the GORM cluster and apparently unused. Confirm no backbone or gateway code
references this type before deletion (§7.1.1). Quick check: `grep -r "consumer/entity" ~/Projects/wellmed/wellmed-backbone --include="*.go"`.

~~OQ-4~~: **CLOSED.** AutoMigrate removal decided: define clean SQL migrations for what cashier needs.
No 1:1 Laravel migration required. The one existing small client can be back-migrated via the canonical
archive pattern (immutable_visit, immutable_transaction) without a table-level migration plan. See §8.
Note: backbone's `internal/db/migration/pos/migration.go` also uses `AutoMigrate` — that's a backbone
cleanup task outside this plan's scope.

OQ-5: **Seeder file.** `internal/database/seeder.go` was not reviewed. Likely references `pos` schema
names or uses AutoMigrate. Review and update/remove as part of Phase 8.

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 06 March 2026 | Alex + Claude | Initial version — 12-phase standards alignment plan based on evaluation of wellmed-cashier repo. Phase 2 preview included as discussion section. |
