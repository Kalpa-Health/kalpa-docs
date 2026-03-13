# 3. Go Seeder Migration Plan

**Status:** Proposed
**Date:** 2026-03-13
**Purpose:** Migrate Laravel seeders to Go dengan konsep yang konsisten
**Output path:** `kalpa-docs/plans/go-seeder-migration/`
**Codebase:** `wellmed-backbone`, `wellmed-consultation`, `wellmed-cashier`

---

## 1. Context

### 1.1 Background

WellMed memiliki 2 project backbone:
- **Laravel (Legacy):** `/var/www/projects/kalpa/wellmed/projects/wellmed-backbone/` dengan 67 PHP seeders
- **Go (New):** `/var/www/projects/kalpa/wellmed-backbone/` dengan 33 Go seeders

Current Go seeders hanya mencakup ~50% dari Laravel seeders. Banyak reference data penting (ClinicalPathology, Radiology, TherapeuticClass, ICD, Geographic) belum di-migrate.

### 1.2 Laravel Seeding Architecture

Laravel menggunakan **3 mode seeding** yang berbeda:

#### Mode 1: Full Install (DatabaseSeeder) — untuk PLUS product / test tenant
```
php artisan wellmed-backbone:install PLUS
↓
WorkspaceSeeder              → Create workspace + tenant
ApiAccessSeeder              → API access tokens
InstallerSeeder              → Core reference data (see below)
RestrictionFeatureSeeder     → Feature restrictions (PLUS)
EmployeeSeeder               → Admin employee (single)
IcdSeeder                    → Disease codes (~13K) ← ONLY full install
PatientSeeder                → Sample patients (~1000) ← ONLY full install
```

#### Mode 2: Lite Install (LiteDatabaseSeeder) — untuk LITE product
```
php artisan wellmed-backbone:install LITE
↓
LiteWorkspaceSeeder          → Lightweight workspace
ApiAccessSeeder              → API access tokens
InstallerSeeder              → Core reference data
RestrictionFeatureSeeder     → Feature restrictions (LITE)
LiteEmployeeSeeder           → Admin + 10 employees
LiteEncodingSeeder           → Lite encoding
IcdSeeder                    → Disease codes ← Included
PatientSeeder                → Sample patients ← Included
ElasticSeeder                → Elasticsearch sync ← LITE only
```

#### Mode 3: New Tenant SAAS (AddDatabaseSeeder) — via REST API background job
```
REST API → Background Job → AddDatabaseSeeder
↓
AddNewTenantSeeder           → Setup new tenant
InstallerSeeder              → Core reference data
UserAdminSeeder              → Admin user only
RestrictionFeatureSeeder     → Feature restrictions
```

**KEY DIFFERENCE:**
- AddDatabaseSeeder **TIDAK** include: IcdSeeder, PatientSeeder
- Karena untuk SAAS, data ini optional atau di-lazy-load

#### InstallerSeeder (SHARED by all modes)
```
PermissionSeeder            → ACL permissions
RoleSeeder                  → User roles
EncodingSeeder              → Document encoding schemas
MasterSeeder                → ALL reference data
AssetSeeder                 → Static assets
MasterReportCollectionSeeder → Report definitions
```

#### MasterSeeder Sources (dari berbagai Laravel repos)
| Source | Seeders |
|--------|---------|
| **ModulePayment** | CoaSeeder, WalletSeeder |
| **ModulePeople** | PeopleCollectionSeeder (FamilyRole, Education) |
| **ModuleExamination** | ExaminationStuffSeeder, MasterVaccineSeeder |
| **ModuleEmployee** | EmployeeTypeSeeder |
| **ModuleAnatomy** | AnatomyCollectionSeeder |
| **ModuleItem** | ItemCollectionSeeder |
| **KlinikStarterpack** | ItemStuffSeeder, PatientTypeSeeder, PatientTypeServiceSeeder, PaymentMethodSeeder, ProfessionSeeder, OccupationSeeder, RegionalSeeder, PatientOccupationSeeder, FundingSeeder, DosageFormSeeder, FreqUnitSeeder, MedicalCompositionUnitSeeder, CompositionUnitSeeder, MedicalNetUnitSeeder, MixUnitSeeder, TherapeuticClassSeeder, PackageFormSeeder, TrademarkSeeder, UsageLocationSeeder, UsageRouteSeeder, BrandSeeder, SoapSeeder, RoomItemCategorySeeder, ClassRoomSeeder, PurchaseLabelSeeder, SampleSeeder, ClinicalPathologySeeder, RadiologySeeder, ProgramCategorySeeder, ProgramOccupationSeeder |
| **Local (wellmed-backbone)** | MedicServiceSeeder, FormSeeder, InfrastructureSeeder, DocumentTypeSeeder |

