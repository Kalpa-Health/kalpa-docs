# Plan 6: Schema Fixes & Entity Patterns

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

## Phase E: Entity Resource Pattern

### PHP Pattern Analysis

Di PHP, setiap entity bisa define:
- `getViewResource()` - Resource class untuk list view (ringkas)
- `getShowResource()` - Resource class untuk detail view (lengkap)
- `viewUsingRelation()` - Relations otomatis di-load saat list
- `showUsingRelation()` - Relations otomatis di-load saat show

```php
// PHP Pattern (Patient.php)
class Patient extends BaseModel {
    public function getViewResource() {
        return \Projects\WellmedBackbone\Resources\Patient\ViewPatient::class;
    }

    public function getShowResource() {
        return \Projects\WellmedBackbone\Resources\Patient\ShowPatient::class;
    }

    public function viewUsingRelation(): array {
        return ['patientType', 'familyRole'];
    }

    public function showUsingRelation(): array {
        return ['patientType', 'familyRole', 'identity', 'people' => ['education', 'profession']];
    }
}
```

### Go Implementation Plan

#### E.1: Create Resource Interface

**File**: `internal/domain/common/resource.go`

```go
package common

// ResourceProvider defines how an entity provides resources for API response
type ResourceProvider interface {
    // ViewResource returns fields for list view
    ToViewResource() map[string]any
    // ShowResource returns fields for detail view
    ToShowResource() map[string]any
}

// RelationLoader defines what relations to preload
type RelationLoader interface {
    // ViewRelations returns relation names to preload for list view
    ViewRelations() []string
    // ShowRelations returns relation names to preload for detail view
    ShowRelations() []string
}
```

#### E.2: Implement on Patient Entity

**File**: `internal/schema/entity/patient.go`

```go
func (p *Patient) ToViewResource() map[string]any {
    return map[string]any{
        "id":           p.Id,
        "medical_no":   p.MedicalNo,
        "people_name":  p.PeopleName,
        "patient_type": p.PatientType, // preloaded
    }
}

func (p *Patient) ToShowResource() map[string]any {
    view := p.ToViewResource()
    view["address"] = p.Address
    view["identity"] = p.Identity // preloaded
    view["people"] = p.People     // preloaded with education, profession
    return view
}

func (p *Patient) ViewRelations() []string {
    return []string{"PatientType", "FamilyRole"}
}

func (p *Patient) ShowRelations() []string {
    return []string{"PatientType", "FamilyRole", "Identity", "People.Education", "People.Profession"}
}
```

#### E.3: Create Generic Handler Helper

**File**: `internal/api/helper/resource.go`

```go
func PreloadForView[T RelationLoader](db *gorm.DB, entity T) *gorm.DB {
    for _, rel := range entity.ViewRelations() {
        db = db.Preload(rel)
    }
    return db
}

func PreloadForShow[T RelationLoader](db *gorm.DB, entity T) *gorm.DB {
    for _, rel := range entity.ShowRelations() {
        db = db.Preload(rel)
    }
    return db
}
```

---

## Phase F: Filter Casts & Auto Search

### PHP Pattern Analysis

Filter Casts memungkinkan entity define:
- Field mana saja yang bisa difilter
- Tipe filter per field (text, date, select, etc.)
- `auto_search` prefix untuk full-text search otomatis

```php
// PHP Pattern (Patient.php)
public function getFilterCasts(): array {
    return [
        'search_medical_no'       => 'string',
        'search_people_name'      => 'string',
        'search_birthdate'        => 'date',
        'patient_type_id'         => 'string',  // exact match
        'funding_id'              => 'string',
    ];
}

public function getCustomFilterOptions(string $filter_name): array {
    return match ($filter_name) {
        'patient_type_id' => PatientType::pluck('name', 'id')->toArray(),
        'funding_id'      => Funding::pluck('name', 'id')->toArray(),
        default           => [],
    };
}
```

### Go Implementation Plan

