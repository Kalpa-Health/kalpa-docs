# Laravel Entity-Table Mapping Reference

**Source**: Generated from WellMed Laravel runtime config
**Date**: 2026-03-13
**Purpose**: Reference untuk mapping entity/model ke connection dan table

---

## How to Use This Mapping

### Finding Table & DB for an Entity

1. **Find entity name** dari Schema: `protected string $__entity = 'EntityName'`
2. **Look up entity** di mapping dibawah untuk find connection & table
3. **Check connection type**:
   - `central` → database `wellmed`, schema `public`
   - `tenant` → database `clinic_*`, schema `public`
   - `emr` → database `clinic_*`, schema `emr_2026`
   - `pos` → database `clinic_*`, schema `pos_2026`
   - `scm` → database `clinic_*`, schema `scm_2026`

### Example Workflow

**Question**: "ClinicalPathology masuk table mana?"

1. Find in entity mapping → `ClinicalPathology: "Hanafalah\\ModuleLabRadiology\\Models\\ClinicalPathology"`
2. Find in connection-table mapping → `{"connection": "tenant", "table": "public.examination_stuffs"}`
3. Answer: **tenant DB, table `examination_stuffs`**

---

## Entity to Model Mapping

Full list dari `wellmed/entity.txt`:

| Entity Name | Model Class Path |
|-------------|------------------|
| ClinicalPathology | Hanafalah\\ModuleLabRadiology\\Models\\ClinicalPathology |
| Radiology | Hanafalah\\ModuleLabRadiology\\Models\\Radiology |
| CompositionUnit | Hanafalah\\ModuleItem\\Models\\CompositionUnit |
| TherapeuticClass | Hanafalah\\ModuleMedicalItem\\Models\\TherapeuticClass |
| Sample | Hanafalah\\ModuleLabRadiology\\Models\\Sample |
| MedicalNetUnit | Hanafalah\\ModuleMedicalItem\\Models\\MedicalNetUnit |
| MedicalCompositionUnit | Hanafalah\\ModuleMedicalItem\\Models\\MedicalCompositionUnit |
| Country | Hanafalah\\ModuleRegional\\Models\\Citizenship\\Country |
| Province | Hanafalah\\ModuleRegional\\Models\\Regional\\Province |
| District | Hanafalah\\ModuleRegional\\Models\\Regional\\District |
| Subdistrict | Hanafalah\\ModuleRegional\\Models\\Regional\\Subdistrict |
| Village | Hanafalah\\ModuleRegional\\Models\\Regional\\Village |
| Service | Projects\\WellmedBackbone\\Models\\ModuleService\\Service |
| Unicode | Hanafalah\\LaravelSupport\\Models\\Unicode\\Unicode |
| ExaminationStuff | Hanafalah\\ModuleExamination\\Models\\ExaminationStuff |
| DosageForm | Hanafalah\\ModuleMedicalItem\\Models\\DosageForm |
| Education | Hanafalah\\ModulePeople\\Models\\Education |
| EmployeeType | Hanafalah\\ModuleEmployee\\Models\\EmployeeType\\EmployeeType |
| FamilyRole | Hanafalah\\ModulePeople\\Models\\FamilyRole |
| FreqUnit | Hanafalah\\ModuleMedicalItem\\Models\\FreqUnit |
| Funding | Hanafalah\\ModuleFunding\\Models\\Funding\\Funding |
| ItemStuff | Hanafalah\\ModuleItem\\Models\\ItemStuff |
| MasterVaccine | Hanafalah\\ModuleExamination\\Models\\MasterVaccine |
| MedicService | Projects\\WellmedBackbone\\Models\\ModuleMedicService\\MedicService |
| Occupation | Hanafalah\\ModuleProfession\\Models\\Occupation\\Occupation |
| PatientOccupation | Hanafalah\\ModulePatient\\Models\\Patient\\PatientOccupation |
| PatientTypeService | Hanafalah\\ModulePatient\\Models\\Patient\\PatientTypeService |
| PaymentMethod | Hanafalah\\ModulePayment\\Models\\Payment\\PaymentMethod |
| PeopleStuff | Hanafalah\\ModulePeople\\Models\\PeopleStuff |
| Profession | Hanafalah\\ModuleProfession\\Models\\Profession\\Profession |

---

## Connection & Table Mapping

Key entities untuk seeder migration:

