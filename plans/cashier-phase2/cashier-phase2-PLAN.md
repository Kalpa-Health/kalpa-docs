# Plan: Cashier Phase 2 — Settlement & Canonical Transaction

**Version:** 1.0
**Date:** 06 March 2026
**Author:** Alex
**Status:** Ready to execute

## Related Docs
- `kalpa-docs/adrs/ADR-005-saga-orchestration.md`
- `kalpa-docs/adrs/ADR-006-domain-service-boundaries.md`
- `kalpa-docs/adrs/ADR-007-per-service-auth-tokens.md`
- `kalpa-docs/adrs/ADR-009-canonical-archive-pattern.md`
- `kalpa-docs/services/cashier.md`
- `kalpa-docs/services/backbone.md`
- `wellmed-cashier/.claude/CLAUDE.md`
- `wellmed-backbone/internal/domain/canonical_visit/handler/canonical_visit_handler.go`
- `wellmed-backbone/internal/domain/canonical_visit/repository/canonical_visit_repository.go`
- `wellmed-backbone/proto/canonical_visit.proto`
- `wellmed-backbone/internal/grpc/client/transaction.go`
- `wellmed-backbone/internal/grpc/client/billing.go`
- `wellmed-backbone/internal/grpc/client/invoice.go`

---

## Scope

Phase 2 delivers the settlement lifecycle: cashier can mark an invoice SETTLED,
backbone writes the canonical transaction record, and the backbone→cashier proxy
works end-to-end with correct auth.

Phase 2 does NOT include:
- Payment merchant integrations (Xendit, BPJS) — Phase 3
- Jurnal accounting integration — Phase 3
- Multi-invoice payer settlement — Phase 3
- Doctor sign-off / CanonicalVisitService wiring — separate consultation plan

---

## Architecture Summary

### Settlement flow (user-initiated)

```
User settles invoice in UI
  → Gateway → Backbone (InvoiceService.Settle proxy)
    → Cashier: marks invoice SETTLED / WRITTEN_OFF
    → Cashier: calls CanonicalTransactionClient.WriteTransactionRecord (gRPC → Backbone)
      → Backbone CanonicalTransactionService: writes immutable_transactions row
      → Backbone: fires wellmed.events canonical_transaction.created (fire-and-forget)
```

This mirrors the consultation doctor sign-off → CanonicalVisitService pattern exactly.
ADR-006 gains a second explicit cashier exception: CanonicalTransactionClient alongside
SagaCallbackClient.

### CanonicalTransactionService location

New domain in **backbone**: `internal/domain/canonical_transaction/`.
Follows the same handler + repository pattern as `canonical_visit` — no service layer,
thin handler calls repo directly.

### immutable_transactions table

Lives in **backbone's tenant DB** (per-tenant schema, same as canonical_visits).
UBL-concept-shaped per ADR-009. Written once at settlement; supports supersession
(CORRECTION | VOID) per ADR-009 §2.6.2.

---

## Phase 1: Backbone Proxy Auth Fix

**Prerequisite for all E2E testing.** Backbone's gRPC clients for cashier do not
inject `x-service-key` metadata. Cashier's AuthInterceptor (ADR-007) will reject
every backbone→cashier call without this fix.

### Task 1.1: Inject x-service-key in backbone cashier clients
- **Type**: AI
- **Input**: `wellmed-backbone/internal/grpc/client/transaction.go`, `billing.go`, `invoice.go`
- **Action**: Add `BACKBONE_API_KEY_CASHIER` env var read to backbone config. In each
  client's `prepareContext` (or equivalent), inject `x-service-key: $BACKBONE_API_KEY_CASHIER`
  into outgoing gRPC metadata. Follow the existing `db_name`/`schema` metadata injection pattern.
- **Output**: Modified `transaction.go`, `billing.go`, `invoice.go`; updated env config
- **Acceptance**: `grep "x-service-key" wellmed-backbone/internal/grpc/client/*.go` returns hits
  in all three files; `go build ./...` passes in backbone

