# Plan 4: Go Migration Infrastructure

**Status**: Proposed
**Author**: Claude
**Created**: 2026-03-13
**Related Plans**: Plan 3 (Go Seeder Migration)

---

## Executive Summary

Migrasi infrastructure Laravel migrations ke Go untuk wellmed-backbone. Plan ini mencakup:
1. Identifikasi entities yang belum ada di Go
2. Pembuatan missing entities
3. Implementasi migration infrastructure (AutoMigrate + custom SQL)

---

## Current State Analysis

### Laravel Migrations Count
| Location | Count | Description |
|----------|-------|-------------|
| `wellmed/database/migrations` | 22 | Core/central migrations |
| `wellmed-backbone/src/Database/Migrations` | 27 | Central backbone migrations |
| `wellmed-backbone/src/Database/Migrations/tenants` | ~100 | Tenant-level migrations |
| `wellmed-backbone/src/Database/Migrations/tenants/clusters/emr` | ~15 | EMR cluster migrations |
| `wellmed-backbone/src/Database/Migrations/tenants/clusters/pos` | ~12 | POS cluster migrations |
| `wellmed-backbone/src/Database/Migrations/tenants/clusters/scm` | ~10 | SCM cluster migrations |
| **Total** | **~186** | |

### Go Entities Count (Verified)
| Service | Count | Location |
|---------|-------|----------|
| wellmed-backbone | 56 | `internal/schema/entity/*.go` |
| wellmed-backbone/emr | 17 | `internal/schema/entity/emr/*.go` |
| wellmed-backbone/pos | 14 | `internal/schema/entity/pos/*.go` |
| wellmed-cashier | ~14 | `internal/schema/entity/*.go` (duplicates) |
| **Total Unique** | **~87** | |

### Gap Summary
- **Laravel tables**: ~120 unique tables
- **Go entities**: ~87 entities
- **Missing entities**: ~33 entities need to be created (reduced from initial estimate)

---

## Entity Gap Analysis

### PHASE 1: Core Infrastructure (Already Exists in Go)
| Entity | Go File | Status |
|--------|---------|--------|
| User | `user.go` | ✅ EXISTS |
| Tenant | `tenant.go` | ✅ EXISTS |
| Domain | `domain.go` | ✅ EXISTS |
| Role | `role.go` | ✅ EXISTS |
| Permission | `permission.go` | ✅ EXISTS |
| ModelHasRole | `model_has_role.go` | ✅ EXISTS |
| ModelHasPermission | `model_has_permission.go` | ✅ EXISTS |

### PHASE 2: Geographic Data (Already Exists in Go)
| Entity | Go File | Status |
|--------|---------|--------|
| Country | `country.go` | ✅ EXISTS |
| Province | `province.go` | ✅ EXISTS |
| District | `district.go` | ✅ EXISTS |
| SubDistrict | `sub_district.go` | ✅ EXISTS |
| Village | `village.go` | ✅ EXISTS |

### PHASE 3: Patient & People (Mostly Exists)
| Entity | Go File | Status |
|--------|---------|--------|
| Patient | `patient.go` | ✅ EXISTS |
| People | `people.go` | ✅ EXISTS |
| CardIdentity | `card_identity.go` | ✅ EXISTS |
| Address | `address.go` | ✅ EXISTS |
| FamilyRelationship | `family_relationship.go` | ✅ EXISTS |
| Employee | `employee.go` | ✅ EXISTS |

### PHASE 4: Items & Inventory (Mostly Exists)
| Entity | Go File | Status |
|--------|---------|--------|
| Item | `item.go` | ✅ EXISTS |
| Medicine | `medicine.go` | ✅ EXISTS |
| Variant | `variant.go` | ✅ EXISTS |
| ItemHasVariant | `item_has_variant.go` | ✅ EXISTS |
| Batch | `batch.go` | ✅ EXISTS |
| Stock | `stock.go` | ✅ EXISTS |
| StockBatch | `stock_batch.go` | ✅ EXISTS |
| CardStock | `card_stock.go` | ✅ EXISTS |
| OpnameStock | `opname_stock.go` | ✅ EXISTS |
| WarehouseItem | `warehouse_item.go` | ✅ EXISTS |
| MedicalItem | `medical_item.go` | ✅ EXISTS |
| ServiceItem | `service_item.go` | ✅ EXISTS |
| ItemRent | `item_rent.go` | ✅ EXISTS |