### 1.3 Objectives

1. Migrate critical Laravel seeders ke Go
2. Maintain **3 seeding modes** yang sama dengan Laravel
3. Support multi-tenant database architecture
4. Implement idempotent seeding (upsert pattern)
5. Support product types: LITE, PLUS, ENTERPRISE

---

## 2. Current State Analysis

### 2.1 Existing Go Seeders (33 files)

**Location:** `wellmed-backbone/internal/db/seed/`

| Category | Files | Status |
|----------|-------|--------|
| Reference Data (Unicode) | 16 | Complete |
| Service Data | 3 | Complete |
| Hierarchical (2-level) | 4 | Complete |
| Encoding | 2 | Partial |
| Form | 1 | Complete |
| Examination Stuff | 1 | Complete |
| Orchestrator | 1 | Partial |

**Current Go modes:**
- `SeedLite()` — 16 seeders (active, minimal reference data)
- `SeedPlus()` — Empty (not implemented)
- `SeedEMR()` — Empty (not implemented)

### 2.2 Target Go Seeding Modes (mirror Laravel)

```
Go Seeder Mode              Laravel Equivalent           Use Case
─────────────────────────────────────────────────────────────────────
SeedFullInstall()           DatabaseSeeder               Full install / test tenant (PLUS)
SeedLiteInstall()           LiteDatabaseSeeder           LITE product install
SeedNewTenant()             AddDatabaseSeeder            SAAS new tenant (REST API)
SeedMaster()                MasterSeeder                 Core reference data (shared)
SeedInstaller()             InstallerSeeder              Permission, Role, Encoding, Master, Asset
```

**Mode Comparison:**

| Component | SeedFullInstall | SeedLiteInstall | SeedNewTenant |
|-----------|-----------------|-----------------|---------------|
| Workspace | WorkspaceSeeder | LiteWorkspace | AddNewTenant |
| API Access | ✅ | ✅ | — |
| Installer (Master) | ✅ | ✅ | ✅ |
| Restriction | ✅ (PLUS) | ✅ (LITE) | ✅ |
| Employee | Single Admin | Admin + 10 | Admin Only |
| ICD Diseases | ✅ (~13K) | ✅ (~13K) | ❌ |
| Sample Patients | ✅ (~1000) | ✅ (~1000) | ❌ |
| Elasticsearch | ❌ | ✅ | ❌ |

### 2.3 MasterSeeder Complete Mapping (Laravel → Go)

**MasterSeeder dipanggil oleh InstallerSeeder dan merupakan inti dari reference data.**

#### From ModulePayment
| Laravel Seeder | Go Status | Go File | Notes |
|----------------|-----------|---------|-------|
| CoaSeeder | ❌ MISSING | — | Chart of Accounts |
| WalletSeeder | ❌ MISSING | — | User wallet types |

#### From ModulePeople (PeopleCollectionSeeder)
| Laravel Seeder | Go Status | Go File | Notes |
|----------------|-----------|---------|-------|
| FamilyRoleSeeder | ✅ | seed_family_role.go | |
| EducationSeeder | ✅ | seed_education.go | |

#### From ModuleExamination
| Laravel Seeder | Go Status | Go File | Notes |
|----------------|-----------|---------|-------|
| ExaminationStuffSeeder | ✅ | seed_examination_stuff.go | GCS, Allergy types |
| MasterVaccineSeeder | ✅ | seed_master_vaccine.go | |

#### From ModuleEmployee
| Laravel Seeder | Go Status | Go File | Notes |
|----------------|-----------|---------|-------|
| EmployeeTypeSeeder | ✅ | seed_employee_type.go | |

#### From ModuleAnatomy (AnatomyCollectionSeeder)
| Laravel Seeder | Go Status | Go File | Notes |
|----------------|-----------|---------|-------|
| AnatomySeeder | ✅ | seed_anatomy.go | Head-to-toe hierarchy |

#### From ModuleItem (ItemCollectionSeeder)
| Laravel Seeder | Go Status | Go File | Notes |
|----------------|-----------|---------|-------|
| ItemCollectionSeeder | ❌ PARTIAL | — | Item reference data |