#### F.1: Create Filter Registry

**File**: `internal/domain/common/filter.go`

```go
package common

type FilterType string

const (
    FilterTypeString   FilterType = "string"
    FilterTypeDate     FilterType = "date"
    FilterTypeDatetime FilterType = "datetime"
    FilterTypeSelect   FilterType = "select"
    FilterTypeBoolean  FilterType = "boolean"
    FilterTypeNumber   FilterType = "number"
)

type FilterCast struct {
    Field      string      `json:"field"`
    Type       FilterType  `json:"type"`
    Label      string      `json:"label"`
    IsSearch   bool        `json:"is_search"`   // auto_search prefix
    Operators  []string    `json:"operators"`   // eq, like, gte, lte, between
    Options    []Option    `json:"options,omitempty"` // for select type
}

type Option struct {
    Value string `json:"value"`
    Label string `json:"label"`
}

// FilterProvider interface for entities
type FilterProvider interface {
    GetFilterCasts() []FilterCast
}
```

#### F.2: Implement Filter Registry for Patient

**File**: `internal/domain/patient/filter.go`

```go
package patient

func (p *Patient) GetFilterCasts() []common.FilterCast {
    return []common.FilterCast{
        {Field: "medical_no", Type: common.FilterTypeString, Label: "No. Rekam Medis", IsSearch: true, Operators: []string{"like"}},
        {Field: "people_name", Type: common.FilterTypeString, Label: "Nama Pasien", IsSearch: true, Operators: []string{"like"}},
        {Field: "birthdate", Type: common.FilterTypeDate, Label: "Tanggal Lahir", IsSearch: true, Operators: []string{"eq", "gte", "lte", "between"}},
        {Field: "patient_type_id", Type: common.FilterTypeSelect, Label: "Tipe Pasien", Operators: []string{"eq"}},
        {Field: "funding_id", Type: common.FilterTypeSelect, Label: "Penjamin", Operators: []string{"eq"}},
    }
}

func GetPatientTypeOptions(ctx context.Context, db *gorm.DB) []common.Option {
    var types []entity.PatientType
    db.Find(&types)
    opts := make([]common.Option, len(types))
    for i, t := range types {
        opts[i] = common.Option{Value: t.Id, Label: t.Name}
    }
    return opts
}
```

#### F.3: Auto Search Query Builder

**File**: `internal/api/helper/filter.go`

```go
package helper

// ApplyFilters applies filter casts to query
// Handles "search_" prefix for auto_search fields
func ApplyFilters(db *gorm.DB, casts []common.FilterCast, params map[string]string) *gorm.DB {
    for _, cast := range casts {
        // Check for search_ prefix (auto_search)
        searchKey := "search_" + cast.Field
        if val, ok := params[searchKey]; ok && cast.IsSearch {
            db = applyOperator(db, cast.Field, "like", val)
            continue
        }

        // Check for exact field match
        if val, ok := params[cast.Field]; ok {
            db = applyOperator(db, cast.Field, "eq", val)
        }
    }
    return db
}

func applyOperator(db *gorm.DB, field, op, value string) *gorm.DB {
    switch op {
    case "like":
        return db.Where(field+" ILIKE ?", "%"+value+"%")
    case "eq":
        return db.Where(field+" = ?", value)
    case "gte":
        return db.Where(field+" >= ?", value)
    case "lte":
        return db.Where(field+" <= ?", value)
    case "between":
        // value format: "2026-01-01,2026-12-31"
        parts := strings.Split(value, ",")
        if len(parts) == 2 {
            return db.Where(field+" BETWEEN ? AND ?", parts[0], parts[1])
        }
    }
    return db
}
```

---

## Phase G: Props Query (JSON Search)

### PHP Pattern Analysis

Props Query memungkinkan search di dalam field JSONB:

```php
// PHP: search inside props JSON
?filter[props->identity_no]=123456
?filter[props->phone_number]=08123

// Implemented via HasPropsQuery trait
```