| Entity | Connection | Table | Schema |
|--------|------------|-------|--------|
| **Geographic** ||||
| Country | central | public.countries | N/A |
| Province | central | public.provinces | N/A |
| District | central | public.districts | N/A |
| Subdistrict | central | public.subdistricts | N/A |
| Village | central | public.villages | N/A |
| **Lab & Radiology** ||||
| ClinicalPathology | tenant | public.examination_stuffs | N/A |
| Radiology | tenant | public.examination_stuffs | N/A |
| Sample | tenant | public.examination_stuffs | N/A |
| **Medical Items** ||||
| TherapeuticClass | tenant | public.unicodes | Flag-based |
| DosageForm | tenant | public.unicodes | Flag-based |
| FreqUnit | tenant | public.unicodes | Flag-based |
| MedicalNetUnit | tenant | public.unicodes | Flag-based |
| MedicalCompositionUnit | tenant | public.unicodes | Flag-based |
| CompositionUnit | tenant | public.unicodes | Flag-based |
| **Item Catalog** ||||
| ItemStuff | tenant | public.unicodes | Flag-based |
| Brand | tenant | public.unicodes | Flag-based |
| Trademark | tenant | public.unicodes | Flag-based |
| PackageForm | tenant | public.unicodes | Flag-based |
| UsageLocation | tenant | public.unicodes | Flag-based |
| UsageRoute | tenant | public.unicodes | Flag-based |
| **People & Organization** ||||
| Education | tenant | public.unicodes | Flag-based |
| FamilyRole | tenant | public.unicodes | Flag-based |
| Occupation | tenant | public.unicodes | Flag-based |
| PatientOccupation | tenant | public.unicodes | Flag-based |
| Profession | tenant | public.unicodes | Flag-based |
| **Employee** ||||
| EmployeeType | tenant | public.unicodes | Flag-based |
| EmployeeStuff | tenant | public.unicodes | Flag-based |
| **Service** ||||
| Service | tenant | public.services | N/A |
| MedicService | tenant | public.unicodes | Flag-based |
| ServiceCluster | tenant | public.unicodes | Flag-based |
| PaymentMethod | tenant | public.unicodes | Flag-based |
| **Examination** ||||
| ExaminationStuff | tenant | public.examination_stuffs | N/A |
| MasterVaccine | tenant | public.examination_stuffs | N/A |
| **Funding** ||||
| Funding | tenant | public.unicodes | Flag-based |
| **Patient Types** ||||
| PatientType | tenant | public.unicodes | Flag-based |
| PatientTypeService | tenant | public.unicodes | Flag-based |

---

## Connection Details

From `wellmed/connection.txt`:

### Central DB (Master)
- **Database**: `wellmed`
- **Schema**: `public`
- **Host**: wellmed-pgbouncer-local:5432
- **Use for**: Geographic data, User, Tenant, WellmedUnicode

### Tenant DB (Per-Clinic)
- **Database**: `clinic_{id}` (dynamic, contoh: `clinic_4`)
- **Schema**: `public`
- **Host**: wellmed-pgbouncer-local:5432
- **Use for**: All clinical/operational data (unicodes, services, patients, items)

### EMR Schema (Medical Records)
- **Database**: `clinic_{id}`
- **Schema**: `emr_2026`
- **Host**: wellmed-pgbouncer-local:5432
- **Use for**: Visit, Examination, PatientIllness, Referral

### POS Schema (Point of Sale)
- **Database**: `clinic_{id}`
- **Schema**: `pos_2026`
- **Host**: wellmed-pgbouncer-local:5432
- **Use for**: Billing, Invoice, Refund, WalletTransaction

### SCM Schema (Supply Chain)
- **Database**: `clinic_{id}`
- **Schema**: `scm_2026`
- **Host**: wellmed-pgbouncer-local:5432
- **Use for**: Procurement, PurchaseOrder, Stock

---

## Go Entity Mapping Strategy

Untuk Go seeder migration:

### Pattern 1: Unicode Table (Flag-based)
Entities yang pake table `unicodes` dengan field `flag`:
```go
type Unicode struct {
    Id       string         `gorm:"primaryKey"`
    Name     string         `gorm:"not null"`
    Label    *string
    Flag     string         `gorm:"not null;index"`  // Discriminator
    ParentID *string        `gorm:"index"`           // For hierarchy
    Props    datatypes.JSON
    // ... timestamps
}
```

### Pattern 2: Examination Stuffs Table
Entities yang pake table `examination_stuffs`:
```go
type ExaminationStuff struct {
    Id          string         `gorm:"primaryKey"`
    Name        string         `gorm:"not null"`
    Type        string         // Discriminator: ClinicalPathology, Radiology, Sample
    ServiceID   *string        `gorm:"index"`
    ParentID    *string        `gorm:"index"`  // For hierarchy
    Props       datatypes.JSON
    // ... relations & timestamps
}
```

### Pattern 3: Dedicated Table
Entities dengan dedicated table (Country, Province, Service, dll):
```go
// Check existing entity definition in wellmed-backbone/internal/schema/entity/
```

---

## Notes

1. **Multi-tenant Context**: Tenant DB dan schemas adalah dynamic per-clinic
2. **Unicode Pattern**: Mayoritas reference data di Go menggunakan `Unicode` entity dengan field `flag` sebagai discriminator
3. **Examination Stuffs**: Lab/Radiology/Sample menggunakan `examination_stuffs` table (bukan services!)
4. **Geographic Data**: Country/Province/District di central DB (shared across tenants)
5. **Hierarchy Support**: `ParentID` untuk nested structure (Profession, Occupation, dll)

---

## References

- Laravel Models: `wellmed/repositories/module-*/src/Models/`
- Laravel Schemas: `wellmed/repositories/module-*/src/Schemas/`
- Go Entities: `wellmed-backbone/internal/schema/entity/`
- Source Data: `wellmed/entity.txt`, `wellmed/connection_table.txt`, `wellmed/connection.txt`