### Task 1.2: Add BACKBONE_API_KEY_CASHIER to backbone env
- **Type**: AI
- **Input**: `wellmed-backbone/internal/config/env.go` (or equivalent env struct)
- **Action**: Add `BackboneApiKeyCashier string` field, read from `BACKBONE_API_KEY_CASHIER`
  env var. This key is sent as `x-service-key` on all outbound cashier gRPC calls.
- **Output**: Updated env config
- **Acceptance**: `go build ./...` passes; env var documented in backbone's `env.example`

---

## Phase 2: Schema Rename Assessment

The cashier schema cluster was renamed from `pos` to `cashier` in Phase 1. Dev/staging
DBs may still have `pos_YYYY` schemas that need renaming.

### Task 2.1: Assess schema rename pervasiveness
- **Type**: AI
- **Input**: `wellmed-backbone/internal/config/` schema registry; `wellmed-backbone/internal/db/`
- **Action**: Search backbone for any hardcoded `pos_` schema references that would break
  with renamed schemas. List all files and string patterns found. Report: (a) is the schema
  name driven by the registry alone, or hardcoded in places? (b) what SQL would be needed
  on the DB server?
- **Output**: Written assessment in progress log — no code changes
- **Acceptance**: Clear list of any hardcoded references; SQL rename statement drafted

### Task 2.2: Schema rename on dev/staging DB
- **Type**: HUMAN
- **Input**: Assessment from Task 2.1; access to dev/staging PostgreSQL
- **Action**: For each tenant DB on dev and staging, run:
  ```sql
  ALTER SCHEMA pos_2025 RENAME TO cashier_2025;
  ALTER SCHEMA pos_2026 RENAME TO cashier_2026;
  ```
  (Adjust year suffixes as needed per live tenant list.)
- **Output**: Renamed schemas on dev/staging
- **Acceptance**: `\dn` in psql shows `cashier_YYYY` schemas; backbone connects successfully

---

### 🔲 CHECKPOINT: Proxy auth + schema rename complete
**Review**: Verify backbone can call cashier without auth errors. Run a manual gRPC call
through backbone to cashier (e.g., TransactionService.Index) on dev and confirm it reaches
cashier with correct tenant context.
**Resume**: "continue cashier-phase2 plan"

---

## Phase 3: immutable_transactions Table Migration

### Task 3.1: Write SQL migration for immutable_transactions
- **Type**: AI
- **Input**: ADR-009 §§2.5, 2.6, 6.3; backbone EMR migration pattern at
  `wellmed-backbone/internal/db/migration/emr/migration.go`
- **Action**: Create `wellmed-backbone/internal/db/migration/sql/003_immutable_transactions.sql`
  with the following table (UBL-concept-shaped per ADR-009):

  ```sql
  CREATE TABLE IF NOT EXISTS canonical_transactions (
    id                  CHAR(26)     PRIMARY KEY,
    -- Supersession fields (ADR-009 §6.3)
    supersedes_id       CHAR(26)     NULL REFERENCES canonical_transactions(id),
    superseded_by_id    CHAR(26)     NULL,
    supersession_type   VARCHAR(20)  NULL CHECK (supersession_type IN ('CORRECTION','VOID')),
    superseded_at       TIMESTAMPTZ  NULL,
    archive_status      VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE'
                                     CHECK (archive_status IN ('ACTIVE','SUPERSEDED')),
    -- Source references
    invoice_id          CHAR(26)     NOT NULL,
    transaction_id      CHAR(26)     NOT NULL,
    reference_type      VARCHAR(50)  NULL,
    reference_id        CHAR(26)     NULL,
    -- AccountingCustomerParty (UBL)
    consument_id        CHAR(26)     NOT NULL,
    consument_name      VARCHAR(255) NOT NULL,
    consument_type      VARCHAR(50)  NOT NULL,
    -- Financial totals (IDR, integer cents)
    currency            CHAR(3)      NOT NULL DEFAULT 'IDR',
    subtotal            BIGINT       NOT NULL DEFAULT 0,
    tax_total           BIGINT       NOT NULL DEFAULT 0,
    payable_amount      BIGINT       NOT NULL DEFAULT 0,
    -- Line items as JSONB (InvoiceLines — UBL concept)
    line_items          JSONB        NOT NULL DEFAULT '[]',
    -- Settlement
    settlement_status   VARCHAR(20)  NOT NULL CHECK (settlement_status IN ('SETTLED','WRITTEN_OFF')),
    settled_at          TIMESTAMPTZ  NOT NULL,
    -- Tenant routing
    tenant_db           VARCHAR(100) NOT NULL,
    tenant_schema       VARCHAR(100) NOT NULL,
    created_at          TIMESTAMPTZ  NOT NULL DEFAULT now()
  );

  CREATE INDEX IF NOT EXISTS idx_canonical_transactions_invoice_id
    ON canonical_transactions (invoice_id);
  CREATE INDEX IF NOT EXISTS idx_canonical_transactions_archive_status
    ON canonical_transactions (archive_status);
  ```