#### From KlinikStarterpack (30 seeders)
| Laravel Seeder | Go Status | Go File | Priority |
|----------------|-----------|---------|----------|
| ItemStuffSeeder | ✅ | seed_item_stuff.go | — |
| PatientTypeSeeder | ✅ | seed_patient_type_service.go | — |
| PatientTypeServiceSeeder | ✅ | seed_patient_type_service.go | — |
| PaymentMethodSeeder | ✅ | seed_payment_method.go | — |
| ProfessionSeeder | ✅ | seed_profession.go | — |
| OccupationSeeder | ✅ | seed_occupation.go | — |
| RegionalSeeder | ❌ MISSING | seed_regional.go | HIGH |
| PatientOccupationSeeder | ✅ | seed_patient_occupation.go | — |
| FundingSeeder | ✅ | seed_funding.go | — |
| DosageFormSeeder | ✅ | seed_dosage_form.go | — |
| FreqUnitSeeder | ✅ | seed_freq_unit.go | — |
| MedicalCompositionUnitSeeder | ❌ MISSING | — | MEDIUM |
| CompositionUnitSeeder | ❌ MISSING | — | MEDIUM |
| MedicalNetUnitSeeder | ❌ MISSING | — | MEDIUM |
| MixUnitSeeder | ❌ MISSING | — | LOW |
| TherapeuticClassSeeder | ❌ MISSING | — | CRITICAL |
| PackageFormSeeder | ✅ | seed_seed_package.go | — |
| TrademarkSeeder | ✅ | seed_trademark.go | — |
| UsageLocationSeeder | ✅ | seed_usage_location.go | — |
| UsageRouteSeeder | ✅ | seed_usage_route.go | — |
| BrandSeeder | ✅ | seed_brand.go | — |
| SoapSeeder | ❌ MISSING | — | LOW |
| RoomItemCategorySeeder | ❌ MISSING | — | LOW |
| ClassRoomSeeder | ❌ MISSING | — | LOW |
| PurchaseLabelSeeder | ❌ MISSING | — | LOW |
| SampleSeeder | ❌ MISSING | — | HIGH |
| ClinicalPathologySeeder | ❌ MISSING | — | CRITICAL |
| RadiologySeeder | ❌ MISSING | — | CRITICAL |
| ProgramCategorySeeder | ❌ MISSING | — | LOW |
| ProgramOccupationSeeder | ❌ MISSING | — | LOW |

#### Local to wellmed-backbone
| Laravel Seeder | Go Status | Go File | Priority |
|----------------|-----------|---------|----------|
| MedicServiceSeeder | ✅ | seed_medic_service.go | — |
| FormSeeder | ✅ | seed_form.go | — |
| InfrastructureSeeder | ❌ MISSING | — | LOW |
| DocumentTypeSeeder | ❌ MISSING | — | HIGH |

### 2.4 Missing Laravel Seeders (Priority Summary)

| Priority | Seeder | Data Type | Volume | Complexity |
|----------|--------|-----------|--------|------------|
| CRITICAL | ClinicalPathologySeeder | 3-4 level hierarchy | ~180 items | High |
| CRITICAL | RadiologySeeder | 2-3 level hierarchy | ~10 items | Medium |
| CRITICAL | TherapeuticClassSeeder | Deep hierarchy | ~250+ items | High |
| CRITICAL | IcdSeeder | Bulk SQL | ~13K rows | Very High |
| HIGH | RegionalSeeder | Geographic (Country→Village) | ~90K items | High |
| HIGH | SampleSeeder | Complex Unicode | ~9 items | Medium |
| HIGH | DocumentTypeSeeder | Form templates | Dynamic | High |
| MEDIUM | CompositionUnitSeeder | Parent-child | ~37 items | Medium |
| MEDIUM | MedicalCompositionUnitSeeder | Simple list | ~8 items | Low |
| MEDIUM | MedicalNetUnitSeeder | Simple list | ~10 items | Low |
| MEDIUM | MixUnitSeeder | Simple list | ~5 items | Low |
| LOW | CoaSeeder | Chart of Accounts | ~20 items | Medium |
| LOW | WalletSeeder | Wallet types | ~5 items | Low |
| LOW | SoapSeeder | SOAP note templates | ~4 items | Low |
| LOW | RoomItemCategorySeeder | Room categories | ~10 items | Low |
| LOW | ClassRoomSeeder | Training rooms | ~5 items | Low |
| LOW | PurchaseLabelSeeder | Purchase labels | ~10 items | Low |
| LOW | ProgramCategorySeeder | Program types | ~10 items | Low |
| LOW | ProgramOccupationSeeder | Program-occupation map | ~20 items | Low |
| LOW | InfrastructureSeeder | Building/facility | ~5 items | Low |

