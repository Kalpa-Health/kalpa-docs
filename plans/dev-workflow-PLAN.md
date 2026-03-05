# WellMed Go — Dev Workflow Plan

**Version:** 1.0
**Date:** 05 March 2026
**Purpose:** Itemized task list for the team. Ordered by size: small fixes first, then execution tasks, then new workstreams.

---

# 1. Small Fixes (One Person, One PR Each)

## 1.1 Gateway Housekeeping

- [x] Fix `kalpa-health` casing in consultation proto `go_package` declarations — several files still use `Kalpa-Health` mixed case (`wellmed-consultation/proto/*.proto`)
- [x] Move `BACKBONE_GRPC_ADDRESS` into `config/env.go` — currently inline `os.Getenv()` in `internal/route/api.go:83` and `internal/app/app.go`
- [x] Remove debug log `log.Infof("RESP (raw): %+v", resp.Data)` from `visit_examination/service/visit_examination.go:32`
- [ ] Wire input validation middleware (`validation.RequestValidator[T]()`) to all mutation routes in `route/api.go` — 11 unvalidated POST/PUT routes identified:
  - `medicalTreatmentGroup.Post` (line 338)
  - `itemGroup.Post` (line 360)
  - `patientGroup.Post` (line 368)
  - `patientGroup.Post("/:patient_id/visit-patient")` (line 371)
  - `visitRegistrationGroup.Post("/:visit_registration_id/visit-examination")` (line 378)
  - `visitRegistrationGroup.Put("/:visit_registration_id/visit-examination")` (line 379)
  - `visitRegistrationGroup.Post("/:visit_registration_id/referral")` (line 383)
  - `referralGroup.Post` (line 390)
  - `frontlineGroup.Post` (line 404)
  - `frontlineGroup.Put("/:frontline_id/examination/:type")` (line 405)
  - `pharmacySaleGroup.Post` (line 411)
- [ ] Requires: define DTO structs per endpoint (use existing `roleDto.RoleRequest` as template)

## 1.2 Inter-Service Authentication (ADR-007)

- [ ] **Backbone:** replace single shared token check with a `map[token → service_name]` loaded from env vars (`BACKBONE_API_KEY_GATEWAY`, `BACKBONE_API_KEY_CONSULTATION`); log `caller=<service_name>` via gRPC interceptor
- [ ] **Gateway:** rename env var from shared `BACKBONE_API_KEY` to service-specific `BACKBONE_API_KEY` (value changes, key name can stay, but SSM path changes)
- [ ] **Consultation:** same as gateway
- [ ] **SSM manifests:** add `/wellmed/{env}/gateway/BACKBONE_API_KEY` and `/wellmed/{env}/consultation/BACKBONE_API_KEY` to `wellmed-infrastructure` manifests; remove shared key entry
- [ ] Rotate all tokens after migration — verify each service independently before removing old shared key

## 1.3 Backbone Housekeeping

- [ ] Drop `props` column from `practitioner_evaluation` table — always empty/null, never written
- [ ] Drop `props` column from `prescription` table — no BuildProps calls in service layer
- [ ] Audit non-clinical tables with props columns for empty/unused props (geographic tables: provinces, districts, subdistricts, villages; access control: roles, permissions, personal_access_tokens; product: versions, installed_features) — drop column where consistently null

## 1.4 Documentation Gaps

- [ ] Fill in `operations/database-migrations.md` — 8 TODO items covering migration tooling, naming conventions, rollback procedures, multi-tenant execution
- [ ] Remove or replace `services/emr.md` stub — superseded by `services/consultation.md`
- [ ] Update `audit-findings.md` and `laravel-go-pattern-map.md` with resolution notes from this week's work

---

# 2. Props Resolution (Output of Plan 1 Review)

- [ ] Complete props module review session using `plans/props-module-review/props-module-review-PLAN.md`
- [ ] Execute `[C]` decisions — write migration SQL for columnized fields, backfill from props JSON, add indexes
- [ ] Execute `[J]` decisions — rewrite service methods to use JOINs instead of props snapshots, update response builders
- [ ] Execute `[D]` decisions — remove unused props keys after confirming no frontend dependency
- [ ] Replace `visit_registrations` B-tree expression index (`idx_vr_props_patient_id`) with proper column index once patient_id is columnized
- [ ] Update `helper.BuildProps()` callers as modules are resolved — track which modules still need the function
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

- [ ] Write scoping plan (similar to `plans/consultation-extraction/`) — define which backbone domains move out
- [ ] Likely scope: pharmacy_sale domain + related saga builders
- [ ] Create `wellmed-pharmacy` repo, proto definitions, gRPC service registration

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