- **Output**: `wellmed-backbone/internal/db/migration/sql/003_immutable_transactions.sql`
- **Acceptance**: File exists; SQL is valid (check with `psql --dry-run` or equivalent)

### Task 3.2: Register migration in backbone runner
- **Type**: AI
- **Input**: `wellmed-backbone/internal/db/migration/migration.go`; Task 3.1 output
- **Action**: Extend the backbone migration runner to execute `003_immutable_transactions.sql`
  against the tenant DB (same approach as existing SQL file migrations in
  `wellmed-backbone/internal/db/migration/sql/`). Follow the existing pattern — if the
  migration runner uses embed.FS or reads files, add the new file to the same mechanism.
- **Output**: Updated migration runner
- **Acceptance**: `go build ./...` passes in backbone; migration would create the table on
  a fresh tenant DB

---

## Phase 4: CanonicalTransactionService Proto

### Task 4.1: Write canonical_transaction.proto
- **Type**: AI
- **Input**: `wellmed-backbone/proto/canonical_visit.proto` (template); ADR-009 §§2.5, 6.2
- **Action**: Create `wellmed-backbone/proto/canonical_transaction.proto` modelled on
  `canonical_visit.proto`. The service has one RPC: `WriteTransactionRecord`. Request
  carries invoice/transaction IDs, consument info, financial totals, line items (as JSON
  string), settlement status, settled_at, and tenant routing. Response returns accepted bool
  + canonical_record_id.

  Package: `canonical.transaction.v1`
  go_package: `canonicaltransactionpb/`
- **Output**: `wellmed-backbone/proto/canonical_transaction.proto`
- **Acceptance**: File exists; proto syntax is valid

### Task 4.2: Generate Go code from proto
- **Type**: HUMAN
- **Input**: Task 4.1 output; protoc installed with go + grpc plugins
- **Action**: From wellmed-backbone root, run:
  ```bash
  protoc --go_out=. --go-grpc_out=. proto/canonical_transaction.proto
  ```
  Verify `proto/canonicaltransactionpb/` directory is created with `.pb.go` and
  `_grpc.pb.go` files.
- **Output**: `wellmed-backbone/proto/canonicaltransactionpb/canonical_transaction.pb.go`
  and `canonical_transaction_grpc.pb.go`
- **Acceptance**: `go build ./...` passes in backbone

---

## Phase 5: CanonicalTransactionService Domain in Backbone

### Task 5.1: Write canonical_transaction repository
- **Type**: AI
- **Input**: `wellmed-backbone/internal/domain/canonical_visit/repository/canonical_visit_repository.go`
  (template); Task 3.1 schema; ADR-009 §6.3
- **Action**: Create `wellmed-backbone/internal/domain/canonical_transaction/repository/canonical_transaction_repository.go`.
  Follow canonical_visit_repository.go exactly:
  - `CanonicalTransactionRecord` struct (matching the SQL table columns)
  - `CanonicalTransactionRepository` struct with `cm *configDB.ConnectionManager`
  - `Save(ctx, rec)` using raw SQL INSERT (not GORM AutoMigrate)
  - Use `r.cm.WithTenantTable(ctx, "canonical_transactions")`
  - ULID assigned by caller
- **Output**: `wellmed-backbone/internal/domain/canonical_transaction/repository/canonical_transaction_repository.go`
- **Acceptance**: `go build ./...` passes in backbone

