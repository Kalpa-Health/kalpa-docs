# Props Module Review — Human Decision Document

**Version:** 1.0
**Date:** 05 March 2026
**Purpose:** Walk through each module's props usage, annotate decisions inline, produce actionable output for developers.

---

# 1. How to Use This Document

For each module below, the current props usage is enumerated with the Go metadata struct and source file. Your job:

1. Read the "What's in props" section
2. Mark each field with a decision in the `[ ]` checkbox:
   - `[C]` = Columnize (promote to typed column + FK/index)
   - `[J]` = Replace with JOIN (delete from props, fetch at query time)
   - `[K]` = Keep in props (genuinely flexible, tenant-specific, or rarely queried)
   - `[D]` = Delete (unused, redundant, or legacy)
3. Add notes in the "Notes" line if context is needed for the developer

The output of this review feeds directly into the dev workflow plan as execution tasks.

---

# 2. Global Context

## 2.1 The `prop_` prefix pattern

All `prop_` prefixed keys were created by `helper.BuildProps()` (`wellmed-backbone/internal/helper/virtual_column.go`). This function takes a Go struct and serializes it into the props JSON with an auto-generated `prop_` prefix. This was the Go equivalent of Laravel's eager-load flattening — avoids N+1 JOINs by snapshotting related data at write time.

**In Go, this pattern is largely unnecessary.** GORM's `Preload()` or explicit JOINs solve the same problem without data duplication. The exception is when the snapshot has *business meaning* (e.g., "what was the patient's name at the time of this visit" for the visit artifact).

## 2.2 The indexed prop

`visit_registrations` has a B-tree expression index:
```sql
CREATE INDEX idx_vr_props_patient_id
ON emr_2026.visit_registrations ((props->'prop_visit_patient'->'patient'->>'id'));
```

This is a leading indicator for columnization. A field important enough to index should be a column.

## 2.3 Models with props column but NO data ever written

| Model | Entity File | Status |
|-------|------------|--------|
| PractitionerEvaluation | `entity/emr/practitioner_evaluation.go` | Always `{}` or null — safe to drop column |
| Prescription | `entity/emr/prescription.go` | No BuildProps calls in service — safe to drop column |

**Decision needed:** [ ] Drop `props` column from these two tables via migration?

---

# 3. Module-by-Module Review

## 3.1 visit_registration

**Source:** `internal/domain/visit_registration/service/visit_registration.go:46-50`
**Metadata struct:** `metadata.VisitRegistrationProps` (`internal/schema/metadata/props_visit_registration.go`)
**Indexed:** YES — `props->'prop_visit_patient'->'patient'->>'id'`

### What's in props:

| Key | Contents | Type | Decision |
|-----|----------|------|----------|
| `prop_visit_patient` | Nested: id, visit_code, flag, visited_at, patient_type_service_id, status | Snapshot | [ ] |
| `prop_visit_patient.patient` | Nested: id, uuid, name, medical_record, profile, patient_type_id/name, patient_occupation_id/name | Snapshot | [ ] |
| `prop_visit_patient.patient.people` | Nested: id, uuid, name, first_name, last_name, dob, pob, age, sex, phone_1, phone_2 | Snapshot | [ ] |
| `prop_visit_patient.patient.card_identity` | Nested: medical_record, old_medical_record, ihs_number, bpjs | Snapshot | [ ] |
| `prop_visit_patient.patient.people.card_identity` | Nested: nik, sim, passport, visa, kk, npwp | Snapshot | [ ] |
| `prop_visit_examination` | Nested: id, visit_examination_code, patient_id, visit_patient_id, is_commit, sign_off_at, is_addendum, status | Snapshot | [ ] |
| `prop_medic_service` | Nested: id, name, flag, label | Snapshot | [ ] |

**Notes:**
- This is the most deeply nested props structure in the system
- The patient.id inside prop_visit_patient is the one with the B-tree index
- visit_examination metadata tracks sign-off state

**Annotation space:**
```
Decision notes:
```

---

## 3.2 visit_patient

**Source:** `internal/domain/visit_patient/service/visit_patient.go:120-126, 312-316, 394-402`
**Metadata struct:** `metadata.VisitPatientMeta` (`internal/schema/metadata/props_visit_patient.go`)

### What's in props:

| Key | Contents | Type | Decision |
|-----|----------|------|----------|
| `prop_patient_type_service` | id, parent_id, name, flag, label | Snapshot | [ ] |
| `prop_family_role` | Unicode family role reference | Snapshot | [ ] |
| `prop_medic_service` | id, name, label, flag | Snapshot | [ ] |
| `prop_patient` | Full patient: id, uuid, name, medical_record, profile, patient_type_id/name | Snapshot | [ ] |
| `prop_patient.people` | Full people: id, uuid, name, first_name, last_name, dob, pob, age, sex, card_identity (nik, sim, etc), phone_1, phone_2 | Snapshot | [ ] |
| `prop_patient.card_identity` | medical_record, old_medical_record, ihs_number, bpjs | Snapshot | [ ] |
| `prop_visit_registration` | id, visit_registration_code, visit_patient_type, status | Snapshot | [ ] |
| `tenant_id` | Tenant identifier (base map, not prop_ prefixed) | Business | [ ] |
| `name` | Patient name (base map) | Business | [ ] |
| `status` | Visit status (base map, default "PROCESSING") | Business | [ ] |

**Notes:**
- Two BuildProps versions exist (legacy + V2)
- Base map includes tenant_id, name, status — these are clearly column candidates
- prop_patient duplicates data from the patient table

**Annotation space:**
```
Decision notes:
```

---

## 3.3 visit_examination

**Source:** `internal/domain/visit_examination/service/visit_examination.go:106, 166`
**Props column:** Exists in entity but **not directly written by this service**