### PHASE 5: Services & Room (Mostly Exists)
| Entity | Go File | Status |
|--------|---------|--------|
| Service | `service.go` | ✅ EXISTS |
| ServicePrice | `service_price.go` | ✅ EXISTS |
| Room | `room.go` | ✅ EXISTS |
| Shift | `shift.go` | ✅ EXISTS |

### PHASE 6: EMR Cluster (Mostly Exists)
| Entity | Go File | Status |
|--------|---------|--------|
| VisitPatient | `emr/visit_patient.go` | ✅ EXISTS |
| VisitRegistration | `emr/visit_registration.go` | ✅ EXISTS |
| VisitExamination | `emr/visit_examination.go` | ✅ EXISTS |
| Assessment | `emr/assessment.go` | ✅ EXISTS |
| Diagnose | `emr/diagnose.go` | ✅ EXISTS |
| Prescription | `emr/prescription.go` | ✅ EXISTS |
| ExaminationSummary | `emr/examination_summary.go` | ✅ EXISTS |
| ExaminationTreatment | `emr/examination_treatment.go` | ✅ EXISTS |
| Summary | `emr/summary.go` | ✅ EXISTS |
| Referral | `emr/referral.go` | ✅ EXISTS |
| PatientIllness | `emr/patient_illness.go` | ✅ EXISTS |
| PatientTypeHistory | `emr/patient_type_history.go` | ✅ EXISTS |
| InformedConsent | `emr/informed_concern.go` | ✅ EXISTS |
| PractitionerEvaluation | `emr/practitioner_evaluation.go` | ✅ EXISTS |
| Support | `emr/support.go` | ✅ EXISTS |

### PHASE 7: POS Cluster (Mostly Exists)
| Entity | Go File | Status |
|--------|---------|--------|
| Transaction | `pos/transaction.go` | ✅ EXISTS |
| TransactionItem | `pos/transaction_item.go` | ✅ EXISTS |
| TransactionHasConsument | `pos/transaction_has_consument.go` | ✅ EXISTS |
| Billing | `pos/billing.go` | ✅ EXISTS |
| Invoice | `pos/invoice.go` | ✅ EXISTS |
| PaymentSummary | `pos/payment_summary.go` | ✅ EXISTS |
| PaymentDetail | `pos/payment_detail.go` | ✅ EXISTS |
| SplitPayment | `pos/split_payment.go` | ✅ EXISTS |
| Refund | `pos/refund.go` | ✅ EXISTS |
| WalletTransaction | `pos/wallet_transaction.go` | ✅ EXISTS |
| Activity | `pos/activity.go` | ✅ EXISTS |
| ActivityStatus | `pos/activity_status.go` | ✅ EXISTS |

---

## Missing Entities (Need to Create)

### PRIORITY 1: Core/Polymorphic Tables
| Entity | Laravel Table | Connection | Status |
|--------|---------------|------------|--------|
| ModelHasPhone | `model_has_phones` | tenant | ✅ EXISTS |
| ModelHasService | `model_has_services` | tenant | ✅ EXISTS |
| ModelHasStorage | `model_has_storages` | tenant | ✅ EXISTS |
| ModelHasWarehouse | `model_has_warehouses` | tenant | ✅ EXISTS |
| RoleHasPermission | `role_has_permissions` | tenant | ✅ EXISTS |
| ModelHasEncoding | `model_has_encodings` | tenant | ❌ MISSING |
| ModelHasRelation | `model_has_relations` | central | ❌ MISSING |
| ModelHasOrganization | `model_has_organization` | tenant | ❌ MISSING |
| ModelHasMonitoring | `model_has_monitorings` | tenant | ❌ MISSING |
| ModelHasVoucher | `model_has_voucher` | tenant | ❌ MISSING |

### PRIORITY 2: Workforce Management
| Entity | Laravel Table | Connection | Priority |
|--------|---------------|------------|----------|
| Attendance | `attendences` | tenant | HIGH |
| AbsenceRequest | `absence_requests` | tenant | MEDIUM |
| Worker | `workers` | tenant | MEDIUM |
| Program | `programs` | tenant | LOW |
| EmployeeHasType | `employee_has_types` | tenant | MEDIUM |
| EmployeeShift | `employee_shifts` | tenant | MEDIUM |
| ShiftHasSchedule | `shift_has_schedules` | tenant | LOW |