### Task 5.2: Write canonical_transaction handler
- **Type**: AI
- **Input**: `wellmed-backbone/internal/domain/canonical_visit/handler/canonical_visit_handler.go`
  (template); Task 4.2 proto output; Task 5.1 repository
- **Action**: Create `wellmed-backbone/internal/domain/canonical_transaction/handler/canonical_transaction_handler.go`.
  Follow canonical_visit_handler.go exactly:
  - `CanonicalTransactionHandler` struct embedding `UnimplementedCanonicalTransactionServiceServer`
  - `NewCanonicalTransactionHandler(cm, publisher)` constructor
  - `WriteTransactionRecord(ctx, req)` method:
    1. Validate required fields (invoice_id, transaction_id, consument_id, tenant, settled_at)
    2. Parse tenant string ("db_name:schema"), inject with `tenant.WithTenant`
    3. Build `CanonicalTransactionRecord`, assign ULID
    4. `repo.Save(ctx, rec)`
    5. Fire `backbone.canonical_transaction.created` event to `wellmed.events` exchange
       (fire-and-forget, same pattern as canonical_visit)
    6. Return `accepted: true, canonical_record_id: rec.ID`
- **Output**: `wellmed-backbone/internal/domain/canonical_transaction/handler/canonical_transaction_handler.go`
- **Acceptance**: `go build ./...` passes in backbone

### Task 5.3: Register CanonicalTransactionService in backbone gRPC server
- **Type**: AI
- **Input**: `wellmed-backbone/internal/app/application.go`; Task 5.2 output
- **Action**: Register `CanonicalTransactionHandler` with the backbone gRPC server
  (follow how `CanonicalVisitHandler` is registered). Wire `NewCanonicalTransactionHandler`
  with the existing `ConnectionManager` and `MessagePublisher`.
- **Output**: Updated `wellmed-backbone/internal/app/application.go`
- **Acceptance**: `go build ./...` passes in backbone; `CanonicalTransactionServiceServer`
  is registered

---

### 🔲 CHECKPOINT: Canonical service complete — review before cashier integration
**Review**: Review the proto, handler, and repository. Confirm field names match ADR-009
§§2.5/6.3. Confirm the tenant routing pattern matches canonical_visit. Confirm the event
routing key (`backbone.canonical_transaction.created`) is consistent with the events
exchange pattern used by canonical_visit.
**Resume**: "continue cashier-phase2 plan"

---

## Phase 6: CanonicalTransactionClient in Cashier

### Task 6.1: Copy proto generated files to cashier
- **Type**: AI
- **Input**: `wellmed-backbone/proto/canonicaltransactionpb/` (Task 4.2 output)
- **Action**: Copy the generated `canonicaltransactionpb` package into
  `wellmed-cashier/proto/canonicaltransactionpb/` (same pattern as `sagacallbackpb`).
  Update `wellmed-cashier/go.mod` if backbone proto types are referenced via a local
  replace directive or copy — follow whichever pattern the existing `sagacallbackpb`
  uses in cashier.
- **Output**: `wellmed-cashier/proto/canonicaltransactionpb/` with pb files
- **Acceptance**: `go build ./...` passes in cashier

### Task 6.2: Write CanonicalTransactionClient
- **Type**: AI
- **Input**: `wellmed-cashier/internal/clients/saga_callback_client.go` (template);
  Task 6.1 proto; `wellmed-cashier/internal/config/env.go`
- **Action**: Create `wellmed-cashier/internal/clients/canonical_transaction_client.go`.
  Follow saga_callback_client.go pattern:
  - `CanonicalTransactionClient` interface with `WriteTransactionRecord(ctx, req) (*WriteTransactionResponse, error)`
  - Concrete `canonicalTransactionClient` struct with gRPC connection to `BACKBONE_GRPC_ADDRESS`
    (same address as SagaCallbackClient — backbone is the single outbound target)
  - Add `x-service-key: BACKBONE_API_KEY_CASHIER` to outgoing metadata (ADR-007)
  - `WriteTransactionResponse` struct: `Accepted bool`, `CanonicalRecordID string`
  - `StepResultRequest`-equivalent struct for the request payload
