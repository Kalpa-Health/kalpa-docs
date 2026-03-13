# Go Seeder Migration - PRD

**Version:** 1.0
**Date:** 2026-03-13
**Author:** Development Team

---

## 1. Problem Statement

WellMed Go microservices memerlukan reference data yang konsisten untuk berfungsi. Saat ini:
- Laravel project memiliki 67 seeders dengan data lengkap
- Go project hanya memiliki 33 seeders (~50% coverage)
- Critical reference data (ClinicalPathology, ICD, TherapeuticClass) belum ada di Go
- Multi-tenant setup membutuhkan seeding per tenant database

---

## 2. Goals

### 2.1 Primary Goals
1. Migrate semua critical Laravel seeders ke Go
2. Maintain idempotent seeding (safe to run multiple times)
3. Support multi-tenant database architecture
4. Reduce manual data entry untuk new tenant setup

### 2.2 Success Metrics
- 100% coverage untuk critical reference data
- SeedLite() completes in < 30 seconds
- SeedPlus() completes in < 5 minutes
- Zero FK violations after seeding

---

## 3. Scope

### 3.1 In Scope
- Reference data seeders (Unicode, Service, etc.)
- Geographic data (Country, Province, District, Village)
- Medical classification (ICD, TherapeuticClass)
- Document templates (Informed Consent)
- Seeder orchestration (SeedLite, SeedPlus, SeedEMR)

### 3.2 Out of Scope
- User/Employee seeders (manual or admin-driven)
- Sample patient data (Faker-based, not reference)
- Permission/Role seeders (YAML-driven)
- Infrastructure seeders (API access, workspace)

---

## 4. Requirements

### 4.1 Functional Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-1 | Semua seeder harus idempotent (upsert pattern) | P0 |
| FR-2 | Support hierarchical data (3+ levels) | P0 |
| FR-3 | Support bulk import dari SQL/CSV | P0 |
| FR-4 | Support JSON Props field | P0 |
| FR-5 | SeedLite() harus include minimal reference data | P0 |
| FR-6 | SeedPlus() harus include all reference data | P1 |
| FR-7 | CLI flags untuk selective seeding | P2 |
| FR-8 | Progress logging untuk monitoring | P2 |

### 4.2 Non-Functional Requirements

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-1 | SeedLite() performance | < 30 seconds |
| NFR-2 | SeedPlus() performance | < 5 minutes |
| NFR-3 | Geographic seeding | < 2 minutes |
| NFR-4 | Zero FK violations | 100% |
| NFR-5 | Memory usage for bulk import | < 500MB |

---

## 5. Entity-Seeder Matrix

### 5.1 Unicode-based Entities

| Entity Flag | Laravel Seeder | Go Seeder | Status |
|-------------|----------------|-----------|--------|
| DosageForm | DosageFormSeeder | seed_dosage_form.go | ✅ |
| Profession | ProfessionSeeder | seed_profession.go | ✅ |
| Brand | BrandSeeder | seed_brand.go | ✅ |
| Funding | FundingSeeder | seed_funding.go | ✅ |
| Occupation | OccupationSeeder | seed_occupation.go | ✅ |
| FreqUnit | FreqUnitSeeder | seed_freq_unit.go | ✅ |
| Education | EducationSeeder | seed_education.go | ✅ |
| UsageLocation | UsageLocationSeeder | seed_usage_location.go | ✅ |
| UsageRoute | UsageRouteSeeder | seed_usage_route.go | ✅ |
| Trademark | TrademarkSeeder | seed_trademark.go | ✅ |
| ClinicalPathology | ClinicalPathologySeeder | seed_clinical_pathology.go | ❌ TODO |
| Radiology | RadiologySeeder | seed_radiology.go | ❌ TODO |
| Sample | SampleSeeder | seed_sample.go | ❌ TODO |
| CompositionUnit | CompositionUnitSeeder | seed_composition_unit.go | ❌ TODO |

### 5.2 Service-based Entities

