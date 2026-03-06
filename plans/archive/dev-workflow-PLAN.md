# WellMed Go — Dev Workflow Plan

**Version:** 1.0
**Date:** 05 March 2026
**Purpose:** Itemized task list for the team. Ordered by size: small fixes first, then execution tasks, then new workstreams.

---

# 1. Small Fixes (One Person, One PR Each)

## 1.1 Gateway Housekeeping

- [x] Fix `kalpa-health` casing in consultation proto `go_package` declarations — several files still use `Kalpa-Health` mixed case (`wellmed-consultation/proto/*.proto`)
- [X] Move `BACKBONE_GRPC_ADDRESS` into `config/env.go` — currently inline `os.Getenv()` in `internal/route/api.go:83` and `internal/app/app.go`
- [X] Remove debug log `log.Infof("RESP (raw): %+v", resp.Data)` from `visit_examination/service/visit_examination.go:32`
- [N/A] ~~Wire input validation middleware to all mutation routes~~ — **reverted, not applicable.** Gateway is intentionally a thin proxy; pass-through handlers send raw `ctx.Body()` as string to gRPC. The `role` handler is the deliberate exception — it uses a typed service interface and `RequestValidator` correctly. Validation on pass-through routes adds a redundant parse with no real benefit since the gRPC service validates the payload. Task removed from backlog.

## 1.2 Inter-Service Authentication (ADR-007)

- [ ] **Backbone:** replace single shared token check with a `map[token → service_name]` loaded from env vars (`BACKBONE_API_KEY_GATEWAY`, `BACKBONE_API_KEY_CONSULTATION`); log `caller=<service_name>` via gRPC interceptor
- [ ] **Gateway:** rename env var from shared `BACKBONE_API_KEY` to service-specific `BACKBONE_API_KEY` (value changes, key name can stay, but SSM path changes)
- [ ] **Consultation:** same as gateway
- [ ] **SSM manifests:** add `/wellmed/{env}/gateway/BACKBONE_API_KEY` and `/wellmed/{env}/consultation/BACKBONE_API_KEY` to `wellmed-infrastructure` manifests; remove shared key entry
- [ ] Rotate all tokens after migration — verify each service independently before removing old shared key

## 1.3 Backbone Housekeeping

- [X] Drop `props` column from `practitioner_evaluation` table — confirm zero non-empty rows first (see Section 2 pre-flight 2.4), then run Phase 6 SQL + remove `Props` field from `entity/emr/practitioner_evaluation.go` in both repos
- [X] Drop `props` column from `prescription` table — same pre-flight check required (see 2.4)
- [X] Audit non-clinical tables with props columns for empty/unused props (geographic tables: provinces, districts, subdistricts, villages; access control: roles, permissions, personal_access_tokens; product: versions, installed_features) — drop column where consistently null

## 1.4 Documentation Gaps

- [ ] Fill in `operations/database-migrations.md` — 8 TODO items covering migration tooling, naming conventions, rollback procedures, multi-tenant execution
- [ ] Remove or replace `services/emr.md` stub — superseded by `services/consultation.md`
- [ ] Update `audit-findings.md` and `laravel-go-pattern-map.md` with resolution notes from this week's work

---

# 2. Props Resolution (ADR-008 — see `plans/props-migration/`)

Code changes for Phases 1–3 are **complete and committed** in both backbone and consultation. Remaining steps are database operations that must run in sequence after the wider dev push is deployed.

## 2.1 Pre-flight (DB — do before data cleanup)

- [X] **2.1a** Confirm `visit_patient_id` FK column has a B-tree index on `visit_registrations` in dev:
  ```sql
  CREATE INDEX IF NOT EXISTS idx_vr_visit_patient_id ON emr_2026.visit_registrations (visit_patient_id);
  ```
- [ ] **2.1b** Drop the old expression index in dev and confirm query plan switches to FK path:
  ```sql
  DROP INDEX IF EXISTS emr_2026.idx_vr_props_patient_id;
  -- then: EXPLAIN ANALYZE SELECT ... FROM visit_registrations WHERE visit_patient_id = '...'
  ```
- [X] **2.1c** Confirm zero non-empty rows in prod for the columns targeted for drop in Phase 6:
  ```sql
  SELECT COUNT(*) FROM emr_2026.practitioner_evaluations WHERE props IS NOT NULL AND props::text != '{}';
  SELECT COUNT(*) FROM emr_2026.prescriptions WHERE props IS NOT NULL AND props::text != '{}';
  ```

## 2.2 Phase 4 — Data Cleanup Migration (after deploy to staging)

Strip dead `prop_*` keys from all existing rows. Runs after Phases 1–3 are live so no new snapshot data is being written.