- **Output**: `wellmed-cashier/internal/clients/canonical_transaction_client.go`
- **Acceptance**: `go build ./...` passes in cashier

### Task 6.3: Wire client into cashier app
- **Type**: AI
- **Input**: `wellmed-cashier/internal/app/app.go`; Task 6.2 output
- **Action**: Instantiate `CanonicalTransactionClient` in `app.go` alongside
  `SagaCallbackClient`. Inject into `InvoiceService` (Phase 7). No new env vars needed —
  reuses `BACKBONE_GRPC_ADDRESS`.
- **Output**: Updated `wellmed-cashier/internal/app/app.go`
- **Acceptance**: `go build ./...` passes

---

## Phase 7: Invoice Settlement in Cashier

### Task 7.1: Add invoice status field and migration
- **Type**: AI
- **Input**: `wellmed-cashier/internal/schema/entity/invoice.go`;
  `wellmed-cashier/internal/database/migrations/001_cashier_tables.sql`
- **Action**: Add `Status` field to the `Invoice` entity (VARCHAR, default `OPEN`):
  `OPEN | PARTIAL | SETTLED | WRITTEN_OFF | VOIDED`. Create
  `wellmed-cashier/internal/database/migrations/002_invoice_status.sql`:
  ```sql
  ALTER TABLE invoices ADD COLUMN IF NOT EXISTS status VARCHAR(20) NOT NULL DEFAULT 'OPEN'
    CHECK (status IN ('OPEN','PARTIAL','SETTLED','WRITTEN_OFF','VOIDED'));
  CREATE INDEX IF NOT EXISTS idx_invoices_status ON invoices (status);
  ```
- **Output**: Updated entity; new migration file
- **Acceptance**: `go build ./...` passes; migration file syntactically valid

### Task 7.2: Add Settle method to InvoiceService interface + implementation
- **Type**: AI
- **Input**: `wellmed-cashier/internal/domain/invoice/service/invoice.go`;
  `wellmed-cashier/internal/domain/invoice/repository/` (invoice repository interface);
  Task 6.2 client interface
- **Action**:
  1. Add `Settle(ctx context.Context, req *SettleInvoiceRequest) (*SettleInvoiceResult, error)`
     to `InvoiceService` interface
  2. `SettleInvoiceRequest`: `InvoiceID`, `SettlementStatus` (SETTLED | WRITTEN_OFF),
     `SettledAt time.Time`, consument snapshot fields, financial totals, line items JSON
  3. `SettleInvoiceResult`: `CanonicalRecordID string`
  4. Implementation:
     - Validate invoice exists and is in a settleable state (not already SETTLED/WRITTEN_OFF/VOIDED)
     - Update invoice status to `req.SettlementStatus`
     - Build `WriteTransactionRecord` request from invoice + snapshot fields
     - Call `CanonicalTransactionClient.WriteTransactionRecord`
     - Return canonical record ID on success
  5. Inject `CanonicalTransactionClient` into `invoiceService` struct
- **Output**: Updated `invoice.go` service
- **Acceptance**: `go build ./...` passes; settle rejects double-settle; calls canonical client

### Task 7.3: Add Settle RPC to invoice proto and gRPC handler
- **Type**: AI
- **Input**: `wellmed-cashier/proto/invoice.proto`;
  `wellmed-cashier/internal/grpc/server/invoice.go`
- **Action**:
  1. Add `SettleInvoice` RPC to `InvoiceService` in `invoice.proto`:
     request carries invoice_id, settlement_status, settled_at (RFC3339), consument
     snapshot fields, financial totals, line_items (JSON string)
  2. Regenerate proto (HUMAN step below) or add stub handler with compile-time check
  3. Implement `SettleInvoice` handler in `server/invoice.go` — thin handler: unmarshal,
     call `InvoiceService.Settle`, return canonical_record_id
- **Output**: Updated `invoice.proto`; updated `server/invoice.go`
- **Notes**: Proto regeneration is a HUMAN step (Task 7.4)

