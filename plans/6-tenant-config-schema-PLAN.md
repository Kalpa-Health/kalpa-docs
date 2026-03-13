# Plan 6: Schema Fixes & Cluster Documentation

**Status**: Planning
**Priority**: High
**Dependencies**: Plan 5 (Migration Testing) ✅

---

## Analysis: PHP model_connections vs Go Microservices

### Conclusion: model_connections NOT NEEDED in Go

After analyzing both PHP and Go implementations:

| Aspect | PHP (Monolith) | Go (Microservices) |
|--------|----------------|-------------------|
| Architecture | Single app, all models | Domain-specific services |
| Connection Routing | `model_connections` config | Service owns its domain |
| Cross-domain Access | Direct DB with connection switch | gRPC / Saga |
| Schema Clusters | ProcessClusters + Job | GORM callbacks + ensureSchemas() |

**Why model_connections is not needed:**
1. Each Go service already handles specific domain (Backbone=Patient/Item, Consultation=Visit, Cashier=Transaction)
2. Cross-service data access is via gRPC calls, not direct DB
3. Schema clusters (emr_2026, pos_2026) already working via GORM callbacks
4. Tenant context comes from JWT, not model config

### What Go Already Has (Working)

```
wellmed-backbone/internal/db/config/
├── connection_manager.go   # Multi-tenant connection pooling
├── schema_registry.go      # Cluster definition (emr, pos models)
└── transaction_manager.go  # Multi-DB transaction support
```

**schema_registry.go** defines clusters:
```go
var clusterRegistry = []SchemaCluster{
    {Name: "emr", IsCluster: true, Models: []*emr.VisitPatient{}, ...},
    {Name: "pos", IsCluster: true, Models: []*pos.Transaction{}, ...},
}
```

**GORM Callbacks** auto-resolve schema:
```go
// RegisterDynamicSchemaResolver sets up callbacks for all CRUD operations
// Model "VisitPatient" → table becomes "emr_2026.visit_patients"
```

---

## Phase A: Fix Missing Tables

### A.1: Add `diseases.parent_id` Column

**Problem**: ICD hierarchical seeder needs `parent_id` for tree structure.

**File**: `internal/schema/entity/disease.go`

```go
type Disease struct {
    // ... existing fields
    ParentID  *string  `gorm:"column:parent_id;type:char(26);index"`
    Parent    *Disease `gorm:"foreignKey:ParentID"`
    Children  []Disease `gorm:"foreignKey:ParentID"`
}
```

### A.2: Add `examination_stuffs` Entity

**Problem**: GCS, Allergy seeder needs this table.

**File**: `internal/schema/entity/examination_stuff.go`

```go
type ExaminationStuff struct {
    Id        string         `gorm:"type:char(36);primaryKey"`
    Flag      string         `gorm:"type:varchar(100);not null;uniqueIndex:idx_exam_stuff_flag_name,priority:1"`
    Label     *string        `gorm:"type:varchar(100)"`
    Name      string         `gorm:"type:varchar(100);not null;uniqueIndex:idx_exam_stuff_flag_name,priority:2"`
    Status    *string        `gorm:"type:varchar(100)"`
    Ordering  *int           `gorm:"type:int"`
    Props     datatypes.JSON `gorm:"type:jsonb"`
    ParentID  *string        `gorm:"type:char(36);index"`
    Parent    *ExaminationStuff `gorm:"foreignKey:ParentID"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

### A.3: Add `services` Entity

**Problem**: MedicService seeder needs this table.

**File**: `internal/schema/entity/service.go`