---

## 3. Architecture & Patterns

### 3.1 Existing Go Pattern

```go
// Simple reference data (Unicode-based)
func SeedDosageForm(db *gorm.DB) {
    data := []entity.Unicode{
        {Id: helper.GenerateUlid(), Name: "Tablet", Label: "Tablet", Flag: "DosageForm"},
        // ...
    }
    helper.UpdateOrCreate(db, &data, []string{"flag", "name"}, []string{"label", "updated_at"})
}

// Hierarchical (2-level)
func SeedProfession(db *gorm.DB) {
    // Step 1: Insert parents
    parents := []entity.Unicode{...}
    helper.UpdateOrCreate(db, &parents, ...)

    // Step 2: Fetch parents back, map name→ID
    var parentRecords []entity.Unicode
    db.Where("flag = ? AND parent_id IS NULL", "Profession").Find(&parentRecords)
    parentMap := make(map[string]string)
    for _, p := range parentRecords {
        parentMap[p.Name] = p.Id
    }

    // Step 3: Insert children with ParentID
    children := []entity.Unicode{
        {Id: helper.GenerateUlid(), Name: "Dokter Umum", ParentID: &parentMap["Dokter"], ...},
    }
    helper.UpdateOrCreate(db, &children, ...)
}
```

### 3.2 Proposed Patterns for Complex Seeders

#### Pattern A: Deep Hierarchy (3+ levels)

```go
// For ClinicalPathology, TherapeuticClass
func seedDeepHierarchy(db *gorm.DB, flag string, tree []HierarchyNode) {
    // Recursive function
    var insertLevel func(nodes []HierarchyNode, parentID *string, level int)
    insertLevel = func(nodes []HierarchyNode, parentID *string, level int) {
        for _, node := range nodes {
            item := entity.Unicode{
                Id:       helper.GenerateUlid(),
                Name:     node.Name,
                Label:    node.Label,
                Flag:     flag,
                ParentID: parentID,
                Ordering: node.Ordering,
            }
            helper.UpdateOrCreate(db, &[]entity.Unicode{item}, ...)

            // Fetch back to get ID
            var inserted entity.Unicode
            db.Where("flag = ? AND name = ?", flag, node.Name).First(&inserted)

            if len(node.Children) > 0 {
                insertLevel(node.Children, &inserted.Id, level+1)
            }
        }
    }
    insertLevel(tree, nil, 0)
}
```

#### Pattern B: Bulk SQL Import

```go
// For ICD codes
func SeedICD(db *gorm.DB) {
    // Option 1: SQL file
    sqlContent, _ := os.ReadFile("internal/db/seed/data/icd10.sql")
    db.Exec(string(sqlContent))

    // Option 2: CSV streaming
    file, _ := os.Open("internal/db/seed/data/icd10.csv")
    reader := csv.NewReader(file)
    batch := make([]entity.Disease, 0, 1000)
    for {
        record, err := reader.Read()
        if err == io.EOF { break }
        batch = append(batch, entity.Disease{...})
        if len(batch) >= 1000 {
            db.CreateInBatches(batch, 100)
            batch = batch[:0]
        }
    }
}
```

#### Pattern C: Service with Complex Props

```go
// For DocumentType (Informed Consent)
func SeedDocumentType(db *gorm.DB) {
    type FormDefinition struct {
        Fields   []FormField `json:"fields"`
        Schema   string      `json:"schema"`
        Template string      `json:"template"`
    }

    docs := []struct {
        Name  string
        Label string
        Form  FormDefinition
    }{
        {
            Name:  "Surat Sakit",
            Label: "SICK_LETTER",
            Form: FormDefinition{
                Fields: []FormField{
                    {Key: "diagnosis", Type: "text", Required: true},
                    {Key: "days", Type: "number", Required: true},
                },
            },
        },
    }

    for _, doc := range docs {
        propsJson, _ := json.Marshal(doc.Form)
        item := entity.MasterInformedConsent{
            Id:    helper.GenerateUlid(),
            Name:  doc.Name,
            Label: doc.Label,
            Props: datatypes.JSON(propsJson),
        }
        helper.UpdateOrCreate(db, &[]entity.MasterInformedConsent{item}, ...)
    }
}
```