### Task 7.4: Regenerate cashier invoice proto
- **Type**: HUMAN
- **Input**: Task 7.3 updated `invoice.proto`
- **Action**: From wellmed-cashier root:
  ```bash
  protoc --go_out=. --go-grpc_out=. proto/invoice.proto
  ```
- **Output**: Updated `proto/invoicepb/` files
- **Acceptance**: `go build ./...` passes in cashier

---

## Phase 8: Backbone Proxy for InvoiceService.Settle

### Task 8.1: Add Settle to backbone invoice gRPC client
- **Type**: AI
- **Input**: `wellmed-backbone/internal/grpc/client/invoice.go`
- **Action**: Add `SettleInvoice(ctx, req)` method to backbone's `InvoiceRpc` client.
  Follow existing method pattern: `prepareContext` → get client → call `client.SettleInvoice`
  → return response. Inject `x-service-key` (Task 1.1 already adds this to prepareContext).
- **Output**: Updated `wellmed-backbone/internal/grpc/client/invoice.go`
- **Acceptance**: `go build ./...` passes in backbone

### Task 8.2: Add Settle to backbone invoice gRPC server handler
- **Type**: AI
- **Input**: `wellmed-backbone/internal/grpc/server/invoice.go`;
  `wellmed-backbone/internal/domain/invoice/service/invoice.go`
- **Action**: Add `SettleInvoice` handler to backbone's invoice server — thin proxy to
  `InvoiceRpc.SettleInvoice`. Add `Settle` to backbone's `InvoiceService` interface and
  implementation (delegates to `InvoiceRpc`).
- **Output**: Updated `server/invoice.go`; updated backbone invoice service
- **Acceptance**: `go build ./...` passes in backbone

---

## Phase 9: Tests

### Task 9.1: Unit tests for CanonicalTransactionService handler (backbone)
- **Type**: AI
- **Input**: `wellmed-backbone/internal/domain/canonical_transaction/handler/canonical_transaction_handler.go`
- **Action**: Write `canonical_transaction_handler_test.go` using table-driven tests.
  Mock `CanonicalTransactionRepository` interface (if not already an interface, extract one).
  Cover: missing required fields → InvalidArgument, valid request → Save called + event
  published, repo error → Internal error.
- **Output**: `wellmed-backbone/internal/domain/canonical_transaction/handler/canonical_transaction_handler_test.go`
- **Acceptance**: `go test ./internal/domain/canonical_transaction/...` passes

### Task 9.2: Unit tests for InvoiceService.Settle (cashier)
- **Type**: AI
- **Input**: `wellmed-cashier/internal/domain/invoice/service/invoice.go` (Task 7.2)
- **Action**: Add tests to `wellmed-cashier/internal/domain/invoice/service/invoice_test.go`.
  Mock `CanonicalTransactionClient` interface. Cover:
  - Settle happy path → status updated, canonical client called, record ID returned
  - Settle already SETTLED invoice → error, canonical client NOT called
  - Settle VOIDED invoice → error
  - Canonical client error → error propagated, invoice status NOT updated (rollback)
- **Output**: Updated `invoice_test.go`
- **Acceptance**: `go test ./internal/domain/invoice/...` passes

### Task 9.3: Run full test suites
- **Type**: AI
- **Input**: Both repos
- **Action**: Run `go test -race ./...` in both `wellmed-cashier` and `wellmed-backbone`.
  Fix any failures. Run `golangci-lint run` in both. Fix any lint issues.
- **Output**: Clean test + lint results in both repos
- **Acceptance**: Zero test failures, zero lint errors in both repos

---

### 🔲 CHECKPOINT: E2E smoke test — deploy to dev and verify settlement flow
**Review**:
1. Deploy backbone and cashier to dev
2. Create a transaction via the existing patient saga (POSTransaction_Consument)
3. Call `InvoiceService.Settle` through backbone proxy (use grpcurl or a test script)
4. Verify: invoice status = SETTLED; `canonical_transactions` row exists in tenant DB with
   correct fields and `archive_status = ACTIVE`
5. Verify: `backbone.canonical_transaction.created` event visible on RabbitMQ
6. Attempt double-settle → expect error, no duplicate canonical record
**Resume**: "continue cashier-phase2 plan"

