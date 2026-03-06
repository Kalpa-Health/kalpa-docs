# Service: Cashier

**Version:** 1.0
**Date:** 2026-03-06
**Status:** Active — Phase 1 standards alignment complete
**Maintained by:** Alex (Kalpa Inovasi Digital)

---

## 1. Overview

1.1 Cashier owns the billing lifecycle for WellMed clinic encounters: transactions, billings, invoices, payment summaries, and the `Consument` (billing party) entity. It is a participant in Backbone sagas per ADR-005 — it receives RabbitMQ step triggers from Backbone and reports results via `SagaCallbackService`. Cashier never initiates a saga.

1.2 Gateway calls Backbone only. Backbone proxies cashier-bound gRPC calls to this service via `CASHIER_GRPC_ADDRESS`. There is no direct gateway→cashier routing.

1.3 Patient data does not live in cashier. Cashier stores reference IDs (`reference_type`, `reference_id`) but never fetches patient records from Backbone (ADR-006). The `Consument` entity is the cashier-domain representation of the billing party.

---

## 2. Repository

| Item | Value |
|------|-------|
| **Repo** | `wellmed-cashier` |
| **Language** | Go 1.23+ |
| **gRPC address** | `CASHIER_GRPC_ADDRESS` (env on backbone) |
| **gRPC port** | `:50053` |
| **Module** | `github.com/kalpa-health/wellmed-cashier` |

---

## 3. Key Responsibilities

- **Transaction management** — creates and manages billing transactions for clinical encounters; saga trigger creates a `Transaction` + `PaymentSummary` pair atomically
- **Billing lifecycle** — `Billing` entity ties a transaction to an authorizing actor (cashier/nurse); Phase 2 will add billing saga triggers
- **Invoice management** — `Invoice` entity tracks payment obligation per billing party; settlement triggers canonical archive (Phase 2, ADR-009)
- **Payment summaries** — tracks financial totals (amount, debt, discount, tax, paid, refund) per transaction
- **Consument** — cashier's canonical billing party entity; not renamed to Patient or Consumer — matches existing DB schema
- **Saga participant** — receives step triggers from Backbone, reports results via `SagaCallbackService`. Does not orchestrate (ADR-005)
- **Multi-tenant database access** — one PostgreSQL database per tenant, per-service schema isolation (`cashier_YYYY`)

---

## 4. gRPC Interface Catalog

4.1 Services registered at `:50053`:

| # | Service | Key Methods | Phase 1 Status |
|---|---------|-------------|----------------|
| 1 | TransactionService | Index, Show, Store, Delete | Functional |
| 2 | BillingService | GetAll, GetById | Functional CRUD |
| 3 | InvoiceService | GetAll, GetById | Functional CRUD |
| 4 | PaymentSummaryService | Store | Functional |

---

## 5. Multi-tenant Design

5.1 Every gRPC request carries `db_name` and `schema` via gRPC metadata (injected by `TenantInterceptor`). `ConnectionManager` maintains a lazy-initialized pool keyed by `db_name`. Queries execute with `SET search_path TO cashier_YYYY` before each request.

5.2 Schema cluster name: `cashier`. Schemas follow `cashier_{year}` convention: `cashier_2025`, `cashier_2026`.