### PRIORITY 3: Inventory & SCM
| Entity | Laravel Table | Connection | Priority |
|--------|---------------|------------|----------|
| Inventory | `inventories` | tenant | HIGH |
| InventoryAsset | `inventory_assets` | tenant | MEDIUM |
| InventoryItem | `inventory_items` | tenant | MEDIUM |
| MedicTool | `medic_tools` | tenant | MEDIUM |
| Composition | `compositions` | tenant | MEDIUM |
| ModelHasComposition | `model_has_composition` | tenant | LOW |
| Reagent | `reagents` | tenant | LOW |

### PRIORITY 4: SCM Cluster (New Schema)
| Entity | Laravel Table | Connection | Priority |
|--------|---------------|------------|----------|
| Procurement | `procurements` | scm | HIGH |
| ProcurementList | `procurement_lists` | scm | HIGH |
| ProcurementService | `procurement_services` | scm | MEDIUM |
| PurchaseRequest | `purchase_requests` | scm | HIGH |
| PurchaseOrder | `purchase_orders` | scm | HIGH |
| ReceiveOrder | `receive_orders` | scm | HIGH |
| Distribution | `distributions` | scm | MEDIUM |
| Purchasing | `purchasings` | scm | LOW |

### PRIORITY 5: Financial & Accounting
| Entity | Laravel Table | Connection | Priority |
|--------|---------------|------------|----------|
| Coa | `coas` | tenant | MEDIUM |
| CoaEntry | `coa_entries` | tenant | MEDIUM |
| JournalEntry | `journal_entries` | tenant | MEDIUM |
| JournalBatch | `journal_batches` | tenant | LOW |
| TariffComponent | `tariff_components` | tenant | MEDIUM |
| ComponentDetail | `component_details` | tenant | LOW |

### PRIORITY 6: Appointments & Scheduling
| Entity | Laravel Table | Connection | Priority |
|--------|---------------|------------|----------|
| Appointment | `appointments` | tenant | HIGH |
| UnidentifiedPatient | `unidentified_patients` | tenant | MEDIUM |

### PRIORITY 7: Voucher & Loyalty
| Entity | Laravel Table | Connection | Priority |
|--------|---------------|------------|----------|
| Voucher | `voucher` | tenant | LOW |
| VoucherRule | `voucher_rules` | tenant | LOW |

### PRIORITY 8: Material Management
| Entity | Laravel Table | Connection | Priority |
|--------|---------------|------------|----------|
| MaterialCategory | `material_categories` | tenant | LOW |
| Material | `materials` | tenant | LOW |
| BillOfMaterial | `bill_of_materials` | tenant | LOW |
| Product | `products` | tenant | LOW |
| ProductBatch | `product_batches` | tenant | LOW |

### PRIORITY 9: System & Logging
| Entity | Laravel Table | Connection | Priority |
|--------|---------------|------------|----------|
| ApiAccess | `api_accesses` | central | HIGH |
| LogHistory | `log_histories` | central | MEDIUM |
| PayloadMonitoring | `payload_monitorings` | central | LOW |
| Workspace | `workspaces` | central | MEDIUM |
| Monitoring | `monitorings` | tenant | LOW |
| DocumentReference | `document_references` | tenant | LOW |
| Notification | `notifications` | tenant | MEDIUM |
| Export | `exports` | tenant | LOW |
| QueueTransaction | `queue_transactions` | tenant | LOW |
| Event | `events` | tenant | LOW |
| ClassRoom | `class_rooms` | tenant | LOW |
| FormHasSurvey | `form_has_surveys` | tenant | LOW |
| ScreeningHasModel | `screening_has_model` | tenant | LOW |
| ConfigProp | `config_props` | tenant | LOW |
| SatuSehatLog | `satu_sehat_logs` | tenant | LOW |
| ElasticsearchLog | `elasticsearch_logs` | tenant | LOW |

---

## Implementation Strategy

### Phase A: Entity Creation (Week 1-2)