---

## 4. Handler Integration (SAAS Tenant Provisioning)

### 4.1 Current State

**Sudah Ada:**
- `TenantService.InstallTenant()` di `internal/domain/tenant/service/tenant.go`
- Sudah memanggil `seed.SeedLite()` atau `seed.SeedPlus()` berdasarkan product type
- Multi-tenant DB creation dan schema migration

**Belum Ada:**
- gRPC endpoint untuk trigger InstallTenant via REST/gRPC
- Background job consumer untuk async tenant provisioning
- Mode `SeedNewTenant()` (tanpa ICD, tanpa sample patients)

### 4.2 Target Architecture (Mirror Laravel)

```
┌─────────────────┐                    ┌─────────────────────────────────┐
│   HQ App        │                    │  wellmed-backbone (Go)          │
│   (Laravel)     │                    │                                 │
├─────────────────┤                    │  ┌───────────────────────────┐  │
│                 │  gRPC/REST         │  │  TenantProvisionService   │  │
│ Payment         │ ─────────────────► │  │  ─────────────────────    │  │
│ Confirmed       │  ProvisionTenant() │  │  • Receive request        │  │
│                 │                    │  │  • Publish to RabbitMQ    │  │
└─────────────────┘                    │  └───────────────────────────┘  │
                                       │              │                  │
                                       │              ▼                  │
                                       │  ┌───────────────────────────┐  │
                                       │  │  RabbitMQ                 │  │
                                       │  │  tenant.provision queue   │  │
                                       │  └───────────────────────────┘  │
                                       │              │                  │
                                       │              ▼                  │
                                       │  ┌───────────────────────────┐  │
                                       │  │  TenantProvisionConsumer  │  │
                                       │  │  ─────────────────────    │  │
                                       │  │  • Create DB              │  │
                                       │  │  • Migrate schemas        │  │
                                       │  │  • SeedNewTenant()        │  │
                                       │  │  • Callback to HQ         │  │
                                       │  └───────────────────────────┘  │
                                       └─────────────────────────────────┘
```

### 4.3 Required Changes

#### A. New Proto Definition (`proto/tenant_provision.proto`)

```protobuf
syntax = "proto3";
package tenant;
option go_package = "tenantpb/";

service TenantProvisionService {
  // Sync: langsung provision (blocking)
  rpc ProvisionTenant(ProvisionTenantRequest) returns (ProvisionTenantResponse);

  // Async: trigger background job, return immediately
  rpc ProvisionTenantAsync(ProvisionTenantRequest) returns (ProvisionTenantAsyncResponse);

  // Status check
  rpc GetProvisionStatus(GetProvisionStatusRequest) returns (ProvisionStatus);
}

message ProvisionTenantRequest {
  string tenant_uuid = 1;
  string product_type = 2;  // "lite", "plus", "enterprise"
  bool include_sample_data = 3;  // ICD, sample patients
}

message ProvisionTenantResponse {
  bool success = 1;
  string message = 2;
  string db_name = 3;
}

message ProvisionTenantAsyncResponse {
  string job_id = 1;
  string status = 2;  // "queued", "processing", "completed", "failed"
}
```

#### B. New Seeder Mode

```go
// internal/db/seed/seeder.go

// SeedNewTenant — untuk SAAS tenant baru (tanpa ICD, tanpa sample patients)
// Equivalent to Laravel AddDatabaseSeeder
func SeedNewTenant(db *gorm.DB, productType string) {
    log.Println("[Seeder] Run mode NEW_TENANT...")

    // Core reference data (sama untuk semua product type)
    SeedMaster(db)

    // Product-specific
    switch productType {
    case "lite":
        SeedLiteRestriction(db)
    case "plus":
        SeedPlusRestriction(db)
    case "enterprise":
        SeedEnterpriseRestriction(db)
    }

    // NOTE: TIDAK memanggil SeedICD() dan SeedPatient()
    log.Println("[Seeder] Done mode NEW_TENANT ✅")
}

// SeedFullInstall — untuk test/dev (include ICD, sample patients)
// Equivalent to Laravel DatabaseSeeder
func SeedFullInstall(db *gorm.DB, productType string) {
    log.Println("[Seeder] Run mode FULL_INSTALL...")

    SeedNewTenant(db, productType)
    SeedICD(db)           // ~13K diseases
    SeedPatient(db, 1000) // 1000 sample patients

    log.Println("[Seeder] Done mode FULL_INSTALL ✅")
}
```