### What's in props:

| Key | Contents | Type | Decision |
|-----|----------|------|----------|
| `prop_visit_patient.patient.id` | Read via gjson for context extraction | Read-only | [ ] |

**Notes:**
- This service only READS from props (via gjson path), never writes
- The props data comes from the parent visit_registration
- If visit_registration props are resolved, this read path changes too

**Annotation space:**
```
Decision notes:
```

---

## 3.4 frontline (Assessment prescriptions)

**Source:** `internal/domain/frontline/service/frontline.go:243-310`
**Entity:** Assessment (used for prescription data in frontline context)

### What's in props:

| Key | Contents | Type | Decision |
|-----|----------|------|----------|
| `prescription_type` | String: MedicinePrescription, MedicToolPrescription, MixPrescription | Business | [ ] |
| `created_at` | Timestamp | Business | [ ] |
| Dynamic fields | Flattened prescription model data — varies by prescription_type | Business | [ ] |

**Notes:**
- NO `prop_` prefix — these are direct business fields
- The dynamic schema per prescription type is the legitimate props use case (variable shape per row)
- prescription_type itself is structurally stable and queried — column candidate

**Annotation space:**
```
Decision notes:
```

---

## 3.5 referral

**Source:** `internal/domain/referral/service/referral.go:101-105, 211-214, 280-284, 341-344, 385-387`

### What's in props:

| Key | Contents | Type | Decision |
|-----|----------|------|----------|
| `prop_medic_service` | id, name, label, flag | Snapshot | [ ] |
| `prop_external_referral` | External referral business data (varies by referral type) | Business | [ ] |
| (empty `{}`) | Inpatient referrals store empty props | N/A | [ ] |

**Notes:**
- Three referral paths: external, outpatient, inpatient
- Inpatient path writes empty props — column can be nullable
- prop_external_referral contains actual business data for external referrals
- prop_medic_service is the same snapshot pattern as other modules

**Annotation space:**
```
Decision notes:
```

---

## 3.6 pharmacy_sale (saga builder)

**Source:** `internal/domain/pharmacy_sale/saga/builder.go:131-134, 400-417`

### What's in props:

| Key | Contents | Type | Decision |
|-----|----------|------|----------|
| `prop_patient_type_service` | id, flag, name, label | Snapshot | [ ] |
| POS transaction props | Sent as `"{}"` (empty) to POS system | N/A | [ ] |

**Notes:**
- Minimal props usage — only Step 1 (CreateVisitPatient) sets prop_patient_type_service
- POS transaction gets empty props — confirm this is always the case

**Annotation space:**
```
Decision notes:
```

---

## 3.7 visit_registration_referral

**Source:** `internal/domain/visit_registration_referral/service/visit_registration_referral.go:326-331, 437-442, 544-549`

### What's in props:

| Key | Contents | Type | Decision |
|-----|----------|------|----------|
| `prop_visit_patient` | Full VisitPatient map (extracted from parent props) | Snapshot | [ ] |
| `prop_medic_service` | id, flag, label, name | Snapshot | [ ] |

**Notes:**
- Creates new VisitRegistration records for referral workflows
- Props are copied/rebuilt from the parent visit_registration's props
- Same snapshot pattern as 3.1 — decisions here should be consistent with visit_registration

**Annotation space:**
```
Decision notes:
```

---

## 3.8 assessment

**Source:** `internal/domain/assessment/service/assessment.go:70-124`

### What's in props:

| Key | Contents | Type | Decision |
|-----|----------|------|----------|
| (no `prop_` prefix) | Raw examination data: Anthropometry, VitalSign, Symptom, etc. | Business | [ ] |
| Dynamic morphs | Variable shape per assessment type | Business | [ ] |

**Notes:**
- NO `prop_` prefix — uses marshalExamProps() with direct business data
- Variable shape per assessment type (legitimate props use case)
- Similar pattern to frontline prescriptions — genuinely variable data

**Annotation space:**
```
Decision notes:
```

---

# 4. Summary Decision Matrix

After completing the review above, fill in this summary:

| Module | prop_ Snapshot Keys | Business Keys | Recommended Action |
|--------|-------------------|---------------|-------------------|
| visit_registration | 3 (visit_patient, visit_examination, medic_service) | 0 | |
| visit_patient | 4 (patient_type_service, family_role, medic_service, patient) + visit_registration | 3 (tenant_id, name, status) | |
| visit_examination | 0 (reads only) | 0 | |
| frontline | 0 | 2+ (prescription_type, dynamic) | |
| referral | 1 (medic_service) | 1 (external_referral) | |
| pharmacy_sale | 1 (patient_type_service) | 0 | |
| visit_registration_referral | 2 (visit_patient, medic_service) | 0 | |
| assessment | 0 | Variable (exam data) | |

**Quick counts:**
- `prop_medic_service` appears in: visit_registration, visit_patient, referral, visit_registration_referral (4 modules)
- `prop_visit_patient` / `prop_patient` appears in: visit_registration, visit_patient, visit_registration_referral (3 modules)
- These are the two highest-impact refactors if decided to columnize/JOIN

---

# 5. Output Checklist

After this review session:

- [ ] Transfer decisions to dev workflow plan as execution tasks
- [ ] For `[C]` decisions: draft migration SQL (add column, backfill from props, drop from props)
- [ ] For `[J]` decisions: identify which service methods need JOIN rewrites
- [ ] For `[D]` decisions: confirm no frontend dependency, then remove
- [ ] For `[K]` decisions: document the schema in a comment or metadata struct
- [ ] Update `visit_registrations` index to use new column once prop_visit_patient.patient.id is columnized

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 05 Mar 2026 | Alex | Initial review document from codebase analysis |
