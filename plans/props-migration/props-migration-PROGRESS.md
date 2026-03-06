# Progress Log: Props Migration

---

## Session: 2026-03-06T00:00:00Z

### Phase 0: Pre-flight
- **Status**: ✅ DONE (partial — 2.3 and 2.4 require DB access, flagged as HUMAN)
- **Completed**: 2026-03-06

#### 2.1 — Cross-repo grep for prop_* reads
- **Status**: ✅ DONE
- **Findings**:
  - backbone: all prop_* writes confirmed in domain services (visit_registration, visit_patient, visit_registration_referral, referral, frontline, pharmacy_sale)
  - consultation: identical pattern, all same files
  - gateway: zero prop_* reads (proto files only) ✅
  - No external consumers outside domain services found
- **Gap found**: `wellmed-backbone/internal/domain/pharmacy_sale/saga/builder.go:133` writes `prop_patient_type_service` — **not included in plan Phase 1 tasks**. Added as task 3.6 during execution.

#### 2.2 — FK index confirmation (code-level)
- **Status**: ✅ DONE (code-level confirmed; DB-level is developer responsibility)
- **Findings**:
  - `visit_registrations.VisitPatientID` — `gorm:"index:vr_visit_ref,priority:1"` ✅
  - `visit_patients.PatientID` — `gorm:"type:char(26);index"` ✅
  - No additional index migrations needed

#### 2.3 — Drop expression index on visit_registrations (dev DB)
- **Status**: ⏸️ WAITING_HUMAN
- **Action required**: Run in dev DB:
  ```sql
  DROP INDEX emr_2026.idx_vr_props_patient_id;
  ```
  Confirm query plan uses FK path before proceeding to prod.

#### 2.4 — Confirm zero rows in prescriptions.props and practitioner_evaluations.props
- **Status**: ⏸️ WAITING_HUMAN
- **Action required**: Run in prod:
  ```sql
  SELECT COUNT(*) FROM emr_2026.prescriptions WHERE props IS NOT NULL AND props::text != '{}';
  SELECT COUNT(*) FROM emr_2026.practitioner_evaluations WHERE props IS NOT NULL AND props::text != '{}';
  ```
  Report results before Phase 6 executes.

---

### Task 3.1.1 + 3.1.2: visit_registration — remove snapshot writes (backbone)
- **Status**: ✅ DONE
- **Started**: 2026-03-06
- **Completed**: 2026-03-06
- **Files modified**: `wellmed-backbone/internal/domain/visit_registration/service/visit_registration.go`

### Task 3.2.1–3.2.6: visit_patient — remove snapshot writes (backbone)
- **Status**: ✅ DONE
- **Started**: 2026-03-06
- **Completed**: 2026-03-06
- **Files modified**: `wellmed-backbone/internal/domain/visit_patient/service/visit_patient.go`

### Task 3.3.1–3.3.5: visit_registration_referral — remove snapshot writes (backbone)
- **Status**: ✅ DONE
- **Started**: 2026-03-06
- **Completed**: 2026-03-06
- **Files modified**: `wellmed-backbone/internal/domain/visit_registration_referral/service/visit_registration_referral.go`

### Task 3.4.1–3.4.3: referral — remove snapshot writes (backbone)
- **Status**: ✅ DONE
- **Started**: 2026-03-06
- **Completed**: 2026-03-06
- **Files modified**: `wellmed-backbone/internal/domain/referral/service/referral.go`

### Task 3.5.1–3.5.2: frontline — remove prop_visit_patient/prop_visit_examination from filter (backbone)
- **Status**: ✅ DONE
- **Started**: 2026-03-06
- **Completed**: 2026-03-06
- **Files modified**: `wellmed-backbone/internal/domain/frontline/service/frontline.go`

### Task 3.6 (GAP): pharmacy_sale — remove prop_patient_type_service write (backbone)
- **Status**: ✅ DONE
- **Started**: 2026-03-06
- **Completed**: 2026-03-06
- **Files modified**: `wellmed-backbone/internal/domain/pharmacy_sale/saga/builder.go`

### Task 4.1.1–4.1.3: visit_examination — replace gjson reads with JOIN (backbone)
- **Status**: ✅ DONE
- **Started**: 2026-03-06
- **Completed**: 2026-03-06
- **Files modified**: `wellmed-backbone/internal/domain/visit_examination/service/visit_examination.go`

### Task 4.2.1: Grep backbone for remaining gjson.Get.*prop_ reads
- **Status**: ✅ DONE
- **Findings**: None remaining after Phase 1+2 changes

### Task 5.1–5.7: Mirror Phases 1 & 2 in wellmed-consultation
- **Status**: ✅ DONE
- **Started**: 2026-03-06
- **Completed**: 2026-03-06
- **Files modified**:
  - `wellmed-consultation/internal/domain/visit_registration/service/visit_registration.go`
  - `wellmed-consultation/internal/domain/visit_patient/service/visit_patient.go`
  - `wellmed-consultation/internal/domain/visit_registration_referral/service/visit_registration_referral.go`
  - `wellmed-consultation/internal/domain/visit_registration_referral/wires.go`
  - `wellmed-consultation/internal/domain/referral/service/referral.go`
  - `wellmed-consultation/internal/domain/frontline/service/frontline.go`
  - `wellmed-consultation/internal/domain/visit_examination/service/visit_examination.go`
- **Build**: `go build ./...` passes ✅

---

## 🔲 CHECKPOINT: Phases 1–3 complete — deploy to dev before continuing

All code changes for Phases 1 (backbone write removal), 2 (backbone gjson→JOIN), and 3 (consultation mirror) are complete. Both repos build clean.

**Review before continuing to Phase 4:**
- Deploy backbone + consultation to dev
- Run full visit registration flow end-to-end
- Run consultation visit examination flow
- Confirm no regressions on frontline/dispense, referral, VRR handlers

**Next phase (Phase 4) is database operations — do not run until E2E test passes.**

**Resume:** "continue the props-migration plan"
