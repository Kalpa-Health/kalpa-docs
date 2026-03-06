# Props Migration Plan

**Version:** 1.0
**Date:** 2026-03-06
**Policy:** ADR-008 — Props Field Classification Policy
**Scope:** wellmed-backbone, wellmed-consultation (parallel); wellmed-pos (Phase 7, at intake)

---

# 1. Overview

This plan removes all [J] (snapshot) data from `props` columns across the Kalpa EMR services, leaving only [K] (legitimately variable) data in place. No new columns are added — all [J] reads are replaced with relational JOINs on existing FK columns.

**Classification summary going into this migration:**

| Class | Action | Examples |
|-------|--------|---------|
| [K] Keep | Leave as-is; document schema | Assessment exam data, frontline prescription payload, external referral payload |
| [J] Join | Remove write; replace read with JOIN/Preload | All `prop_*` keys, visit_patient tenant_id/name/status in props |
| [D] Drop | Drop column after confirming zero reads | `practitioner_evaluations.props`, `prescriptions.props` |

No [C] (columnize) items were identified. No new column migrations required.

---

# 2. Pre-flight

- [ ] **2.1** Grep all repos (backbone, consultation, gateway, any frontend) for each `prop_*` key listed in section 3. Log any reads outside the domain service itself. These are additional callsites that need updating before writes are removed.
  - Keys to search: `prop_visit_patient`, `prop_medic_service`, `prop_visit_registration`, `prop_family_role`, `prop_patient_type_service`, `prop_visit_examination`, `prop_patient`, `prop_external_referral`
- [ ] **2.2** Confirm FK indexes exist on all columns that will replace the props reads (visit_registrations → visit_patient_id, visit_patients → patient_id, etc.). If any are missing, add them before removing props reads to avoid full-table scans.
- [ ] **2.3** Drop the `visit_registrations` props expression index in dev and confirm query plan uses FK path instead.
  ```sql
  DROP INDEX emr_2026.idx_vr_props_patient_id;
  ```
- [ ] **2.4** Confirm `prescriptions.props` and `practitioner_evaluations.props` have zero non-null / non-`{}` rows in prod.
  ```sql
  SELECT COUNT(*) FROM emr_2026.prescriptions WHERE props IS NOT NULL AND props::text != '{}';
  SELECT COUNT(*) FROM emr_2026.practitioner_evaluations WHERE props IS NOT NULL AND props::text != '{}';
  ```

---

# 3. Phase 1 — Remove Snapshot Writes (wellmed-backbone)

Stop writing `prop_*` keys. After this phase, new rows will have clean props (only [K] data or empty `{}`).

## 3.1 visit_registration service

**File:** `internal/domain/visit_registration/service/visit_registration.go`

- [ ] **3.1.1** Remove `BuildPropsWithFilter` call (line ~46) that writes `prop_visit_patient`, `prop_medic_service`, `prop_visit_examination`
- [ ] **3.1.2** Set `Props: datatypes.JSON("{}")` or omit if column is nullable

## 3.2 visit_patient service

**File:** `internal/domain/visit_patient/service/visit_patient.go`

- [ ] **3.2.1** Remove `BuildPropsWithFilter` call (~line 112) writing `prop_patient_type_service`
- [ ] **3.2.2** Remove `BuildPropsWithFilter` call (~line 200) writing `prop_family_role`
- [ ] **3.2.3** Remove `BuildPropsV2` call (~line 295) writing `prop_medic_service`, `prop_visit_patient`
- [ ] **3.2.4** Remove `BuildPropsWithFilter` call (~line 362) — confirm what keys this writes, remove all `prop_*` entries
- [ ] **3.2.5** Remove `BuildPropsWithFilter` call (~line 376) writing `prop_patient`, `prop_patient_type_service`, `prop_visit_registration`
- [ ] **3.2.6** Remove base map entries `tenant_id`, `name`, `status` from props (these are real columns)

## 3.3 visit_registration_referral service

**File:** `internal/domain/visit_registration_referral/service/visit_registration_referral.go`

- [ ] **3.3.1** Remove `BuildPropsWithFilter` call (~line 319) writing `prop_visit_patient`, `prop_medic_service`
- [ ] **3.3.2** Remove `BuildPropsWithFilter` call (~line 354) — confirm keys, remove `prop_*`
- [ ] **3.3.3** Remove `BuildPropsWithFilter` call (~line 430) writing `prop_visit_patient`, `prop_medic_service`
- [ ] **3.3.4** Remove `BuildPropsWithFilter` call (~line 490) — confirm keys, remove `prop_*`
- [ ] **3.3.5** Remove `BuildPropsWithFilter` call (~line 536) writing `prop_visit_patient`, `prop_medic_service`
- [ ] **3.3.6** Remove `BuildPropsWithFilter` call (~line 641) writing `prop_external_referral` — **leave [K]**: this stays

## 3.4 referral service

**File:** `internal/domain/referral/service/referral.go`

- [ ] **3.4.1** Remove `prop_medic_service` from `BuildPropsWithFilter` call (~line 99) — leave `prop_external_referral` [K]
- [ ] **3.4.2** Remove `prop_medic_service` from calls at lines ~209, ~278, ~339, ~366
- [ ] **3.4.3** Remove `prop_visit_patient` from call at line ~278 (referral also writes this snapshot)

## 3.5 frontline service

**File:** `internal/domain/frontline/service/frontline.go`

