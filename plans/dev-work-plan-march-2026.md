# WellMed Go — Dev Work Plan: March 2026

**Version:** 1.0
**Date:** 06 March 2026
**Purpose:** Active work plan for the March go-live sprint. Supersedes `dev-workflow-PLAN.md`.

---

## Sprint Context

**Goal:** Go-live in ~2 weeks with the current Go-lite architecture.
**Constraint:** CI/CD hardening and ops doc work deferred to post-Lebaran (Idul Fitri holiday).
**Team focus:** Validating open PRs → Doctor Sign-Off flow.
**Alex focus:** wellmed-appointment scaffold + SSM migration + auth token reversal.

---

# 1. Infrastructure Pre-Flight

*Do this before E2E testing. Gives the team a foothold when live debugging surfaces issues.*

- [ ] **1.1** Verify RabbitMQ exchange/queue declarations match code across all services:
  - Exchange: `wellmed.saga` (backbone → all saga participants)
  - Queue: `pharmacy.steps` (pharmacy consumer), `cashier.steps` (cashier consumer)
  - Verify exchange type, durable flags, and routing key bindings are consistent between code and broker config
- [ ] **1.2** Verify Redis addressing in all services — confirm `REDIS_ADDR` is set correctly and session/saga-status cache is reachable from each service container
- [ ] **1.3** Add a brief note (or inline comment in each consumer's startup) listing exchange names, queue names, and Redis key prefixes — reference for the team during debugging

---

# 2. E2E Validation Gate

*Nothing in §9 (Consultation Phase 2) or §10 (Pharmacy Phase 5) should proceed until this passes.*

- [ ] **2.1** Merge open PRs; deploy backbone + consultation to dev VM
- [ ] **2.2** Run CreateVisit saga end-to-end: trigger → backbone orchestrator → consultation step handler → callback → saga completion
- [ ] **2.3** Fix `consultation/internal/queue/consumer.go:116` — unrouted messages are currently logged only; should be dead-lettered. Fix before E2E to avoid silent failures during testing
- [ ] **2.4** Note: `consultation/internal/domain/prescription/service.go:68` — prescription saga callback stub does not block the CreateVisit E2E path but must be wired before DoctorSignOff (§3)

---

# 3. Doctor Sign-Off Flow (TEAM)

- [ ] **3.1** Write scoping plan or ADR before any code — define the step sequence and compensation logic
- [ ] **3.2** Backbone saga builder — 4 steps:
  1. Validate sign-off (practitioner authorization, visit state check)
  2. Finalize EMR records (consultation step handler)
  3. Billing trigger (cashier participant)
  4. Write canonical visit (ADR-009 `CLINICAL`)
  - **Note:** SATU SEHAT is not a saga step — see §12 for async publish wiring
- [ ] **3.3** Consultation step handler — receive `DoctorSignOff` trigger, finalize prescription + EMR records, callback to backbone
- [ ] **3.4** Wire prescription saga callback (`prescription/service.go:68`) — call backbone `SagaCallbackService` after writing prescription records; this is required before DoctorSignOff can complete
- [ ] **3.5** `canonical_visit_handler.go:85` — `wellmed.events` exchange declaration stays deferred until §12 SATU SEHAT async publish is wired
- [ ] **3.6** Update `plans/visit-saga-wiring/visit-saga-wiring-PROGRESS.md` Phase 5 status when complete

---

# 4. Auth Token Reversal (Alex)

*Pattern: each MS holds one `BACKBONE_API_KEY` (the secret it sends); backbone holds a named key per caller (`BACKBONE_API_KEY_GATEWAY`, etc.) loaded into a map at startup.*

- [ ] **4.1** Backbone: replace single shared token check with `map[token]serviceName` — load from `BACKBONE_API_KEY_GATEWAY`, `BACKBONE_API_KEY_CONSULTATION`, `BACKBONE_API_KEY_PHARMACY`, `BACKBONE_API_KEY_CASHIER`; log `caller=<service_name>` in gRPC interceptor
- [ ] **4.2** Each MS: confirm `BACKBONE_API_KEY` is loaded via `config/env.go` and sent in `x-service-key` header on all outbound backbone calls
- [ ] **4.3** SSM: add `/wellmed/{env}/backbone/BACKBONE_API_KEY_{SERVICE}` entries to backbone manifest; add `/wellmed/{env}/{service}/BACKBONE_API_KEY` to each MS manifest in `wellmed-infrastructure`
- [ ] **4.4** Rotate all tokens after migration — verify each service independently before removing the old shared key

---

# 5. SSM Config Migration (Alex)

- [ ] **5.1** Audit remaining inline `os.Getenv()` calls not yet routed through `config/env.go` — check consultation, pharmacy, cashier (gateway + backbone are partially done)
- [ ] **5.2** Validate `wellmed-infrastructure/ssm/parameters/*.json` manifests against actual values deployed in AWS for each service
- [ ] **5.3** Update `services/gateway.md:25` — module path still shows `ingarso` with "pending update" note

---

# 6. wellmed-appointment (Alex — next week)

*Renamed from `wellmed-scheduler`. Broader scope.*

## 6.1 Scoping Plan

- [ ] Write scoping plan consistent with the consultation/pharmacy extraction pattern
- [ ] Define what moves from backbone vs. what is net-new
- [ ] Assign gRPC port; create SSM manifest placeholder
- [ ] Module: `github.com/kalpa-health/wellmed-appointment`

## 6.2 Domain Scope (Placeholders — Not All Sprint Work)

### 6.2.1 Antrian / Onsite Queue Flow
- Patient check-in queue
- Queue operations: reorder, call, repeat call, retire — all timestamped
- **Start with Kyoo integration hooks** (third-party before building our own antrian UI)
- Reuse the RabbitMQ step-consumer pattern from other MSes for async queue actions

### 6.2.2 Offsite / Direct Book Flow
- Patient-initiated appointment booking
- Confirmation and reminder hooks (future)

### 6.2.3 Doctor Scheduling
- Shift and availability management (currently embedded in backbone `entity/shift.go`)
- Doctor fee sharing logic (later phase)

### 6.2.4 Equipment / Procedure Appointments
- Booking slots for shared equipment and procedure rooms (later phase)

### 6.2.5 Room Booking
- Separate phase — not a typical clinic requirement; defer until hospital/multi-room scope is confirmed

### 6.2.6 CancelVisit — Design Decision
- No "walked-out" as a separate status — cancellation covers it; reason string is required
- On the next visit: surface a prompt "Patient had a cancellation on [date] — flag this visit?" (UI logic, driven by config/patient/backbone)
- Patient registration flagging (warning signs, patient banner, clinic-configurable) is a post-go-live feature — see §14.3

---

# 7. Props DB Operations (Alex — Parallel, Independent of E2E Gate)

- [ ] **7.1** *(was 2.1b)* Drop old expression index in dev, confirm query plan switches to FK path:
  ```sql
  DROP INDEX IF EXISTS emr_2026.idx_vr_props_patient_id;
  EXPLAIN ANALYZE SELECT ... FROM visit_registrations WHERE visit_patient_id = '...';
  ```
- [ ] **7.2** *(was 2.2)* Phase 4 data cleanup — run on staging first, spot-check row counts and sample rows, then prod:
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
- [ ] **7.3** *(was 2.3)* Drop expression index in prod after Phase 4 completes
- [ ] **7.4** *(was 2.4)* Drop `props` columns (`practitioner_evaluations`, `prescriptions`) in staging → prod; then PR to remove `Props` field from Go code in backbone and consultation

---

# 8. Cashier Props Cleanup (Separate Plan Required)

*Analogous to ADR-008 for backbone. Do not implement without a scoping plan.*

- [ ] **8.1** Write scoping plan before any code changes
- [ ] **8.2** Known items the plan must resolve:
  - `grpc/server/invoice.go` — 3 TODOs: parse props JSON → PaymentSummary, PaymentHistory, Author
  - `grpc/server/billing.go` — 2 TODOs: parse props JSON → Author/Cashier info, debt/amount

---

# 9. Consultation Phase 2 Stubs (Post E2E Gate)

*All currently `// TODO: Phase 2` — need backbone saga payload integration. Do not start until §2 passes.*

- [ ] **9.1** `visit_patient/service` — patient_id, card_identity, reference_service from saga payload (not direct DB lookup)
- [ ] **9.2** `treatment/service` — reference_service (Pricing package) + service_price catalog entry via saga payload
- [ ] **9.3** `referral/service` + `visit_registration_referral/service` — item_rent records created as a backbone saga step
- [ ] **9.4** `application.go:182` — add `VisitService` proto and expose `Execute()` via gRPC when the visit flow requires it

---

# 10. Pharmacy Phase 5 Implementation (Post E2E Gate)

*All `// TODO: Phase 5/6` stubs. Do not start until §2 passes.*

- [ ] **10.1** `dispense/service` — PrepareDispense (set PREPARED, auto-confirm stock), ConfirmDispense (set DISPENSED, record dispensed_by, saga callback)
- [ ] **10.2** `medication_request/service` — Create (validate direction/source/items), Approve (set APPROVED), Reject (set REJECTED)
- [ ] **10.3** `pharmacy_sale/service` — CreateSale (linked to dispense), ProcessSale saga step handler (`dispense_requires_payment` flag from trigger payload)
- [ ] **10.4** `prescription/service` — populate from item catalog snapshot in trigger payload; FullyDispensed status + saga callback
- [ ] **10.5** `grpc/server/*.go` — wire all unmarshal/delegate wrappers: prescription, medication_request, pharmacy_sale, dispense
- [ ] **10.6** `clients/consument_patient_client.go` — implement real gRPC call to backbone ConsumentService (currently `nil, nil` stub)
- [ ] **10.7** `clients/saga_callback_client.go` — import sagacallbackpb from backbone proto (currently marked Phase 8)
- [ ] **10.8** Backbone: step order swap in `PharmacySaleSagaBuilder` when `dispense_requires_payment=true`

---

# 11. Proto Contract Typing (Longer-Term, No Sprint Pressure)

*Tracked here to avoid loss. Coordinate across gateway + backbone + consultation.*

- [ ] **11.1** Replace `string data` / `string payload` fields with typed proto messages across the 8 clinical module protos
- [ ] **11.2** Define `BillingParty` typed struct to replace `interface{}` in transaction meta; align across POS and gateway

---

# 12. SATU SEHAT Integration (Async, Separate Track)

*Not in saga. Fire-and-forget async RabbitMQ publish only. No saga dependency.*

- [ ] **12.1** Confirm SATU SEHAT step is absent from DoctorSignOff saga builder (§3.2)
- [ ] **12.2** At visit create: publish async event to `wellmed.events` exchange → SATU SEHAT consumer
- [ ] **12.3** At doctor sign-off: publish async event (fire-and-forget, no callback expected)
- [ ] **12.4** `canonical_visit_handler.go:85` — declare `wellmed.events` exchange at startup as part of this wiring
- [ ] **12.5** Future consideration: service request integration bus — LIS, PACS, and SATU SEHAT as peer subscribers on the same exchange. No other internal triggers (ops actions should not pollute the integration stream)

---

# 13. Saga Backlog — Open Decisions

*Get a decision before building either of these.*

- [ ] **13.1** `UpdatePatient` saga — only build if downstream services cache patient fields. Per ADR-006 they should not. Confirm compliance before scheduling.
- [ ] **13.2** `CancelVisit` saga — decided in §6.2.6: reason string required; next-visit flag prompt; no separate "walked-out" status.

---

# 14. Future Workstreams (Post Go-Live Placeholders)

*Not sprint work. Captured here so decisions and context don't get lost.*

## 14.1 Doctor Assistant — ICD-10 Wiring (Haiku)
- Extend existing Haiku doctor assistant with ICD-10 code support
- Prerequisite to the broader medical coding framework (§14.2)

## 14.2 Medical Coding Framework (AI-Assisted)
- Map clinical activity to: KFA (drugs), LOINC (lab), SNOMED CT (SOAP/plans)
- AI proposes codes; clinician confirms ("is this what you mean?") — doctors are not burdened with manual coding
- Needs scoping: how deep, what validation UX, what is stored vs. inferred

## 14.3 Patient Registration Flagging
- Clinic-configurable warning flags on patient banner (operational, not clinical)
- Config lives in backbone/patient domain
- UI: flag visibility on patient banner, clearance workflow ("I need a senior to clear this")
- First use case: CancelVisit flag prompt (§6.2.6)

## 14.4 Pre-Populated Forms
- Patient/app data → field overlay on PDF → flatten → save to S3
- Timestamped; linked to patient record (not to the immutable canonical JSON)
- Likely a report artifact rather than a clinical record

## 14.5 AI Report Definition Engine (Separate MS)
- Natural language report requests → AI generates report definition using existing Elasticsearch data
- New interaction model for ad-hoc reporting

## 14.6 AI Migration Engine (Separate MS)
- AI-assisted schema and data migration tooling
- Longer-horizon item

---

# 15. Post-Lebaran: CI/CD + Ops Docs

*Deferred intentionally — team on Idul Fitri holiday.*

- [ ] CI/CD hardening (GitHub Actions, ECS deployment pipeline)
- [ ] Integration testing patterns — give this its own document (not buried in the ops TODO stub)
- [ ] `operations/incident-response.md` — on-call rotation, triage steps, escalation path, comms, hotfix process
- [ ] `development/ci-cd-guide.md` — ECS deployment, monitoring, rollback, CloudWatch conventions
- [ ] `development/setup.md` — Go version, tooling, AWS CLI, Docker Compose, env vars, IDE config
- [ ] `development/conventions.md` — proto naming and versioning conventions
- [ ] `operations/database-migrations.md` — tooling, naming, rollback, multi-tenant execution

---

# 16. Doc Cleanup (Quick Wins — Any Time)

- [ ] Delete `services/emr.md` — superseded by `services/consultation.md`, no useful content
- [ ] `services/consultation.md` — fill any remaining TODO sections
- [ ] `development/saga-backlog.md` — update saga statuses as DoctorSignOff progresses

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 06 Mar 2026 | Alex | Created. Supersedes `dev-workflow-PLAN.md`. Full codebase TODO scan consolidated. March sprint context, infrastructure pre-flight, new workstreams (appointment, forms, AI). SATU SEHAT removed from saga (§12). CancelVisit decision (§6.2.6). |

---

# Addendum A: Completed Items

*From `dev-workflow-PLAN.md` v1.1 and service extraction plans.*

## A.1 Gateway Housekeeping
- [x] Fix `kalpa-health` casing in consultation proto `go_package` declarations
- [x] Move `BACKBONE_GRPC_ADDRESS` into `config/env.go`
- [x] Remove debug log from `visit_examination/service/visit_examination.go:32`
- [N/A] Input validation middleware on pass-through routes — reverted; gateway is intentionally thin, gRPC layer validates

## A.2 Backbone Housekeeping
- [x] Drop `props` column from `practitioner_evaluation` — zero rows confirmed, Go field removed
- [x] Drop `props` column from `prescription` — zero rows confirmed, Go field removed
- [x] Audit non-clinical tables with props columns (geographic, access control, product) — dropped where consistently null

## A.3 QUESTIONS.md — All Resolved
- [x] JWT secret naming (`JWT_SESSION_TOKEN_SECRET`, `JWT_CREDENTIAL_TOKEN_SECRET`)
- [x] JWT claims stored in Fiber locals
- [x] gRPC timeout on `user.Login`
- [x] Module path (`ingarso` → `kalpa-health`)
- [x] Dead `generateTokenID` function

## A.4 Microservice Extractions
- [x] wellmed-consultation Phase 1 — repo, proto, 10 gRPC services, saga step scaffold, step handler registered
- [x] wellmed-pharmacy Phase 1 — repo, proto, 4 domains, `CreatePrescriptionHandler` wired
- [x] wellmed-cashier Phase 1 — repo, proto, 4 gRPC services (`Transaction`, `Billing`, `Invoice`, `PaymentSummary`), saga participant scaffold

## A.5 Props Code Changes (ADR-008 Phases 1–3)
- [x] Phase 1: props snapshot removal from service layer (backbone + consultation)
- [x] Phase 2: FK column migration + B-tree index on `visit_registrations.visit_patient_id`
- [x] Phase 3: code references cleaned up across backbone and consultation

## A.6 Visit Saga Wiring (Phases 1–4)
- [x] Backbone `SagaCallbackHandler` wired
- [x] Backbone `CreateVisit` saga builder registered
- [x] Consultation `CreateVisitHandler` registered for trigger + compensate routing keys
- [x] Saga framework complete: orchestrator, continuator, reply store, step tracking, Redis status cache

## A.7 Scheduling MS (Section 5.2 of old plan)
- [x] Renamed to `wellmed-appointment`; scoping moved to §6 of this plan