```go
type Service struct {
    Id             string         `gorm:"type:char(36);primaryKey"`
    Name           string         `gorm:"type:varchar(255);not null"`
    ReferenceID    *string        `gorm:"type:char(36);uniqueIndex:idx_services_ref,priority:1"`
    ReferenceType  *string        `gorm:"type:varchar(100);uniqueIndex:idx_services_ref,priority:2"`
    ParentID       *string        `gorm:"type:char(36);index"`
    ServiceLabelID *string        `gorm:"type:char(36)"`
    Status         string         `gorm:"type:varchar(50);default:'ACTIVE'"`
    Price          float64        `gorm:"type:decimal(15,2);default:0"`
    Cogs           float64        `gorm:"type:decimal(15,2);default:0"`
    Margin         float64        `gorm:"type:decimal(15,2);default:0"`
    Props          datatypes.JSON `gorm:"type:jsonb"`
    CreatedAt      time.Time
    UpdatedAt      time.Time
    DeletedAt      gorm.DeletedAt `gorm:"index"`
}
```

---

## Phase B: Fix Medicine Index Migration

### B.1: Update `medicine_indexes.go`

**Problem**: Index created on `emr_2026.items`, `pos_2026.items`, `scm_2026.items` but `items` only exists in `public` schema.

**Solution**: Only create index on `public.items`

```go
func MigrateMedicineIndexes(db *gorm.DB, dbName string, schemas []string) error {
    // Filter: only process schemas that have items table
    // items only exists in "public" schema
    for _, schema := range schemas {
        if schema != "public" {
            continue // Skip yearly schemas for items
        }
        // ... create index
    }
}
```

---

## Phase C: Add SCM Cluster to Registry

### C.1: Add SCM entities to schema_registry.go

Currently missing SCM cluster in Go. Need to add:

```go
{
    Name:      "scm",
    IsCluster: true,
    Models: []any{
        &scm.CardStock{},
        &scm.Procurement{},
        &scm.PurchaseRequest{},
        &scm.PurchaseOrder{},
        &scm.ReceiveOrder{},
        &scm.Distribution{},
        &scm.WorkOrder{},
    },
},
```

---

## Phase D: Update Main Migration

### D.1: Add new entities to migration runner

Update `internal/db/migration/main/migration.go`:

```go
func Migrate(db *gorm.DB) error {
    return db.AutoMigrate(
        // ... existing entities
        &entity.ExaminationStuff{},
        &entity.Service{},
    )
}
```

---

## Phase E: Documentation

### E.1: Document Go Cluster Architecture

Create `wellmed-backbone/.claude/memory/schema-clusters.md`:

- How clusters work (emr, pos, scm)
- GORM callback mechanism
- Yearly schema auto-creation
- When to add new cluster vs use public schema

### E.2: Document Entity-Schema Mapping

| Schema | Entities | Migration |
|--------|----------|-----------|
| wellmed.public | Country, Province, Disease, Village, Tenant | main/migration.go |
| clinic_*.public | Patient, Item, Unicode, Service, ExaminationStuff | main/migration.go |
| clinic_*.emr_YYYY | VisitPatient, VisitExamination, Assessment, etc. | emr/migration.go |
| clinic_*.pos_YYYY | Transaction, Billing, Invoice, PaymentSummary, etc. | pos/migration.go |
| clinic_*.scm_YYYY | Procurement, PurchaseOrder, CardStock, etc. | scm/migration.go |

---

## Deliverables

1. ✅ Analysis: model_connections not needed in Go microservices
2. `internal/schema/entity/disease.go` - Add parent_id
3. `internal/schema/entity/examination_stuff.go` - New entity
4. `internal/schema/entity/service.go` - New entity
5. `internal/db/migration/medicine_indexes.go` - Fix to target public only
6. `internal/db/config/schema_registry.go` - Add SCM cluster
7. Documentation in `.claude/memory/`

---

## Notes

- **No PHP-style model_connections needed** - Go microservices handle this via domain separation
- **GORM callbacks already handle cluster routing** - No additional config needed
- **Cross-service access via gRPC** - Not direct DB, so no cross-schema transactions within service
- **Saga pattern for distributed transactions** - Already implemented in backbone