#### C. Update TenantService

```go
// internal/domain/tenant/service/tenant.go

func (s *tenantService) InstallTenant(ctx context.Context, req *dto.TenantRequest) error {
    // ... existing code ...

    // UPDATED: Use SeedNewTenant for SAAS, SeedFullInstall for test
    if req.IncludeSampleData {
        seed.SeedFullInstall(dbTenant, tenant.ProductType)
    } else {
        seed.SeedNewTenant(dbTenant, tenant.ProductType)
    }

    return nil
}
```

#### D. Background Job Consumer (Optional - leverage existing RabbitMQ)

```go
// internal/messaging/tenant_provision_consumer.go

func (c *TenantProvisionConsumer) ConsumeProvisionRequests(ctx context.Context) {
    // Listen to "tenant.provision" queue
    // On message:
    //   1. Parse ProvisionTenantRequest
    //   2. Call TenantService.InstallTenant()
    //   3. Publish result to callback queue or REST API
}
```

### 4.4 Comparison: Laravel vs Go (Target)

| Feature | Laravel | Go (Current) | Go (Target) |
|---------|---------|--------------|-------------|
| Sync Install | `artisan wellmed-backbone:install` | ❌ | CLI command |
| Async Provision | REST API → Queue → AddDatabaseSeeder | ❌ | gRPC → RabbitMQ → Consumer |
| Mode: Full | DatabaseSeeder | ❌ | `SeedFullInstall()` |
| Mode: Lite | LiteDatabaseSeeder | `SeedLite()` | `SeedLiteInstall()` |
| Mode: New Tenant | AddDatabaseSeeder | ❌ | `SeedNewTenant()` |
| ICD in New Tenant | ❌ | N/A | ❌ |
| Sample Patients | Only in Full/Lite | N/A | Only in Full |
| Background Job | Laravel Queue | RabbitMQ (saga) | RabbitMQ (tenant queue) |
| Status Callback | REST API | ❌ | gRPC callback / webhook |

---

## 5. Implementation Phases

### Phase 1: Geographic & Simple Data (Week 1)

**Objective:** Seed geographic master data dan simple reference

| Task | Entity | Source | Est. Records |
|------|--------|--------|--------------|
| 5.1.1 | Country | countries.sql | ~200 |
| 5.1.2 | Province | provinces.sql | ~34 |
| 5.1.3 | District | districts.sql | ~500 |
| 5.1.4 | SubDistrict | subdistricts.sql | ~7K |
| 5.1.5 | Village | villages.sql | ~82K |
| 5.1.6 | Sample | Manual | ~9 |
| 5.1.7 | MedicalNetUnit | Manual | ~10 |
| 5.1.8 | MedicalCompositionUnit | Manual | ~8 |

**Deliverables:**
- `seed_country.go`
- `seed_province.go`
- `seed_district.go`
- `seed_sub_district.go`
- `seed_village.go`
- `seed_sample.go`
- `seed_medical_net_unit.go`
- `seed_medical_composition_unit.go`

### Phase 2: Medical Service Hierarchy (Week 2)

**Objective:** Implement 3-4 level hierarchy seeders

| Task | Entity | Pattern | Est. Records |
|------|--------|---------|--------------|
| 4.2.1 | ClinicalPathology | Deep Hierarchy | ~180 |
| 4.2.2 | Radiology | Deep Hierarchy | ~10 |
| 4.2.3 | CompositionUnit | Parent-child | ~37 |

**Deliverables:**
- `seed_clinical_pathology.go`
- `seed_radiology.go`
- `seed_composition_unit.go`
- Helper: `seed_hierarchy_helper.go`

### Phase 3: Classification Data (Week 3)

**Objective:** Bulk import medical classification data

| Task | Entity | Pattern | Est. Records |
|------|--------|---------|--------------|
| 4.3.1 | TherapeuticClass | Deep Hierarchy + JSON | ~250+ |
| 4.3.2 | ICD-10 | Bulk SQL/CSV | ~13K+ |