- [ ] Run the following on **staging** first, verify row counts, spot-check sample rows:
  ```sql
  UPDATE emr_2026.visit_registrations
  SET props = (SELECT jsonb_object_agg(key, value) FROM jsonb_each(props) WHERE key NOT LIKE 'prop_%')
  WHERE props::text != '{}' AND EXISTS (SELECT 1 FROM jsonb_object_keys(props) k WHERE k LIKE 'prop_%');

  UPDATE emr_2026.visit_patients
  SET props = (SELECT jsonb_object_agg(key, value) FROM jsonb_each(props) WHERE key NOT LIKE 'prop_%')
  WHERE props::text != '{}' AND EXISTS (SELECT 1 FROM jsonb_object_keys(props) k WHERE k LIKE 'prop_%');

  UPDATE emr_2026.referrals
  SET props = (SELECT jsonb_object_agg(key, value) FROM jsonb_each(props) WHERE key NOT LIKE 'prop_%')
  WHERE props::text != '{}' AND EXISTS (SELECT 1 FROM jsonb_object_keys(props) k WHERE k LIKE 'prop_%');
  ```
- [ ] Run in **prod** during low-traffic window (batch if tables are large)

## 2.3 Phase 5 — Drop Expression Index (after Phase 4 in prod)

- [ ] Drop the expression index in prod (pre-flight 2.1b covers dev/staging):
  ```sql
  DROP INDEX IF EXISTS emr_2026.idx_vr_props_patient_id;
  ```

## 2.4 Phase 6 — Drop Unused Props Columns (after pre-flight 2.1c confirms zero rows)

- [ ] Run in staging then prod:
  ```sql
  ALTER TABLE emr_2026.practitioner_evaluations DROP COLUMN IF EXISTS props;
  ALTER TABLE emr_2026.prescriptions DROP COLUMN IF EXISTS props;
  ```
- [ ] Remove `Props` field from `entity/emr/practitioner_evaluation.go` and `entity/emr/prescription.go` in both backbone and consultation — PR after DB column is dropped

## 2.5 Remaining Non-DB Props Items

- [ ] Proto contract typing for clinical modules — replace `string data` / `string payload` with typed proto messages in the 8 clinical module protos (gateway + backbone + consultation coordinated change)
- [ ] Consument type formalization — define `BillingParty` typed struct to replace `interface{}` in transaction meta, align across POS and gateway

---

# 3. Saga & Visit Flow

- [ ] E2E test CreateVisit saga on dev environment — deploy backbone + consultation, run full trigger-callback cycle
- [ ] DoctorSignOff saga (Phase 5 of visit-saga-wiring) — backbone builder (5 steps: validate, finalize, billing, canonical, SATU SEHAT) + consultation step handler
- [ ] Write ADR or plan doc for DoctorSignOff saga scope before implementation

---

# 4. Front-End API Wiring

- [ ] Continue screen-by-screen patient/visit flow debugging — props cleanup (Section 2) directly unblocks typing issues encountered here
- [ ] As props decisions land, update gateway response shapes and coordinate with frontend on field path changes

---

# 5. Microservice Extractions (New Workstreams — Need Scoping Plans)

## 5.1 Pharmacy MS Extraction

- [X] Write scoping plan (similar to `plans/consultation-extraction/`) — define which backbone domains move out
- [X] Likely scope: pharmacy_sale domain + related saga builders
- [X] Create `wellmed-pharmacy` repo, proto definitions, gRPC service registration

## 5.2 Scheduling MS Extraction

- [ ] Write scoping plan — define scheduling domain boundaries
- [ ] Current state: shift management (`entity/shift.go`) + visit scheduling logic embedded in visit domain
- [ ] Define what moves vs. what stays in backbone (shift management, appointment booking, room/practitioner availability)
- [ ] Create `wellmed-scheduler` repo

## 5.3 POS as wellmed-cashier

- [ ] Write scoping plan — define POS/financial domain boundaries
- [ ] Current scope in backbone: 13 POS entities, 4 gRPC services (Transaction, Billing, Invoice, PaymentSummary), plus pharmacy_sale integration
- [ ] Key entities: Transaction, Invoice, Billing, PaymentSummary, PaymentDetail, TransactionItem, SplitPayment, WalletTransaction, Refund, TransactionHasConsument
- [ ] Wallet/payment strategy patterns add complexity — scope carefully
- [ ] Create `wellmed-cashier` repo

---

# 6. Pending Documentation TODOs (From Existing Docs)

- [ ] `services/consultation.md` — fill remaining TODO sections if any
- [ ] `development/saga-backlog.md` — update saga statuses as DoctorSignOff progresses
- [ ] `plans/visit-saga-wiring/visit-saga-wiring-PROGRESS.md` — update Phase 5 status
- [ ] `plans/consultation-extraction/` — update with ops/deployment status once E2E runs

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 05 Mar 2026 | Alex | Initial task list from audit review discussion + codebase analysis |
| 1.1 | 06 Mar 2026 | Alex | Section 2 rewritten: props code changes complete (Phases 1–3), remaining DB steps sequenced as 2.1–2.4 |