### Go Implementation Plan

#### G.1: Props Query Builder

**File**: `internal/api/helper/props_query.go`

```go
package helper

// ApplyPropsFilters handles JSONB field queries
// Query format: props.identity_no=123456
func ApplyPropsFilters(db *gorm.DB, params map[string]string) *gorm.DB {
    for key, val := range params {
        if strings.HasPrefix(key, "props.") {
            jsonPath := strings.TrimPrefix(key, "props.")
            // PostgreSQL JSONB query: props->>'identity_no' ILIKE '%123456%'
            db = db.Where("props->>? ILIKE ?", jsonPath, "%"+val+"%")
        }
    }
    return db
}
```

---

## Phase H: Filter Metadata Injection

### PHP Pattern Analysis

Di PHP, filter metadata otomatis di-inject ke response:

```php
// Response.php
class Response {
    public function collection($collection, $model = null) {
        $response = [
            'data' => $collection,
            'meta' => [
                'pagination' => [...],
                'filter' => $this->getFilterMetadata($model), // injected!
            ]
        ];
    }

    protected function getFilterMetadata($model) {
        if ($model instanceof FilterProvider) {
            return [
                'casts' => $model->getFilterCasts(),
                'options' => $this->getFilterOptions($model),
            ];
        }
    }
}
```

Frontend dapat langsung generate filter UI dari metadata ini.

### Go Implementation Plan

#### H.1: Filter Metadata Response

**File**: `internal/api/response/filter_meta.go`

```go
package response

type FilterMetadata struct {
    Casts   []common.FilterCast     `json:"casts"`
    Options map[string][]common.Option `json:"options"` // field -> options
}

// BuildFilterMetadata generates filter metadata for frontend
func BuildFilterMetadata(provider common.FilterProvider, options map[string][]common.Option) FilterMetadata {
    return FilterMetadata{
        Casts:   provider.GetFilterCasts(),
        Options: options,
    }
}
```

#### H.2: Paginated Response with Filter Metadata

**File**: `internal/api/response/paginated.go`

```go
package response

type PaginatedResponse struct {
    Data       any             `json:"data"`
    Pagination PaginationMeta  `json:"pagination"`
    Filter     *FilterMetadata `json:"filter,omitempty"`
}

func NewPaginatedWithFilter(data any, pagination PaginationMeta, filter FilterMetadata) PaginatedResponse {
    return PaginatedResponse{
        Data:       data,
        Pagination: pagination,
        Filter:     &filter,
    }
}
```

---

## Phase I: Elasticsearch Integration (Future-Ready)

### PHP Pattern Analysis

```php
// PHP Pattern (Patient.php)
protected array $elastic_config = [
    'enabled'     => true,
    'index_name'  => 'patient',
    'searchable'  => ['people_name', 'medical_no', 'props.identity_no'],
    'filterable'  => ['patient_type_id', 'funding_id', 'birthdate'],
    'sortable'    => ['people_name', 'created_at'],
];

// Auto-sync via model events (created, updated, deleted)
```

### Go Implementation Plan

#### I.1: Elasticsearch Config Interface

**File**: `internal/domain/common/elastic.go`

```go
package common

type ElasticConfig struct {
    Enabled     bool     `json:"enabled"`
    IndexName   string   `json:"index_name"`
    Searchable  []string `json:"searchable"`
    Filterable  []string `json:"filterable"`
    Sortable    []string `json:"sortable"`
}

type ElasticProvider interface {
    GetElasticConfig() ElasticConfig
    ToElasticDocument() map[string]any
}
```

#### I.2: Elasticsearch Sync Service

**File**: `internal/infrastructure/elastic/sync.go`

