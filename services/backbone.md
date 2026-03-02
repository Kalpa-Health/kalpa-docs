# Service: Backbone

**Version:** 1.0
**Date:** 2026-03-02
**Status:** Active — core gRPC backend
**Maintained by:** Alex (Kalpa Inovasi Digital)

---

## 1. Overview

1.1 Backbone is the core internal gRPC backend for WellMed. It owns all business logic, direct database access, multi-tenant architecture, and the saga framework for distributed transactions.

1.2 The gateway (`wellmed-gateway-go`) acts as a thin HTTP-to-gRPC proxy. All real work happens in backbone.

1.3 For the full system context, see [`wellmed-system-architecture.md §2.1`](../wellmed-system-architecture.md).

---

## 2. Repository

| Item | Value |
|------|-------|
| **Repo** | `wellmed-backbone` |
| **Language** | Go 1.25+ |
| **gRPC address** | `BACKBONE_GRPC_ADDRESS` (env on gateway) |
| **gRPC port** | `:50051` |
| **Module** | `github.com/ingarso/wellmed-backbone` |

---

## 3. Key Responsibilities

- **Business logic** for all 19 domain services (patient, pharmacy_sale, visit_patient, etc.)
- **Multi-tenant database access** — one PostgreSQL database per tenant, year-based schema isolation
- **Auth** — JWT issuance, validation, Redis session management
- **Saga framework** — distributed transactions (SYNC blocking + ASYNC non-blocking + compensation)
- **POS integration** — gRPC client to `wellmed-pos` for Transaction, Billing, Invoice

---

## 4. gRPC Interface Catalog

19 services registered at `internal/app/application.go`:

| # | Service | Key Methods |
|---|---------|-------------|
| 1 | UserService | Login, RefreshToken |
| 2 | UnicodeService | GetAll, GetByFlag |
| 3 | MenuService | GetAll |
| 4 | AssessmentService | Store, GetByVisit |
| 5 | AutoListService | GetAll |
| 6 | TreatmentService | GetAll, Store |
| 7 | RoleService | GetAll, Store |
| 8 | ItemService | GetAll, Store |
| 9 | PatientService | GetAll, GetById, Store |
| 10 | VisitPatientService | GetAll, GetById, Store, UpdateStatus |
| 11 | VisitRegistrationService | GetAll, Store |
| 12 | VisitRegistrationReferralService | GetAll, Store |
| 13 | VisitExaminationService | GetAll, Store |
| 14 | ReferralService | GetAll, Store |
| 15 | FrontlineService | GetAll, Store |
| 16 | PharmacySaleService | GetAll, GetById, Store |
| 17 | TransactionService | → POS pass-through |
| 18 | BillingService | → POS pass-through |
| 19 | InvoiceService | → POS pass-through |

---

## 5. Multi-Tenant Design

5.1 Each tenant (clinic) has a dedicated PostgreSQL database. Data is further partitioned by year-based schema (`schema_2026`, `schema_2027`, etc.).

5.2 The `ConnectionManager` manages lazy connection pooling per database. On first use, a connection is opened and cached.

5.3 Tenant identity flows from JWT claims (`tenant_db`, `tenant_schema`) → `ConnectionManager` → `SET LOCAL search_path` on each query session.

5.4 DB and schema identifiers are validated before use as PostgreSQL identifiers to prevent SQL injection.

---

## 6. Saga Framework

6.1 Backbone uses a generic saga framework (`internal/saga/`) for multi-step operations that require atomic execution with compensation on failure.

6.2 Two execution modes:

| Mode | Blocks? | Example |
|------|---------|---------|
| SYNC | Yes (caller waits) | `pharmacy_sale` — needs POS transaction ID for receipt |
| ASYNC | No (returns saga_id) | `patient` create — long multi-step registration |

6.3 All compensations are stored in saga context (JSONB) so they survive process restarts.

6.4 Full documentation: [`wellmed-backbone/internal/saga/README.md`](../../wellmed-backbone/internal/saga/README.md)

---

## 7. Auth Architecture

7.1 JWT with HMAC-SHA256, stored in Redis with JTI per session (single-session model).

7.2 Whitelist endpoints: `/user.UserService/Login`, `/user.UserService/RefreshToken`.

7.3 Token carries full tenant context: `tenant_db`, `tenant_schema`, `tenant_id`, `user_id`, `employee_id`.

7.4 Session invalidation: login replaces Redis JTI, invalidating all previous tokens for that user.

---

## 8. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Database-per-tenant | Strong isolation; tenant data never co-mingled |
| ULID for all IDs | Lexicographically sortable, globally unique, no UUID collision risk |
| JSONB `props` column | Flexible properties without schema migrations for non-indexed data |
| Saga framework | Compensating transactions across gRPC boundaries without 2PC |
| No HTTP in backbone | gRPC is the only interface — protocol enforced at gateway |

---

## 9. Service-Specific Documentation

| Document | Location |
|----------|---------|
| Directory structure | `wellmed-backbone/docs/backbone-structure.md` |
| Domain modules | `wellmed-backbone/docs/backbone-modules.md` |
| Infrastructure (saga, DB, Redis) | `wellmed-backbone/docs/backbone-infrastructure.md` |
| Middleware & auth | `wellmed-backbone/docs/backbone-middleware.md` |
| Test baseline | `wellmed-backbone/docs/backbone-test-baseline.md` |
| Service docs | `wellmed-backbone/docs/services/` |
| Saga framework | `wellmed-backbone/internal/saga/README.md` |

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 01 Mar 2026 | Alex + Claude | Initial stub |
| 1.0 | 02 Mar 2026 | Alex + Claude | Full service doc — all sections complete |