---

## Phase 10: Documentation + ADR Updates

### Task 10.1: Amend ADR-006 — add CanonicalTransactionClient exception
- **Type**: AI
- **Input**: `kalpa-docs/adrs/ADR-006-domain-service-boundaries.md`
- **Action**: Add amendment to ADR-006: cashier has a second permitted outbound call —
  `CanonicalTransactionClient` to backbone (same as consultation's `CanonicalVisitClient`
  exception). Document: when it is called (invoice settlement only), what it sends
  (snapshot of settled invoice), and that it reuses `BACKBONE_GRPC_ADDRESS`.
- **Output**: Updated `ADR-006-domain-service-boundaries.md`
- **Acceptance**: ADR has an amendment section; edit log updated

### Task 10.2: Update cashier.md — Phase 2 status + new methods
- **Type**: AI
- **Input**: `kalpa-docs/services/cashier.md`
- **Action**:
  - Update gRPC interface catalog to include `InvoiceService.Settle`
  - Update Phase 1 status section → mark complete; add Phase 2 status section
  - Add settlement state machine table (OPEN → PARTIAL → SETTLED / WRITTEN_OFF / VOIDED)
  - Add `CanonicalTransactionClient` to the list of permitted outbound calls
  - Note: Phase 3 scope (Jurnal, Xendit, BPJS, multi-invoice payer settlement)
- **Output**: Updated `kalpa-docs/services/cashier.md`

### Task 10.3: Update backbone.md — CanonicalTransactionService
- **Type**: AI
- **Input**: `kalpa-docs/services/backbone.md`
- **Action**: Add `CanonicalTransactionService` to backbone's service catalog section.
  Document: write path (cashier calls at settlement), table (`canonical_transactions`),
  event emitted (`backbone.canonical_transaction.created`). Mirror how CanonicalVisitService
  is documented.
- **Output**: Updated `kalpa-docs/services/backbone.md`

### Task 10.4: Update cashier CLAUDE.md + MEMORY.md
- **Type**: AI
- **Input**: `wellmed-cashier/.claude/CLAUDE.md`; `~/.claude/projects/.../memory/MEMORY.md`
- **Action**: Update cashier CLAUDE.md Phase 2 scope section to mark complete; add
  `CanonicalTransactionClient` to permitted outbound calls. Update MEMORY.md cashier entry:
  Phase 2 complete, note Phase 3 scope.
- **Output**: Updated CLAUDE.md; updated MEMORY.md

---

## Phase 11: Final Validation

### Task 11.1: Final validation checklist
- **Type**: AI
- **Action**: Verify all of the following:
  - [ ] 11.1: `x-service-key` injected in all 3 backbone cashier clients
  - [ ] 11.2: `immutable_transactions` migration SQL exists and is registered
  - [ ] 11.3: `canonical_transaction.proto` exists and is generated
  - [ ] 11.4: `CanonicalTransactionHandler` registered in backbone gRPC server
  - [ ] 11.5: `CanonicalTransactionClient` exists in cashier `internal/clients/`
  - [ ] 11.6: `InvoiceService.Settle` implemented with double-settle guard
  - [ ] 11.7: `InvoiceService.Settle` calls `CanonicalTransactionClient`
  - [ ] 11.8: `InvoiceService.SettleInvoice` registered in cashier gRPC server
  - [ ] 11.9: Backbone `InvoiceService.Settle` proxy added (client + server)
  - [ ] 11.10: `go test -race ./...` passes in cashier (0 failures)
  - [ ] 11.11: `go test -race ./...` passes in backbone (0 failures)
  - [ ] 11.12: `golangci-lint run` passes in cashier (0 warnings)
  - [ ] 11.13: `golangci-lint run` passes in backbone (0 warnings)
  - [ ] 11.14: ADR-006 amended
  - [ ] 11.15: cashier.md + backbone.md updated
  - [ ] 11.16: HUMAN — push to develop + open PRs; bootstrap CI runs green
- **Output**: Completed checklist in progress log
- **Acceptance**: All 15 automated checks pass; 11.16 logged as WAITING_HUMAN