| Entity | Laravel Seeder | Go Seeder | Status |
|--------|----------------|-----------|--------|
| PaymentMethod | PaymentMethodSeeder | seed_payment_method.go | ✅ |
| MedicService | MedicServiceSeeder | seed_medic_service.go | ✅ |
| ServiceCluster | ServiceClusterSeeder | seed_service_cluster.go | ❌ TODO |

### 5.3 Domain-Specific Entities

| Entity | Laravel Seeder | Go Seeder | Status |
|--------|----------------|-----------|--------|
| TherapeuticClass | TherapeuticClassSeeder | seed_therapeutic_class.go | ❌ TODO |
| Disease (ICD) | IcdSeeder | seed_icd.go | ❌ TODO |
| MasterInformedConsent | DocumentTypeSeeder | seed_document_type.go | ❌ TODO |
| ReportCollection | MasterReportCollectionSeeder | seed_report_collection.go | ❌ TODO |

### 5.4 Geographic Entities

| Entity | Laravel Seeder | Go Seeder | Status |
|--------|----------------|-----------|--------|
| Country | CountrySeeder | seed_country.go | ❌ TODO |
| Province | ProvinceSeeder | seed_province.go | ❌ TODO |
| District | RegionalSeeder | seed_district.go | ❌ TODO |
| SubDistrict | RegionalSeeder | seed_sub_district.go | ❌ TODO |
| Village | RegionalSeeder | seed_village.go | ❌ TODO |

---

## 6. Data Volume Estimates

| Entity | Records | Size | Import Method |
|--------|---------|------|---------------|
| Country | ~200 | 7 KB | Manual/JSON |
| Province | ~34 | 2 KB | Manual/JSON |
| District | ~500 | 34 KB | SQL file |
| SubDistrict | ~7,000 | 200 KB | SQL file |
| Village | ~82,000 | 7.8 MB | SQL file |
| Disease (ICD) | ~13,000 | 4.3 MB | SQL file |
| TherapeuticClass | ~250 | 50 KB | JSON file |
| ClinicalPathology | ~180 | 30 KB | Manual/JSON |
| Other Reference | ~500 | 100 KB | Manual |

**Total estimated:** ~100,000 records, ~12 MB data

---

## 7. Acceptance Criteria

### 7.1 Phase 1 (Geographic)
- [ ] seed_country.go seeds all countries
- [ ] seed_province.go seeds Indonesian provinces
- [ ] seed_village.go seeds villages from SQL
- [ ] Geographic seeding completes in < 2 minutes

### 7.2 Phase 2 (Medical Hierarchy)
- [ ] seed_clinical_pathology.go supports 3-level hierarchy
- [ ] seed_radiology.go supports 2-level hierarchy
- [ ] All parent-child relationships are valid

### 7.3 Phase 3 (Classification)
- [ ] seed_icd.go imports ~13K diseases
- [ ] seed_therapeutic_class.go supports deep hierarchy
- [ ] Classification seeding completes in < 3 minutes

### 7.4 Phase 4 (Documents)
- [ ] seed_document_type.go creates informed consent templates
- [ ] JSON Props field is properly populated

### 7.5 Phase 5 (Integration)
- [ ] SeedPlus() includes all new seeders
- [ ] Running SeedPlus() twice produces same result
- [ ] All FK constraints satisfied
- [ ] Total time < 5 minutes

---

## 8. Dependencies

### 8.1 Technical Dependencies
- GORM v2 with PostgreSQL driver
- `helper.UpdateOrCreate()` utility
- `helper.GenerateUlid()` for ID generation
- Multi-tenant `ConnectionManager`

### 8.2 Data Dependencies
- `diseases.sql` from Laravel project
- `villages.sql` from Laravel project
- JSON data files for complex structures

---

## 9. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Large SQL import timeout | HIGH | Use streaming/batch import |
| FK violations | HIGH | Validate dependency order |
| Memory overflow | MEDIUM | Batch processing for bulk data |
| Inconsistent data | MEDIUM | Use upsert pattern |
| Performance regression | LOW | Benchmark after each phase |

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-13 | Claude | Initial PRD |