#### A.1: Core Polymorphic Entities
Most core polymorphic entities already exist. Create only missing ones:
```
wellmed-backbone/internal/schema/entity/
├── model_has_phone.go        # ✅ EXISTS
├── model_has_service.go      # ✅ EXISTS
├── model_has_storage.go      # ✅ EXISTS
├── model_has_warehouse.go    # ✅ EXISTS
├── role_has_permission.go    # ✅ EXISTS
├── model_has_encoding.go     # ❌ NEW - Need to create
├── model_has_relation.go     # ❌ NEW - Need to create
├── model_has_organization.go # ❌ NEW - Need to create
├── model_has_monitoring.go   # ❌ NEW - Need to create
```

#### A.2: Workforce Entities
```
wellmed-backbone/internal/schema/entity/
├── attendance.go             # NEW
├── absence_request.go        # NEW
├── worker.go                 # NEW
├── employee_has_type.go      # NEW
├── employee_shift.go         # NEW
```

#### A.3: SCM Cluster Entities
```
wellmed-backbone/internal/schema/entity/scm/   # NEW DIRECTORY
├── procurement.go            # NEW
├── procurement_list.go       # NEW
├── purchase_request.go       # NEW
├── purchase_order.go         # NEW
├── receive_order.go          # NEW
├── distribution.go           # NEW
├── card_stock.go             # MOVE from entity/ (if SCM-specific)
```

#### A.4: System Entities
```
wellmed-backbone/internal/schema/entity/
├── api_access.go             # NEW
├── log_history.go            # NEW
├── workspace.go              # NEW
├── appointment.go            # NEW
├── notification.go           # NEW
```

### Phase B: Migration Infrastructure (Week 2-3)

#### B.1: Update unified.go
Extend `internal/db/migration/unified.go`:
- Add all new entities to AutoMigrate
- Group by connection (central, tenant, emr, pos, scm)

#### B.2: Create SCM Migration
New file: `internal/db/migration/scm/migration.go`
```go
package scm

func Migrate(db *gorm.DB) error {
    return db.AutoMigrate(
        &scm.Procurement{},
        &scm.ProcurementList{},
        &scm.PurchaseRequest{},
        &scm.PurchaseOrder{},
        &scm.ReceiveOrder{},
        &scm.Distribution{},
        &scm.CardStock{},
    )
}
```

#### B.3: Create Index Migrations
New files for optimized indexes:
- `internal/db/migration/procurement_indexes.go`
- `internal/db/migration/appointment_indexes.go`

### Phase C: CLI & Makefile (Week 3)

#### C.1: Update cmd/main.go
Add new migration commands:
```
migrate scm              # Migrate SCM cluster only
migrate workforce        # Migrate workforce entities
migrate all --fresh      # Drop + migrate all
```

#### C.2: Update Makefile
```makefile
migrate:scm:
    @go run cmd/main.go migrate scm

migrate:workforce:
    @go run cmd/main.go migrate workforce
```

---

## Entity Creation Templates

### Template A: Simple Entity
```go
package entity

import (
    "time"
    "gorm.io/gorm"
)

type Attendance struct {
    Id            string         `gorm:"column:id;type:char(26);primaryKey"`
    EmployeeID    string         `gorm:"column:employee_id;type:char(26);index"`
    ShiftID       string         `gorm:"column:shift_id;type:char(26)"`
    CheckInAt     *time.Time     `gorm:"column:check_in_at"`
    CheckOutAt    *time.Time     `gorm:"column:check_out_at"`
    Status        string         `gorm:"column:status;type:varchar(50);default:'present'"`
    Props         datatypes.JSON `gorm:"column:props;type:jsonb"`
    CreatedAt     time.Time      `gorm:"column:created_at;autoCreateTime"`
    UpdatedAt     time.Time      `gorm:"column:updated_at;autoUpdateTime"`
    DeletedAt     gorm.DeletedAt `gorm:"column:deleted_at;index"`
}

func (Attendance) TableName() string {
    return "attendences"
}
```

### Template B: Polymorphic Entity
```go
package entity

type ModelHasPhone struct {
    ID         uint   `gorm:"column:id;primaryKey;autoIncrement"`
    ModelType  string `gorm:"column:model_type;type:varchar(255);index:idx_model,priority:1"`
    ModelID    string `gorm:"column:model_id;type:char(26);index:idx_model,priority:2"`
    Phone      string `gorm:"column:phone;type:varchar(20)"`
    Flag       string `gorm:"column:flag;type:varchar(50)"` // primary, secondary, emergency
    IsVerified bool   `gorm:"column:is_verified;default:false"`
}

func (ModelHasPhone) TableName() string {
    return "model_has_phones"
}
```