```go
package elastic

type SyncService struct {
    client *elasticsearch.Client
}

// SyncEntity syncs entity to Elasticsearch
func (s *SyncService) SyncEntity(ctx context.Context, entity ElasticProvider, action string) error {
    config := entity.GetElasticConfig()
    if !config.Enabled {
        return nil
    }

    switch action {
    case "create", "update":
        return s.Index(ctx, config.IndexName, entity.ToElasticDocument())
    case "delete":
        return s.Delete(ctx, config.IndexName, entity.GetID())
    }
    return nil
}
```

#### I.3: GORM Callback for Auto-Sync

```go
// Register in entity initialization
db.Callback().Create().After("gorm:after_create").Register("elastic:sync_create", func(db *gorm.DB) {
    if provider, ok := db.Statement.Dest.(ElasticProvider); ok {
        elasticService.SyncEntity(db.Statement.Context, provider, "create")
    }
})
```

---

## Phase J: Documentation

### J.1: Document Go Cluster Architecture

Create `wellmed-backbone/.claude/memory/schema-clusters.md`:

- How clusters work (emr, pos, scm)
- GORM callback mechanism
- Yearly schema auto-creation
- When to add new cluster vs use public schema

### J.2: Document Entity-Schema Mapping

| Schema | Entities | Migration |
|--------|----------|-----------|
| wellmed.public | Country, Province, Disease, Village, Tenant | main/migration.go |
| clinic_*.public | Patient, Item, Unicode, Service, ExaminationStuff | main/migration.go |
| clinic_*.emr_YYYY | VisitPatient, VisitExamination, Assessment, etc. | emr/migration.go |
| clinic_*.pos_YYYY | Transaction, Billing, Invoice, PaymentSummary, etc. | pos/migration.go |
| clinic_*.scm_YYYY | Procurement, PurchaseOrder, CardStock, etc. | scm/migration.go |

### J.3: Document Entity Patterns

Create `wellmed-backbone/.claude/memory/entity-patterns.md`:

- Resource Pattern (View/Show)
- Filter Casts
- Props Query
- Filter Metadata Injection

---

## Deliverables

### Schema Fixes (Priority 1)
1. ✅ Analysis: model_connections not needed in Go microservices
2. `internal/schema/entity/disease.go` - Add parent_id
3. `internal/schema/entity/examination_stuff.go` - New entity
4. `internal/schema/entity/service.go` - New entity
5. `internal/db/migration/medicine_indexes.go` - Fix to target public only
6. `internal/db/config/schema_registry.go` - Add SCM cluster

### Entity Patterns (Priority 2)
7. `internal/domain/common/resource.go` - Resource interfaces
8. `internal/domain/common/filter.go` - Filter registry
9. `internal/api/helper/filter.go` - Filter query builder
10. `internal/api/helper/props_query.go` - JSONB query builder
11. `internal/api/response/filter_meta.go` - Filter metadata response

### Elasticsearch (Priority 3 - Future Ready)
12. `internal/domain/common/elastic.go` - Elastic config interface
13. `internal/infrastructure/elastic/sync.go` - Sync service
14. GORM callbacks for auto-sync

### Documentation
15. `.claude/memory/schema-clusters.md`
16. `.claude/memory/entity-patterns.md`

---

## Implementation Order

1. **Week 1**: Phase A-D (Schema Fixes)
   - Fix missing tables
   - Add SCM cluster
   - Test with seeder

2. **Week 2**: Phase E-H (Entity Patterns)
   - Resource Pattern
   - Filter Casts
   - Props Query
   - Filter Metadata

3. **Week 3**: Phase I-J (Elastic + Docs)
   - Elasticsearch integration
   - Documentation

---

## Notes

- **No PHP-style model_connections needed** - Go microservices handle this via domain separation
- **GORM callbacks already handle cluster routing** - No additional config needed
- **Cross-service access via gRPC** - Not direct DB, so no cross-schema transactions within service
- **Saga pattern for distributed transactions** - Already implemented in backbone
- **Filter metadata injection** - Enables frontend to auto-generate filter UI
- **Elasticsearch auto-sync** - Via GORM callbacks, similar to Laravel model events
