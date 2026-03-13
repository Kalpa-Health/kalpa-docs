# Plan 6: Tenant Configuration & Schema Fixes

**Status**: Planning
**Priority**: High
**Dependencies**: Plan 5 (Migration Testing) ✅

---

## Overview

Implement Go equivalent of PHP's `micro-tenant.php` configuration to properly route entities to correct database/schema. Also fix missing tables/columns discovered during Plan 5 testing.

---

## Background

### PHP MicroTenant Config (Reference)

From `wellmed/config/micro-tenant.php`:

```php
'model_connections' => [
    "central" => [
        "models" => [
            "Country", "Province", "District", "Subdistrict", "Village",
            "User", "Tenant", "Domain", "Encoding", ...
        ]
    ],
    "central_app" => ["models" => []],
    "central_tenant" => ["models" => []]
]
```

**Connection Types**:
- `central` → Main database (`wellmed`) with `public` schema
- `tenant` → Tenant database (`clinic_*`) with `public` schema
- `tenant_yearly` → Tenant database with yearly schema (`emr_2026`, `pos_2026`, `scm_2026`)

### Current Go Issue

Medicine indexes migration tries to create indexes on `emr_2026.items`, `pos_2026.items`, `scm_2026.items` but `items` table only exists in `clinic_*.public` schema.

---

## Phase A: Entity-to-Schema Mapping

### A.1: Define Schema Constants

Create `internal/config/schema_mapping.go`:

```go
package config

type SchemaType string

const (
    SchemaCentral     SchemaType = "central"      // wellmed.public
    SchemaTenant      SchemaType = "tenant"       // clinic_*.public
    SchemaEMR         SchemaType = "emr"          // clinic_*.emr_YYYY
    SchemaPOS         SchemaType = "pos"          // clinic_*.pos_YYYY
    SchemaSCM         SchemaType = "scm"          // clinic_*.scm_YYYY
)

// EntitySchemaMap defines which schema each entity belongs to
var EntitySchemaMap = map[string]SchemaType{
    // Central entities (wellmed.public)
    "Country":     SchemaCentral,
    "Province":    SchemaCentral,
    "District":    SchemaCentral,
    "SubDistrict": SchemaCentral,
    "Village":     SchemaCentral,
    "Disease":     SchemaCentral,
    "User":        SchemaCentral,
    "Tenant":      SchemaCentral,

    // Tenant entities (clinic_*.public)
    "Item":              SchemaTenant,
    "Unicode":           SchemaTenant,
    "Service":           SchemaTenant,
    "ExaminationStuff":  SchemaTenant,
    "Patient":           SchemaTenant,

    // EMR entities (clinic_*.emr_YYYY)
    "VisitPatient":       SchemaEMR,
    "VisitRegistration":  SchemaEMR,
    "VisitExamination":   SchemaEMR,
    "Assessment":         SchemaEMR,
    "Prescription":       SchemaEMR,

    // POS entities (clinic_*.pos_YYYY)
    "Transaction":        SchemaPOS,
    "Billing":            SchemaPOS,
    "Invoice":            SchemaPOS,

    // SCM entities (clinic_*.scm_YYYY)
    "Procurement":        SchemaSCM,
    "PurchaseOrder":      SchemaSCM,
    "ReceiveOrder":       SchemaSCM,
}
```

### A.2: Update ConnectionManager

Modify `internal/db/config/connection_manager.go` to:
- Accept schema type parameter
- Return connection with correct `search_path`

### A.3: Update Migration Commands

Modify migrations to check entity schema before creating indexes.

---

## Phase B: Fix Missing Tables

### B.1: Add `examination_stuffs` Entity

Create `internal/schema/entity/examination_stuff.go`:
- Similar to `Unicode` but for GCS, Allergy types
- Used by ExaminationStuff seeder

### B.2: Add `services` Entity

Create `internal/schema/entity/service.go`:
- Medical services (Rawat Jalan, Laboratorium, etc.)
- Used by MedicService seeder

### B.3: Add `diseases.parent_id` Column

Update `internal/schema/entity/disease.go`:
- Add `ParentID` field for ICD hierarchical structure

---

## Phase C: Fix Medicine Index Migration

### C.1: Update `medicine_indexes.go`

Only create indexes on `public.items` (not yearly schemas):

```go
func MigrateMedicineIndexes(db *gorm.DB, dbName string, schemas []string) error {
    // Only target public schema for items table
    schema := "public"
    tableName := fmt.Sprintf(`"%s"."items"`, schema)
    // ... create index
}
```

---

## Phase D: Testing

### D.1: Test Schema Routing
- Verify entities go to correct schema
- Test migration on fresh database

### D.2: Test Seeding
- Run `make migrate-fresh-seed`
- Verify all seeders complete without errors

---

## Deliverables

1. `internal/config/schema_mapping.go` - Entity-to-schema configuration
2. `internal/schema/entity/examination_stuff.go` - New entity
3. `internal/schema/entity/service.go` - New entity
4. Updated `disease.go` with `parent_id`
5. Fixed `medicine_indexes.go`
6. All seeders passing

---

## Notes

- This is the Go equivalent of PHP's BaseModel connection resolution
- Schema routing should be automatic based on entity type
- Consider using GORM table name callbacks for schema prefixing