### Template C: SCM Entity
```go
package scm

import (
    "time"
    "gorm.io/datatypes"
    "gorm.io/gorm"
)

type Procurement struct {
    Id              string         `gorm:"column:id;type:char(26);primaryKey"`
    ProcurementCode string         `gorm:"column:procurement_code;type:varchar(50);uniqueIndex"`
    PurchaseLabelID *string        `gorm:"column:purchase_label_id;type:char(26)"`
    WarehouseType   string         `gorm:"column:warehouse_type;type:varchar(255);index:idx_warehouse,priority:1"`
    WarehouseID     string         `gorm:"column:warehouse_id;type:char(26);index:idx_warehouse,priority:2"`
    SenderType      string         `gorm:"column:sender_type;type:varchar(255)"`
    SenderID        string         `gorm:"column:sender_id;type:char(26)"`
    AuthorType      string         `gorm:"column:author_type;type:varchar(255)"`
    AuthorID        string         `gorm:"column:author_id;type:char(26)"`
    Status          string         `gorm:"column:status;type:varchar(50);default:'draft'"`
    ApprovedAt      *time.Time     `gorm:"column:approved_at"`
    Props           datatypes.JSON `gorm:"column:props;type:jsonb"`
    CreatedAt       time.Time      `gorm:"column:created_at;autoCreateTime"`
    UpdatedAt       time.Time      `gorm:"column:updated_at;autoUpdateTime"`
    DeletedAt       gorm.DeletedAt `gorm:"column:deleted_at;index"`
}

func (Procurement) TableName() string {
    return "procurements"
}
```

---

## Migration Execution Order

### Step 1: Create Missing Entities
```bash
# Verify existing entities
ls -la wellmed-backbone/internal/schema/entity/
ls -la wellmed-backbone/internal/schema/entity/emr/
ls -la wellmed-backbone/internal/schema/entity/pos/

# Create new entity files (per priority)
# Priority 1 first, then 2, etc.
```

### Step 2: Update Migration Files
```bash
# Update unified.go
# Create scm/migration.go
# Create index migrations
```

### Step 3: Build & Test
```bash
cd wellmed-backbone
go build ./...
make migrate
```

### Step 4: Verify Tables
```bash
psql -h localhost -p 5432 -U postgres -d wellmed -c "\dt"
psql -h localhost -p 5432 -U postgres -d clinic_4 -c "\dt"
```

---

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Entity field mismatch with Laravel | HIGH | Compare with Laravel models before creating |
| Missing foreign key constraints | MEDIUM | Add proper GORM tags and constraints |
| Schema drift between PHP and Go | HIGH | Document all differences |
| Index naming conflicts | LOW | Use consistent naming convention |
| Multi-tenant schema issues | HIGH | Test on multiple tenant databases |

---

## Deliverables

1. **~25-30 new entity files** in wellmed-backbone (reduced - many already exist)
2. **1 new migration directory** (`internal/db/migration/scm/`)
3. **Updated unified.go** with all entities
4. **Updated cmd/main.go** with new commands
5. **Updated Makefile** with new targets
6. **Progress tracking** in `4-go-migration-infrastructure-PROGRESS.md`

---

## Timeline Estimate

| Phase | Description | Duration |
|-------|-------------|----------|
| Phase A.1 | Core polymorphic entities | 2 days |
| Phase A.2 | Workforce entities | 1 day |
| Phase A.3 | SCM cluster entities | 2 days |
| Phase A.4 | System entities | 1 day |
| Phase B | Migration infrastructure | 2 days |
| Phase C | CLI & Makefile | 1 day |
| Testing | Integration testing | 2 days |
| **Total** | | **~11 days** |

---

## References

- Laravel migrations: `wellmed/projects/wellmed-backbone/src/Database/Migrations/`
- Entity-table mapping: `kalpa-docs/references/laravel-entity-table-mapping.md`
- Connection config: `wellmed/connection.txt`
- Existing Go entities: `wellmed-backbone/internal/schema/entity/`
