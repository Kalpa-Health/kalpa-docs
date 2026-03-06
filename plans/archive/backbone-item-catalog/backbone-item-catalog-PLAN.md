# backbone-item-catalog-PLAN.md

**Status:** Implemented
**Date:** 2026-03-04
**Output path:** `kalpa-docs/plans/backbone-item-catalog/backbone-item-catalog-PLAN.md`
**Codebase:** `/Users/alexknecht/Projects/WellMed/wellmed-backbone`

---

## 1. Context

The Backbone module has no canonical owner for billable item definitions and pricing. Pricing is at risk of being embedded ad-hoc inside domain packages (Consultation, Lab, Radiology). The Saga orchestrator has no stable, isolated structure to read item identity and price from during transaction processing.

This plan creates the `itemcatalog` package — a clean, isolated domain package inside `internal/domain/` — designed from the start as a future MS extraction boundary. It introduces `CatalogService`, the single authority for item definitions and price resolution, callable by the Saga as a Local step.

**Key design decisions resolved:**
- IDs: ULID `char(26)` using `helper.GenerateUlid()` (matches codebase convention)
- ORM: GORM AutoMigrate (matches existing migration pattern)
- Item registration: `RegisterItem()` on `CatalogService` — called as a Saga Local step when domain MS registers a new service definition
- Composite pricing: return typed error if no parent price; no sum-of-children in this phase
- Default pricelist: `is_default=true` on pricelist record, enforced by partial unique index

---

## 2. Critical Files

| File | Role |
|------|------|
| `internal/domain/item/wires.go` | Reference pattern for wires.go |
| `internal/domain/pharmacy_sale/saga/builder.go` | Reference for Local saga step calling a service |
| `internal/db/migration/migration.go` | Reference for AutoMigrate pattern |
| `internal/app/application.go` | Wire new module into app bootstrap |
| `internal/helper/general.go` | `GenerateUlid()` for ID generation |
| `internal/db/config/` | `ConnectionManager`, `ConnectionOrchestrator`, `TransactionManager` |
| `internal/testutil/` | Mock patterns for unit tests |

---

## 3. Package Structure Created

```
internal/domain/itemcatalog/
├── model/
│   ├── item.go              # CatalogItem GORM entity + ItemType enum constants
│   ├── pricelist.go         # Pricelist GORM entity
│   ├── pricelist_entry.go   # PricelistEntry GORM entity
│   └── payer_override.go    # PayerItemOverride GORM entity (schema only)
├── repository/
│   ├── item.go              # ItemRepository interface + private impl
│   └── price.go             # PriceRepository interface + private impl
├── service/
│   ├── catalog.go           # CatalogService interface + implementation
│   └── catalog_test.go      # Table-driven unit tests (9 test cases)
├── dto/
│   ├── item.go              # RegisterItemRequest, ItemResult, BatchItemRequest
│   └── price.go             # PriceRequest, PriceResult, typed errors
├── migration/
│   ├── migrate.go           # AutoMigrate all 4 tables + indexes
│   └── seed.go              # Seed function for dev/test data
└── wires.go                 # Module struct + NewModule constructor
```

---

## 4. Tasks

### 4.1 Models

- [x] **Create `internal/domain/itemcatalog/model/item.go`**
- [x] **Create `internal/domain/itemcatalog/model/pricelist.go`**
- [x] **Create `internal/domain/itemcatalog/model/pricelist_entry.go`**
- [x] **Create `internal/domain/itemcatalog/model/payer_override.go`**

### 4.2 DTOs

- [x] **Create `internal/domain/itemcatalog/dto/item.go`**
- [x] **Create `internal/domain/itemcatalog/dto/price.go`**

### 4.3 Repositories

- [x] **Create `internal/domain/itemcatalog/repository/item.go`**
- [x] **Create `internal/domain/itemcatalog/repository/price.go`**

### 4.4 Service Layer

- [x] **Create `internal/domain/itemcatalog/service/catalog.go`**

### 4.5 Migration & Seed

- [x] **Create `internal/domain/itemcatalog/migration/migrate.go`**
- [x] **Create `internal/domain/itemcatalog/migration/seed.go`**

### 4.6 Wiring

- [x] **Create `internal/domain/itemcatalog/wires.go`**
- [x] **Wire `itemcatalog` module into `internal/app/application.go`**

### 4.7 Unit Tests

- [x] **Create `internal/domain/itemcatalog/service/catalog_test.go`**

### 4.8 Create Canonical Plan File

- [x] **Create `kalpa-docs/plans/backbone-item-catalog/backbone-item-catalog-PLAN.md`**

---

## 5. Acceptance Criteria Cross-Reference

| PRD AC | Satisfied by |
|--------|-------------|
| 4.1 No import deps on other backbone packages | Package only imports `internal/db/config`, `internal/helper`, `internal/tenant` |
| 4.2 Migrations, no FK to other packages | AutoMigrate in `migration/migrate.go`; all FKs are self-contained |
| 4.3 Batch item lookup single round-trip | `FindByIDs` + `FindChildrenByParentIDs` = 2 queries max |
| 4.4 Batch price resolution minimal round-trips | 1 query per distinct pricelist group |
| 4.5 nil pricelist_id → default pricelist | `FindDefaultPricelist` + `is_default = true` logic |
| 4.6 Missing price → typed error, batch continues | `PriceResult.Err = ErrNoPriceFound` |
| 4.7 Effective dating enforced | `effective_from <= asOfDate` in SQL + max selection in Go |
| 4.8 Composite with parent price → parent price | Service returns parent entry directly |
| 4.9 Non-null currency | `Currency default:'IDR'` on `PricelistEntry` model |
| 4.10 UoM required for obat/consumable | Service-layer validation in `RegisterItem` |
| 4.11 payer_item_overrides exists, accepts inserts | Model + AutoMigrate, no service logic |
| 4.12 Saga calls CatalogService via interface only | Only `service.CatalogService` interface exported |
| 4.13 Seed data | `SeedItemCatalog` covers all item_types + composite + pricelist |

---

## 6. Verification

```bash
# 1. Build — no compile errors, no import cycles
cd wellmed-backbone && go build ./...

# 2. Run new package tests
go test ./internal/domain/itemcatalog/...

# 3. Lint
golangci-lint run ./internal/domain/itemcatalog/...

# 4. Migration smoke test (against local dev DB)
# Confirm catalog_items, pricelists, pricelist_entries, payer_item_overrides tables created

# 5. Seed verification
# Run SeedItemCatalog for a test clinic_id

# 6. Saga integration smoke test
# Verify a Local saga step can call catalogModule.Service.RegisterItem() without importing model types
```