**Deliverables:**
- `seed_therapeutic_class.go`
- `seed_icd.go`
- Data files: `data/therapeutic_class.json`, `data/icd10.sql`

### Phase 4: Document Templates & Admin (Week 4)

**Objective:** Seed document templates dan admin data

| Task | Entity | Pattern | Est. Records |
|------|--------|---------|--------------|
| 4.4.1 | DocumentType | Complex Props | ~10 |
| 4.4.2 | ReportCollection | Complex Props | ~6 |
| 4.4.3 | ServiceCluster | Parent-child | Variable |

**Deliverables:**
- `seed_document_type.go`
- `seed_report_collection.go`
- `seed_service_cluster.go`

### Phase 5: Orchestrator Update (Week 5)

**Objective:** Update seeder orchestrator untuk support semua modes

| Task | Description |
|------|-------------|
| 4.5.1 | Implement `SeedPlus()` dengan semua seeders |
| 4.5.2 | Implement `SeedEMR()` untuk EMR-specific data |
| 4.5.3 | Add CLI flags untuk selective seeding |
| 4.5.4 | Add progress logging |

**Deliverables:**
- Updated `seeder.go` dengan:
  - `SeedPlus()` implementation
  - `SeedEMR()` implementation
  - `SeedGeographic()` for geographic data
  - `SeedClassification()` for ICD/Therapeutic

---

## 5. Critical Files

### 5.1 Existing Go Seeders (Reference)

| File | Purpose |
|------|---------|
| `internal/db/seed/seeder.go` | Orchestrator |
| `internal/db/seed/seed_profession.go` | 2-level hierarchy example |
| `internal/db/seed/seed_anatomy.go` | Complex hierarchy example |
| `internal/db/seed/seed_form.go` | Props/JSON example |

### 5.2 Laravel Seeders (Source)

| File | Purpose |
|------|---------|
| `src/Database/Seeders/ClinicalPathologySeeder.php` | 3-level hierarchy source |
| `src/Database/Seeders/TherapeuticClassSeeder.php` | Deep hierarchy source |
| `src/Database/Seeders/IcdSeeder.php` | Bulk SQL source |
| `src/Database/Seeders/data/diseases.sql` | ICD data file |
| `src/Database/Seeders/SampleSeeder.php` | Sample types source |

### 5.3 Entity Definitions

| File | Entity |
|------|--------|
| `internal/schema/entity/unicode.go` | Unicode (reference data) |
| `internal/schema/entity/disease.go` | Disease (ICD) |
| `internal/schema/entity/service.go` | Service |
| `internal/schema/entity/master_vaccine.go` | MasterVaccine |
| `internal/schema/entity/examination_stuff.go` | ExaminationStuff |

---

## 6. Entity-Seeder Mapping (Laravel → Go)

### 6.1 Covered Entities

| Laravel Seeder | Go Seeder | Entity | Status |
|----------------|-----------|--------|--------|
| DosageFormSeeder | seed_dosage_form.go | Unicode (Flag: DosageForm) | ✅ |
| ProfessionSeeder | seed_profession.go | Unicode (Flag: Profession) | ✅ |
| BrandSeeder | seed_brand.go | Unicode (Flag: Brand) | ✅ |
| FundingSeeder | seed_funding.go | Unicode (Flag: Funding) | ✅ |
| OccupationSeeder | seed_occupation.go | Unicode (Flag: Occupation) | ✅ |
| FreqUnitSeeder | seed_freq_unit.go | Unicode (Flag: FreqUnit) | ✅ |
| EducationSeeder | seed_education.go | Unicode (Flag: Education) | ✅ |
| PaymentMethodSeeder | seed_payment_method.go | Service | ✅ |
| MedicServiceSeeder | seed_medic_service.go | Service | ✅ |
| UsageLocationSeeder | seed_usage_location.go | Unicode (Flag: UsageLocation) | ✅ |
| UsageRouteSeeder | seed_usage_route.go | Unicode (Flag: UsageRoute) | ✅ |
| ItemStuffSeeder | seed_item_stuff.go | ItemStuff | ✅ |
| FormSeeder | seed_form.go | DynamicForm | ✅ |
| TrademarkSeeder | seed_trademark.go | Unicode (Flag: Trademark) | ✅ |
| PatientTypeServiceSeeder | seed_patient_type_service.go | PatientTypeService | ✅ |
| PatientOccupationSeeder | seed_patient_occupation.go | PatientOccupation | ✅ |

