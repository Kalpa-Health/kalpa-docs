# Plan 6: Schema Fixes & Entity Patterns - Progress

**Plan File**: `6-tenant-config-schema-PLAN.md`
**Started**: 2026-03-13
**Status**: 🚧 IN PROGRESS

---

## Pre-Implementation Analysis ✅ COMPLETE

- [x] Analysis: `model_connections` NOT needed in Go microservices
- [x] Documented why: Domain-specific services handle routing, not config table
- [x] Identified 3 missing entities: Disease.ParentID, ExaminationStuff, Service

---

## Phase A: Fix Missing Tables 🔲 PENDING

### A.1: Add `diseases.parent_id` Column 🔲 PENDING

**File**: `internal/schema/entity/disease.go`

- [ ] Add `ParentID *string` field
- [ ] Add `Parent *Disease` relation
- [ ] Add `Children []Disease` relation
- [ ] Run migration to verify

### A.2: Add `examination_stuffs` Entity 🔲 PENDING

**File**: `internal/schema/entity/examination_stuff.go`

- [ ] Create new entity file
- [ ] Add all fields (id, flag, label, name, status, ordering, props, parent_id)
- [ ] Add self-referential parent/child relations
- [ ] Register in migration runner

### A.3: Add `services` Entity 🔲 PENDING

**File**: `internal/schema/entity/service.go`

- [ ] Create new entity file
- [ ] Add all fields (id, name, reference, parent, status, price, cogs, margin, props)
- [ ] Register in migration runner

---

## Phase B: Fix Medicine Index Migration 🔲 PENDING

### B.1: Update `medicine_indexes.go` 🔲 PENDING

- [ ] Filter to only process `public` schema (not yearly schemas)
- [ ] `items` table only exists in `public`, not in `emr_YYYY` or `pos_YYYY`
- [ ] Test index creation

---

## Phase C: Add SCM Cluster to Registry 🔲 PENDING

### C.1: Add SCM entities to schema_registry.go 🔲 PENDING

- [ ] Define SCM cluster in registry
- [ ] Add models: CardStock, Procurement, PurchaseRequest, PurchaseOrder, ReceiveOrder, Distribution, WorkOrder
- [ ] Test with GORM callback resolution

---

## Phase D: Update Main Migration 🔲 PENDING

### D.1: Add new entities to migration runner 🔲 PENDING

- [ ] Add `ExaminationStuff` to AutoMigrate
- [ ] Add `Service` to AutoMigrate
- [ ] Run `make migrate` to verify

---

## Phase E: Entity Resource Pattern 🔲 PENDING

### E.1: Create Resource Interface 🔲 PENDING

**File**: `internal/domain/common/resource.go`

- [ ] Define `ResourceProvider` interface (ToViewResource, ToShowResource)
- [ ] Define `RelationLoader` interface (ViewRelations, ShowRelations)

### E.2: Implement on Patient Entity 🔲 PENDING

- [ ] Implement ToViewResource()
- [ ] Implement ToShowResource()
- [ ] Implement ViewRelations()
- [ ] Implement ShowRelations()

### E.3: Create Generic Handler Helper 🔲 PENDING

**File**: `internal/api/helper/resource.go`

- [ ] Implement PreloadForView()
- [ ] Implement PreloadForShow()

---

## Phase F: Filter Casts & Auto Search 🔲 PENDING

### F.1: Create Filter Registry 🔲 PENDING

**File**: `internal/domain/common/filter.go`

- [ ] Define FilterType constants
- [ ] Define FilterCast struct
- [ ] Define FilterProvider interface

### F.2: Implement Filter Registry for Patient 🔲 PENDING

- [ ] Implement GetFilterCasts() for Patient
- [ ] Implement GetPatientTypeOptions()
- [ ] Implement GetFundingOptions()

### F.3: Auto Search Query Builder 🔲 PENDING

**File**: `internal/api/helper/filter.go`

- [ ] Implement ApplyFilters()
- [ ] Handle `search_` prefix for auto_search fields
- [ ] Support operators: eq, like, gte, lte, between

---

## Phase G: Props Query (JSON Search) 🔲 PENDING

### G.1: Props Query Builder 🔲 PENDING

**File**: `internal/api/helper/props_query.go`

- [ ] Implement ApplyPropsFilters()
- [ ] Handle `props.` prefix for JSONB queries
- [ ] PostgreSQL: `props->>'field' ILIKE '%value%'`

---

## Phase H: Filter Metadata Injection 🔲 PENDING

### H.1: Filter Metadata Response 🔲 PENDING

**File**: `internal/api/response/filter_meta.go`

- [ ] Define FilterMetadata struct
- [ ] Implement BuildFilterMetadata()

### H.2: Paginated Response with Filter Metadata 🔲 PENDING

**File**: `internal/api/response/paginated.go`

- [ ] Add Filter field to PaginatedResponse
- [ ] Implement NewPaginatedWithFilter()

---

## Phase I: Elasticsearch Integration 🔲 FUTURE

*Priority 3 - Future Ready*

- [ ] Define ElasticConfig interface
- [ ] Implement SyncService
- [ ] Register GORM callbacks for auto-sync

---

## Phase J: Documentation 🔲 PENDING

### J.1: Document Go Cluster Architecture 🔲 PENDING

- [ ] Create `.claude/memory/schema-clusters.md`
- [ ] Document cluster mechanism (emr, pos, scm)
- [ ] Document GORM callback flow

### J.2: Document Entity-Schema Mapping 🔲 PENDING

- [ ] Map all entities to their target schemas
- [ ] Document migration file locations

### J.3: Document Entity Patterns 🔲 PENDING

- [ ] Create `.claude/memory/entity-patterns.md`
- [ ] Document Resource Pattern
- [ ] Document Filter Casts
- [ ] Document Props Query

---

## Summary

| Phase | Description | Status |
|-------|-------------|--------|
| Analysis | model_connections not needed | ✅ Complete |
| Phase A | Fix Missing Tables | 🔲 Pending |
| Phase B | Fix Medicine Index | 🔲 Pending |
| Phase C | Add SCM Cluster | 🔲 Pending |
| Phase D | Update Migration | 🔲 Pending |
| Phase E | Resource Pattern | 🔲 Pending |
| Phase F | Filter Casts | 🔲 Pending |
| Phase G | Props Query | 🔲 Pending |
| Phase H | Filter Metadata | 🔲 Pending |
| Phase I | Elasticsearch | 🔲 Future |
| Phase J | Documentation | 🔲 Pending |

---

## Commits

*(To be updated as implementation progresses)*

---

## Dependencies

- ✅ Plan 5: Migration & Seed Testing - COMPLETE
- Plan 6 addresses schema issues found during Plan 5 testing:
  - `diseases.parent_id` missing
  - `examination_stuffs` table missing
  - `services` table missing