5.3 The previous schema cluster name was `pos` (from the original POS service). Any existing dev/staging schemas must be renamed: `ALTER SCHEMA pos_YYYY RENAME TO cashier_YYYY` (coordinate with backbone's schema registry update).

---

## 6. Saga Participation

6.1 Cashier participates in Backbone's **patient saga** (ADR-005). Backbone publishes triggers to the `wellmed.saga` RabbitMQ exchange. Cashier binds 8 queues (4 trigger + 4 compensate) on startup.

6.2 Routing key convention: `saga.patient.postransaction_{step}.{trigger|compensate}`

| Step Name (backbone) | Trigger Key | Compensate Key | Saga Context Key |
|---|---|---|---|
| `POSTransaction_Consument` | `saga.patient.postransaction_consument.trigger` | `saga.patient.postransaction_consument.compensate` | `pos_tx_consument_id` |
| `POSTransaction_VisitPatient` | `saga.patient.postransaction_visitpatient.trigger` | `saga.patient.postransaction_visitpatient.compensate` | `pos_tx_visit_patient_id` |
| `POSTransaction_VisitRegistration` | `saga.patient.postransaction_visitregistration.trigger` | `saga.patient.postransaction_visitregistration.compensate` | `pos_tx_visit_reg_id` |
| `POSTransaction_VisitExamination` | `saga.patient.postransaction_visitexamination.trigger` | `saga.patient.postransaction_visitexamination.compensate` | `pos_tx_visit_exam_id` |

6.3 Step results are reported via gRPC `SagaCallbackService.ReportStepResult` on Backbone (ADR-005 §5.6). Status: `COMPLETED` on success, `FAILED` on business error, error code: `CASHIER_TRANSACTION_CREATE_FAILED` / `CASHIER_TRANSACTION_COMPENSATE_FAILED`.

---

## 7. Domain Entities

7.1 All primary keys are `CHAR(26)` ULIDs (application-generated via `helper.GenerateUlid()`). Never use UUID or `gen_random_uuid()`.

| Entity | Table | Description |
|--------|-------|-------------|
| `Consument` | `consuments` | Billing party (patient or institution) |
| `Transaction` | `transactions` | Billing transaction per encounter |
| `TransactionItem` | `transaction_items` | Line items within a transaction |
| `TransactionHasConsument` | `transaction_has_consuments` | Many-to-many: transaction ↔ consument |
| `Billing` | `billings` | Authorization record per billing party |
| `Invoice` | `invoices` | Payment obligation record |
| `PaymentSummary` | `payment_summaries` | Financial totals per transaction |
| `PaymentDetail` | `payment_details` | Per-item payment breakdown |
| `SplitPayment` | `split_payments` | Multi-method payment record |
| `Refund` | `refunds` | Refund record per invoice |
| `WalletTransaction` | `wallet_transactions` | Wallet movement record |
| `Activity` | `activities` | Audit/status trail |
| `ActivityStatus` | `activity_statuses` | Status history per activity |
| `PaymentHasModel` | `payment_has_models` | Polymorphic payment-to-model link |

---

## 8. Backbone Proxy Pattern

8.1 Gateway never calls cashier directly. The call chain is: Gateway → Backbone → Cashier.

8.2 Backbone proxies these services to cashier:

| Backbone Service | Proxied To | Env Var |
|-----------------|------------|---------|
| `TransactionService` | `wellmed-cashier:50053` | `CASHIER_GRPC_ADDRESS` |
| `BillingService` | `wellmed-cashier:50053` | `CASHIER_GRPC_ADDRESS` |
| `InvoiceService` | `wellmed-cashier:50053` | `CASHIER_GRPC_ADDRESS` |

8.3 Auth: backbone injects `x-service-key: $BACKBONE_API_KEY_CASHIER` on proxied calls. Cashier's `AuthInterceptor` validates this key.

---

## 9. Phase 1 Status

Phase 1 (standards alignment) is complete as of 2026-03-06. Changes made:

- Module path: `github.com/kalpa-health/wellmed-cashier` (was `github.com/ingarso/pos`)
- gRPC port: `:50053` (was `:50052`)
- Logger: `go.uber.org/zap` (was logrus)
- Saga callback: gRPC `SagaCallbackService.ReportStepResult` (was RabbitMQ reply)
- Auth interceptor: `x-service-key` per ADR-007 (was absent)
- Entity cleanup: `Consumer` domain deleted; `Consument` is the canonical entity
- Schema cluster: `cashier` (was `pos`)
- Migrations: explicit SQL files via `embed.FS`; `AutoMigrate` removed
- Tests: service layer ≥ 80% coverage; consumer handler tests
- CI: `.github/workflows/` bootstrapped

---

## 10. References

- ADR-005: Saga Orchestration — `kalpa-docs/adrs/ADR-005-saga-orchestration.md`
- ADR-006: Domain Service Boundaries — `kalpa-docs/adrs/ADR-006-domain-service-boundaries.md`
- ADR-007: Per-Service Auth Tokens — `kalpa-docs/adrs/ADR-007-per-service-auth-tokens.md`
- ADR-009: Canonical Archive Pattern — `kalpa-docs/adrs/ADR-009-canonical-archive-pattern.md`
- Backbone service doc: `kalpa-docs/services/backbone.md`
- Phase 1 plan: `kalpa-docs/plans/cashier-standards/cashier-standards-PLAN.md`