- [ ] **3.5.1** Remove `prop_visit_patient` and `prop_visit_examination` key references from the `BuildProps` filter list (~lines 337-338)
- [ ] **3.5.2** Keep the remainder of `BuildProps` call (~line 298) — `prescription_type`, `created_at`, and dynamic prescription fields are [K]

---

# 4. Phase 2 — Replace Props Reads with JOINs (wellmed-backbone)

Remove all `gjson.Get(props, "prop_*")` reads and replace with relational queries.

## 4.1 visit_examination service

**File:** `internal/domain/visit_examination/service/visit_examination.go`

- [ ] **4.1.1** Replace `gjson.Get(string(visitRegistration.Props), "prop_visit_patient.patient.id")` (~line 106) with a JOIN through `visit_registration → visit_patient → patient` to retrieve `patient.id`
- [ ] **4.1.2** Same replacement at ~line 166
- [ ] **4.1.3** Confirm the query path: `visit_registrations.visit_patient_id → visit_patients.patient_id` (or equivalent FK). Add GORM Preload/Association as appropriate.

## 4.2 Any other gjson reads

- [ ] **4.2.1** Grep backbone for remaining `gjson.Get.*prop_` after Phase 1 — there should be none; fix any found

---

# 5. Phase 3 — Mirror Phases 1 & 2 in wellmed-consultation

wellmed-consultation is a full mirror of the same domain services. All changes from Phases 1 and 2 apply identically.

**Files to update (same structure as backbone):**

- [ ] **5.1** `internal/domain/visit_registration/service/visit_registration.go` — mirror 3.1
- [ ] **5.2** `internal/domain/visit_patient/service/visit_patient.go` — mirror 3.2
- [ ] **5.3** `internal/domain/visit_registration_referral/service/visit_registration_referral.go` — mirror 3.3
- [ ] **5.4** `internal/domain/referral/service/referral.go` — mirror 3.4
- [ ] **5.5** `internal/domain/frontline/service/frontline.go` — mirror 3.5
- [ ] **5.6** `internal/domain/visit_examination/service/visit_examination.go` — mirror 4.1
- [ ] **5.7** Grep consultation for remaining `gjson.Get.*prop_` — fix any found

**Note:** Consultation also has `prop_visit_patient` being read at `visit_registration_referral.go:125` to extract sub-keys for a referral rebuild. This specific read needs special attention — trace what it does with `visitPatientProps` and replace with a direct query.

---

# 6. Phase 4 — Data Cleanup Migration

Strip dead `prop_*` keys from existing rows in prod. Runs after Phases 1-3 are deployed (so no new snapshot data is being written).

- [ ] **6.1** Write a Go migration (or raw SQL migration) that strips all `prop_*` keys from props columns on affected tables. Target tables: `visit_registrations`, `visit_patients`, `visit_registration_referral`, `referrals`

  ```sql
  -- Example pattern for each table (run per table)
  UPDATE emr_2026.visit_registrations
  SET props = (
    SELECT jsonb_object_agg(key, value)
    FROM jsonb_each(props)
    WHERE key NOT LIKE 'prop_%'
  )
  WHERE props::text != '{}'
    AND EXISTS (
      SELECT 1 FROM jsonb_object_keys(props) k WHERE k LIKE 'prop_%'
    );
  ```

- [ ] **6.2** Run cleanup migration in staging; verify row count matches expected, spot-check a sample of rows
- [ ] **6.3** Run in prod during low-traffic window; this is a bulk UPDATE — batch if table is large

---

# 7. Phase 5 — Drop Expression Index

- [ ] **7.1** Drop the `visit_registrations` props expression index (superseded by FK column index):
  ```sql
  DROP INDEX IF EXISTS emr_2026.idx_vr_props_patient_id;
  ```
- [ ] **7.2** Confirm the FK column (`visit_patient_id` or equivalent) has a standard B-tree index. Add if missing:
  ```sql
  CREATE INDEX IF NOT EXISTS idx_vr_visit_patient_id ON emr_2026.visit_registrations (visit_patient_id);
  ```

---

# 8. Phase 6 — Drop Unused Props Columns

- [ ] **8.1** Confirm zero non-empty rows (see 2.4), then drop:
  ```sql
  ALTER TABLE emr_2026.practitioner_evaluations DROP COLUMN IF EXISTS props;
  ALTER TABLE emr_2026.prescriptions DROP COLUMN IF EXISTS props;
  ```
- [ ] **8.2** Remove `Props` field from `entity/emr/practitioner_evaluation.go` and `entity/emr/prescription.go` in both backbone and consultation

---

# 9. Phase 7 — POS Repo Intake

- [ ] **9.1** When wellmed-pos is onboarded, classify all existing `props` usage against ADR-008 before writing any new service code
- [ ] **9.2** Apply [J] removals and [K] documentation as part of the intake PR
- [ ] **9.3** Do not introduce any new `prop_*` prefixed keys

---

# 10. Checkpoints

| After Phase | Deploy to dev | Integration test | Deploy to staging | Prod |
|-------------|--------------|-----------------|-------------------|------|
| Phase 1 + 2 (backbone) | Yes | Visit registration full flow | Yes | No |
| Phase 3 (consultation) | Yes | Consultation flow end-to-end | Yes | No |
| Phase 4 (cleanup migration) | Staging first | Row count + spot check | Yes | Low-traffic window |
| Phase 5 + 6 (drop index/columns) | Yes | Query plan review | Yes | Yes |

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-06 | Alex Knecht | Initial plan from props module review and ADR-008 |