### 6.2 Missing Entities (To Implement)

| Laravel Seeder | Target Go Seeder | Entity | Priority |
|----------------|------------------|--------|----------|
| ClinicalPathologySeeder | seed_clinical_pathology.go | Unicode/Service | CRITICAL |
| RadiologySeeder | seed_radiology.go | Unicode/Service | CRITICAL |
| TherapeuticClassSeeder | seed_therapeutic_class.go | TherapeuticClass | CRITICAL |
| IcdSeeder | seed_icd.go | Disease | CRITICAL |
| SampleSeeder | seed_sample.go | Unicode (Flag: Sample) | HIGH |
| CountrySeeder | seed_country.go | Country | HIGH |
| ProvinceSeeder | seed_province.go | Province | HIGH |
| DocumentTypeSeeder | seed_document_type.go | MasterInformedConsent | HIGH |
| CompositionUnitSeeder | seed_composition_unit.go | Unicode (Flag: CompositionUnit) | MEDIUM |
| MedicalCompositionUnitSeeder | seed_medical_composition_unit.go | Unicode | MEDIUM |
| MedicalNetUnitSeeder | seed_medical_net_unit.go | Unicode | MEDIUM |
| MasterReportCollectionSeeder | seed_report_collection.go | ReportCollection | MEDIUM |

---

## 7. Tasks / Checklist

### Phase 1: Geographic & Simple
- [ ] Create seed_country.go
- [ ] Create seed_province.go
- [ ] Create seed_district.go
- [ ] Create seed_sub_district.go
- [ ] Create seed_village.go
- [ ] Create seed_sample.go
- [ ] Create seed_medical_net_unit.go
- [ ] Create seed_medical_composition_unit.go
- [ ] Test Phase 1 seeders

### Phase 2: Medical Service Hierarchy
- [ ] Create seed_hierarchy_helper.go (recursive seeder)
- [ ] Create seed_clinical_pathology.go
- [ ] Create seed_radiology.go
- [ ] Create seed_composition_unit.go
- [ ] Test Phase 2 seeders

### Phase 3: Classification Data
- [ ] Copy diseases.sql to data/ folder
- [ ] Create seed_icd.go (bulk SQL)
- [ ] Create therapeutic_class.json data file
- [ ] Create seed_therapeutic_class.go
- [ ] Test Phase 3 seeders

### Phase 4: Document Templates
- [ ] Create seed_document_type.go
- [ ] Create seed_report_collection.go
- [ ] Create seed_service_cluster.go
- [ ] Test Phase 4 seeders

### Phase 5: Orchestrator
- [ ] Update seeder.go dengan SeedPlus()
- [ ] Update seeder.go dengan SeedEMR()
- [ ] Add SeedGeographic() function
- [ ] Add SeedClassification() function
- [ ] Add CLI flags untuk selective seeding
- [ ] Add progress logging
- [ ] Full integration test

---

## 8. Validation Criteria

### 8.1 Per-Seeder Validation

| Check | Description |
|-------|-------------|
| Idempotent | Running twice produces same result |
| Upsert | Uses UpdateOrCreate pattern |
| FK Valid | Foreign keys reference valid parent records |
| Props Valid | JSON Props field is valid JSON |
| Hierarchy Valid | Parent-child relationships are correct |

### 8.2 Cross-Seeder Validation

| Check | Description |
|-------|-------------|
| No Duplicates | Same record not seeded by multiple seeders |
| Dependency Order | Parents seeded before children |
| Multi-tenant | Works in tenant database context |

### 8.3 Performance Targets

| Seeder | Target Time |
|--------|-------------|
| SeedLite() | < 30 seconds |
| SeedPlus() | < 5 minutes |
| SeedGeographic() | < 2 minutes |
| SeedClassification() | < 3 minutes |

---

## 9. References

- ADR-005: Saga Orchestration (multi-tenant context)
- ADR-006: Domain Service Boundaries (entity ownership)
- Laravel Seeders: `/var/www/projects/kalpa/wellmed/projects/wellmed-backbone/src/Database/Seeders/`
- Go Seeders: `/var/www/projects/kalpa/wellmed-backbone/internal/db/seed/`
- Entity Definitions: `/var/www/projects/kalpa/wellmed-backbone/internal/schema/entity/`

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-13 | Claude | Initial plan with 5 phases, entity mapping, and validation criteria |
