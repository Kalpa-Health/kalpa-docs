# Progress Log: wellmed-consultation Service Extraction

**Plan:** `Consultation-extraction-PLAN.md` (v2.0)

---

## Session: 2026-03-04T00:00:00Z

### Phase 0: Pre-flight
- **Status**: ✅ DONE
- **Completed**: 2026-03-04
- **Paths verified**:
  - `/Users/alexknecht/Projects/WellMed/wellmed-backbone` ✅
  - `/Users/alexknecht/Projects/WellMed/wellmed-gateway-go` ✅
  - `kalpa-docs/adrs/ADR-002-consultation-service-extraction.md` ✅
  - `kalpa-docs/adrs/ADR-005-saga-orchestration.md` ✅
  - `kalpa-docs/adrs/ADR-006-domain-service-boundaries.md` ✅
  - `kalpa-docs/development/saga-pattern-extended-PRD.md` ✅
  - `kalpa-docs/services/backbone.md` ✅
  - `kalpa-docs/wellmed-system-architecture.md` ✅
- **Plan status**: "Ready to execute — updated to incorporate saga architecture decisions" ✅
- **Issues**: None

---

### Step 1.1: Pre-extraction Audit
- **Status**: ✅ DONE (pending human checkpoint review)
- **Completed**: 2026-03-04
- **Files modified**: None (audit only)

---

#### 1.1a: Cross-module import audit

All 13 consultation modules were grepped for imports from backbone-only modules
(module path `github.com/ingarso/wellmed-backbone`).

**Backbone-only module dependencies found:**

| Consultation Module | Backbone-Only Dep | Import Type | Phase 2 Resolution |
|---|---|---|---|
| visit_patient | `patient` | repository | Backbone composes `patient_id` in saga trigger |
| visit_patient | `card_identity` | repository | Backbone composes card identity data in saga trigger |
| visit_patient | `reference_service` | repository | Backbone composes reference data in saga trigger |
| visit_patient | `unicode` | repository | Backbone composes unicode/locale data in saga trigger |
| visit_registration | `unicode` | repository | Backbone composes unicode/locale data in saga trigger |
| visit_registration_referral | `unicode` | repository | Backbone composes unicode/locale data in saga trigger |
| visit_registration_referral | `reference_service` | repository | Backbone composes reference data in saga trigger |
| visit_examination | `unicode` | repository | Backbone composes unicode/locale data in saga trigger |
| treatment | `unicode` | repository | Backbone composes unicode/locale data in saga trigger |
| treatment | `reference_service` | repository | Backbone composes reference data in saga trigger |
| referral | `unicode` | repository | Backbone composes unicode/locale data in saga trigger |
| frontline | `unicode` | repository | Backbone composes unicode/locale data in saga trigger |

**All the above are Phase 2 items.** Per ADR-006 §2.4, no gRPC clients are added
for these. Each one gets a `// TODO: Phase 2 — received via Backbone saga payload` comment
when the module is moved in Step 1.5. All findings logged here as inputs for Phase 2
Backbone saga payload design.

**Consultation-internal deps (healthy — all move together):**

| Module | Imports (consultation-internal) |
|---|---|
| visit_patient | assessment, family_relationship, practitioner_evaluation, referral, visit_examination, visit_registration |
| visit_registration | visit_examination |
| visit_registration_referral | assessment, practitioner_evaluation, referral, visit_examination, visit_registration |
| visit_examination | assessment, visit_patient, visit_registration |
| treatment | medical_service_treatment |
| referral | practitioner_evaluation, visit_patient, visit_registration |
| frontline | assessment, practitioner_evaluation, visit_examination, visit_registration |
| assessment | visit_examination |

**⚠️ Unclassified external dependencies (not in backbone-only or consultation lists):**

| Module | Unclassified Dep | Also used by | Decision |
|---|---|---|---|
| referral | `item_rent` | `patient` (backbone-only) | **Stays in Backbone** — shared with patient module. Referral gets item_rent data via Backbone saga payload (Phase 2). |
| visit_registration_referral | `item_rent` | `patient` (backbone-only) | **Stays in Backbone** — same as above. |
| visit_patient | `item_rent` | `patient` (backbone-only) | **Stays in Backbone** — same as above. |
| treatment | `service_price` | treatment only | **Moves to Consultation** — only used by consultation modules. |

`item_rent` stays in Backbone because it is also imported by `patient` (a backbone-only
module). Moving it would violate ADR-006. The three consultation modules that use
`item_rent` will receive this data via Backbone saga payloads in Phase 2.

`service_price` moves with `treatment` to Consultation — it has no backbone-only dependents.
**Add `service_price` to the modules-moving list.**

**No consultation module uses `internal/apiclient/` or `internal/integration/`.** Skip copying `apiclient`.

---

#### 1.1b: Saga steps audit

The saga framework lives at `internal/saga/` in Backbone. Files:
`context.go`, `continuator.go`, `model.go`, `orchestrator.go`, `repository.go`, `step.go`
plus `infrastructure/` (event_consumer.go, migrations, postgres_repository.go,
rabbitmq_publisher.go, reply_store.go, setup.go, tenant_repository.go).

**Saga framework references to consultation modules: NONE.** The saga framework is
fully generic and does not import any domain module by name. Saga steps that
orchestrate consultation operations are defined externally (in the saga builders
that call domain services).

This confirms: the saga framework is infrastructure, not domain logic. Per ADR-005
it stays entirely in Backbone. No saga code is copied to Consultation.

---

#### 1.1c: Shared utilities audit

All 13 consultation modules use the following shared infrastructure from Backbone.
These patterns must be replicated (not imported from Backbone) in Consultation:

| Package | What it provides | Action |
|---|---|---|
| `internal/db/config/` | ConnectionManager, ConnectionOrchestrator, TransactionManager | **Replicate verbatim** in Step 1.3 |
| `internal/helper/` | GenerateUlid, db helpers, paginate, s3, virtual_column | **Replicate verbatim** in Step 1.3 |
| `internal/schema/entity/emr/` | EMR-specific Go structs (clinical data models) | **Copy verbatim** — these are data shapes, not logic |
| `internal/schema/entity/` | Base entity structs | **Copy verbatim** |
| `internal/schema/metadata/` | Metadata structures | **Copy verbatim** |
| `internal/validation/` | Validation rules | **Copy verbatim** |
| `internal/saga/` | Saga orchestration framework | **DO NOT COPY** — ADR-005. Backbone is sole orchestrator. |
| `internal/apiclient/` | External API clients | **SKIP** — no consultation module uses it |
| `internal/grpc/` | gRPC server bootstrap | **Replicate** — needed for gRPC server setup |

---

#### 1.1d: data_sync special audit — GATE DECISION

**Finding:** `internal/domain/data_sync/saga/builder.go` EXISTS.

Content: `DataSyncSagaBuilder` with `BuildAsync()` method that constructs
`[]*saga.SagaStep` using the Backbone saga framework (`internal/saga`).

**Decision: `data_sync` STAYS IN BACKBONE.**

Criterion met: the module contains a saga builder (ADR-005 gate). Modules with saga
ownership are not extracted to Consultation.

**Impact on plan:**
- Remove `data_sync` from the modules-moving list
- Module count: **13 → 12**
- Update module lists wherever "13" appears: Steps 1.4, 1.5, 1.7, and the Overview section
- `data_sync` is NOT removed from Backbone in Step 1.6

Updated modules moving to Consultation (12):
`visit_patient`, `visit_registration`, `visit_registration_referral`, `visit_examination`,
`assessment`, `treatment`, `examination_treatment`, `medical_service_treatment`,
`referral`, `practitioner_evaluation`, `frontline`, `family_relationship`

Added to modules moving: `service_price` (moves with treatment)

---

### ⏸️ HUMAN CHECKPOINT — Step 1.1 Complete

**Status**: ⏸️ WAITING_HUMAN
**Checkpoint**: End of Step 1.1 Pre-extraction Audit

---

## What to Review Before Continuing

### A. data_sync placement decision ✅ (clear)
`data_sync` stays in Backbone. Its `saga/builder.go` contains saga orchestration
logic — per ADR-005, saga builders belong in Backbone. **No action needed — this
decision is unambiguous.**

### B. item_rent placement decision ✅ (clear)
`item_rent` stays in Backbone. It is used by both `patient` (backbone-only) and
several consultation modules. Moving it to Consultation would violate ADR-006
(modules cannot call each other directly). The three consultation modules
(`referral`, `visit_registration_referral`, `visit_patient`) that use `item_rent`
will receive this data via Backbone saga payload composition in Phase 2.
**No action needed — this decision is unambiguous per ADR-006.**

### C. service_price placement ⚠️ (confirm)
`service_price` is only used by `treatment` (consultation). The agent recommends
moving it to Consultation alongside `treatment`. Please confirm:
- [ ] **CONFIRM**: `service_price` moves to Consultation (add to modules-moving list)
- [ ] **OVERRIDE**: `service_price` stays in Backbone (if there are billing/invoicing
  reasons not visible from grep — e.g., POS/billing modules also use it)

### D. Module count update ✅ (clear once C is confirmed)
Plan references "13 modules" throughout. After data_sync stays in Backbone,
count drops to 12. After confirming service_price, update the plan header and
module list. **Agent will update the plan file once you confirm C.**

### E. Backbone-only deps — Unicode
`unicode` is used by 6 of the 12 moving modules. It provides locale/character
encoding data. Confirm: Is this data available without a live Backbone call, or
must it come via saga payload?
- [ ] **CONFIRM**: unicode data comes via Backbone saga payload (consistent with ADR-006)
- [ ] **OVERRIDE**: unicode lookup is stateless and consultation can call a Backbone
  gRPC endpoint directly (ONLY if a specific ADR-006 exception is acceptable)

### F. Additional modules not in original lists
Two modules appear in the codebase but were not in the plan's module lists:
`item_rent` (covered in B above) and `service_price` (covered in C above).
Are there any other modules in `internal/domain/` that should be reviewed?

> Full domain module list in Backbone:
> address, assessment, autolist, billing, card_identity, card_stock, consumer,
> data_sync, employee, examination_treatment, family_relationship, frontline,
> invoice, item, item_rent, medical_service_treatment, medicine, menu, patient,
> people, permission, pharmacy, pharmacy_sale, practitioner_evaluation, reference_service,
> referral, role, service_price, tenant, transaction, treatment, unicode, user,
> visit_examination, visit_patient, visit_registration, visit_registration_referral

Modules NOT in either list (staying in Backbone implicitly):
`billing`, `card_stock`, `invoice`, `item`, `item_rent`, `medicine`, `people`,
`pharmacy`, `pharmacy_sale`, `service_price` (pending C), `transaction`

---

## Human Checkpoint Resolved — 2026-03-04

All decisions confirmed. Plan updated to v2.1. Summary of confirmed resolutions:

| Question | Decision |
|---|---|
| `data_sync` placement | **Stays in Backbone** — saga builder confirmed |
| `item_rent` placement | **Stays in Backbone** — shared with `patient` module |
| `service_price` placement | **Stays in Backbone** — Pricing package, future extraction target. Consultation references item IDs only. |
| `unicode` resolution | **Consultation carries read-only `UnicodeRepository`** (direct tenant DB read). Phase 2 saga payloads include pre-composed unicode data. |
| Module count | **12** (not 13) |

Plan files updated: both `Consultation-extraction-PLAN.md` and `2-Consultation-extration-PLAN.md` are v2.1.

---

### ⏸️ HUMAN CHECKPOINT — Step 1.2 (Bootstrap)

**Status**: ⏸️ WAITING_HUMAN

Step 1.2 requires human action — running the bootstrap script to create the GitHub
repo and CI pipeline. This cannot be automated.

**Actions required:**

```bash
# 1. Run bootstrap script
cd ~/Projects/WellMed/wellmed-infrastructure/scripts
./bootstrap-repo.sh Kalpa-health/wellmed-consultation \
  --binary-name wellmed-consultation \
  --cmd-path ./cmd

# 2. Verify go.mod module path is set to:
#    github.com/Kalpa-Health/wellmed-consultation

# 3. Add Claude Code API key to new repo
gh secret set ANTHROPIC_API_KEY -r Kalpa-health/wellmed-consultation

# 4. Update .github/workflows/pr-review.yml — fill in the TODO:ARCH section
#    with: "Go microservice owning the clinical visit lifecycle. Extracted from
#    wellmed-backbone. 12 gRPC services. Participant in Backbone-orchestrated sagas."
```

**Resume:** When the repo is bootstrapped, say **"continue the consultation extraction plan"**
and Claude will proceed to Step 1.3 (scaffold).

---

## Session: 2026-03-05T00:00:00Z

### Step 1.2: Bootstrap
- **Status**: ✅ DONE (completed in prior session — repo already bootstrapped)
- **Completed**: 2026-03-04
- **What was done**: GitHub repo `Kalpa-Health/wellmed-consultation` already had main/develop/staging branches and CI workflows in place from prior session. Cloned repo. Updated `.github/workflows/pr-review.yml` with full architecture context (removed TODO:ARCH placeholder).
- **Files modified**: `wellmed-consultation/.github/workflows/pr-review.yml`
- **Issues**: None

---

### Step 1.3: Scaffold the Consultation Service
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**:
  - Copied infrastructure packages from Backbone with module path replacement (`github.com/ingarso/wellmed-backbone` → `github.com/Kalpa-Health/wellmed-consultation`):
    - `internal/db/` — ConnectionManager, ConnectionOrchestrator, TransactionManager, SchemaRegistry
    - `internal/helper/` — GenerateUlid, db helpers, paginate, s3, virtual_column
    - `internal/schema/` — entity/, entity/emr/, metadata/
    - `internal/validation/`
    - `internal/base/`
    - `internal/config/` — env.go, redis.go, crypto.go (rpc_client_manager.go deleted — Backbone-only gRPC clients violate ADR-006)
    - `internal/middleware/`
    - `internal/infrastructure/grpc/`
    - `internal/tenant/` — tenant context resolver (pure infrastructure, stdlib only)
    - `cmd/main.go`
  - Created `go.mod` with module `github.com/Kalpa-Health/wellmed-consultation`, go 1.25.6, full dependency list
  - Rewrote `internal/app/application.go` — clean scaffold: ConnectionManager, Redis, gRPC server on :50052, RabbitMQ connection, StepConsumer stub wiring
  - Deleted `internal/app/integration.go` (Backbone-specific IntegrationService, irrelevant to Consultation)
  - Deleted `internal/grpc/server/*.go` — all domain handler files; these reference domain services not yet moved (Step 1.5 work)
  - Deleted `internal/grpc/client/*.go` — billing/invoice/transaction gRPC clients; violate ADR-006 (Backbone-only services)
  - Created `internal/queue/consumer.go` — RabbitMQ StepConsumer stub with StepHandler interface (Handle/Compensate); all Phase 2 handlers log and ack without processing
  - Created `internal/clients/saga_callback_client.go` — SagaCallbackService gRPC client stub with ReportStepResult → ErrNotImplemented (Phase 2)
  - Created `internal/clients/canonical_visit_client.go` — CanonicalVisitService gRPC client stub with WriteVisitRecord → ErrNotImplemented (Phase 2); comments document ADR-006 exception and data boundary
  - Copied `internal/domain/unicode/repository/unicode.go` — read-only unicode repository (direct tenant DB read, no gRPC per ADR-006)
  - Created `env.example` with all environment variables
  - Ran `go mod tidy` and `go build ./...` — **clean build, zero errors**
- **Files created**:
  - `wellmed-consultation/go.mod`
  - `wellmed-consultation/env.example`
  - `wellmed-consultation/internal/app/application.go` (rewritten)
  - `wellmed-consultation/internal/queue/consumer.go`
  - `wellmed-consultation/internal/clients/saga_callback_client.go`
  - `wellmed-consultation/internal/clients/canonical_visit_client.go`
  - `wellmed-consultation/internal/domain/unicode/repository/unicode.go`
  - `wellmed-consultation/internal/tenant/tenant_resolver.go`
  - All infrastructure packages under `internal/db/`, `internal/helper/`, `internal/schema/`, `internal/validation/`, `internal/base/`, `internal/config/`, `internal/middleware/`, `internal/infrastructure/`
- **Files deleted**: `internal/app/integration.go`, `internal/grpc/server/*.go` (21 files), `internal/grpc/client/*.go` (4 files), `internal/config/rpc_client_manager.go`
- **Issues**: None — build clean

---

### Step 1.4: Define Consultation Proto Interface
- **Status**: ⏸️ WAITING_HUMAN (CHECKPOINT — review proto interface definitions)
- **Completed (pre-checkpoint)**: 2026-03-05
- **What was done**:
  - Audited Backbone proto files: 8 of the 12 moving modules have standalone gRPC services. 4 sub-modules (family_relationship, examination_treatment, medical_service_treatment, practitioner_evaluation) have no gRPC service of their own — they are accessed through parent module handlers.
  - Copied 8 proto files from Backbone to `wellmed-consultation/proto/`, updated `go_package` to `github.com/Kalpa-Health/wellmed-consultation/proto/{module}pb`:
    - `assessment.proto` — AssessmentService: Index
    - `frontline.proto` — FrontlineService: Index, Show, Store, Dispense
    - `referral.proto` — ReferralService: Index, Show, Store
    - `treatment.proto` — TreatmentService: Index, Show, Store
    - `visit_examination.proto` — VisitExaminationService: Index, Show, Create, Update
    - `visit_patient.proto` — VisitPatientService: Index, Show, Store
    - `visit_registration.proto` — VisitRegistrationService: Index, ShowDetail
    - `visit_registration_referral.proto` — VisitRegistrationReferralService: Index, Show, Store
  - Note: proto files use per-file convention matching Backbone. Plan mentioned single file but per-file is the established pattern and easier for Gateway to verify.
  - gRPC stubs NOT yet generated — waiting for human approval of interface definitions.
- **Files created**: `wellmed-consultation/proto/*.proto` (8 files)
- **Issues**: None

---

### Step 1.4: Define Consultation Proto Interface — STUBS GENERATED
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**: Human approved proto interface. Generated gRPC stubs for all 8 proto files using protoc. Moved generated `.pb.go` files to their correct `proto/{module}pb/` subdirectories. `go build ./...` clean.
- **Files created**: `proto/*.proto` (8), `proto/{module}pb/*.pb.go` and `*_grpc.pb.go` (16 generated files)
- **Issues**: Generated files initially landed in wrong directory (paths=source_relative quirk) — fixed by moving to subdirs.

---

### Step 1.5: Move the 12 Modules
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**:
  - Copied all 12 domain modules from Backbone to Consultation in dependency order. Replaced all `github.com/ingarso/wellmed-backbone` imports with `github.com/Kalpa-Health/wellmed-consultation`.
  - Backbone-only dep removals (per ADR-006):
    - **`treatment`**: removed `reference_service` + `service_price` (Pricing package, stays Backbone); stubbed `Service` catalog entry and service price creation with `TODO: Phase 2` comments
    - **`referral`**: removed `item_rent` (shared with patient module); stubbed item rent creation with `TODO: Phase 2`
    - **`visit_registration_referral`**: removed `item_rent` + `reference_service`; stubbed item rent creation with `TODO: Phase 2`
    - **`visit_patient`**: removed `patient`, `card_identity`, `item_rent`, `reference_service`; `patient` stub uses `PatientID`-only placeholder, reference_service treatment catalog stubbed with empty Morph field
  - Copied 8 gRPC handler files to `internal/grpc/server/` (assessment, frontline, referral, treatment, visit_examination, visit_patient, visit_registration, visit_registration_referral)
  - Registered all 8 services in `internal/app/application.go` with `configDB.NewTransactionManager`
  - Added `gjson` to `go.mod` (used by assessment processors)
  - Copied `internal/tenant/` package (used by db/config)
  - Removed `internal/config/rpc_client_manager.go` (billing/invoice/transaction gRPC clients — ADR-006 violation)
  - `go mod tidy` and `go build ./...` — **clean build, zero errors**
- **Modules moved** (12): `family_relationship`, `referral`, `assessment`, `treatment`, `examination_treatment`, `medical_service_treatment`, `practitioner_evaluation`, `visit_examination`, `frontline`, `visit_registration_referral`, `visit_registration`, `visit_patient`
- **Sub-modules without gRPC service** (4, accessed through parent handlers): `family_relationship`, `examination_treatment`, `medical_service_treatment`, `practitioner_evaluation`
- **gRPC services registered** (8): AssessmentService, FrontlineService, ReferralService, TreatmentService, VisitExaminationService, VisitPatientService, VisitRegistrationService, VisitRegistrationReferralService
- **Issues**: None — clean build

---

### ⏸️ HUMAN CHECKPOINT — Step 1.5 Smoke Test

**Status**: ⏸️ WAITING_HUMAN

`wellmed-consultation` compiles cleanly with all 12 domain modules and 8 gRPC services registered. Verify before proceeding to Step 1.6 (update Backbone).

**What to verify:**
1. `go build ./...` passes (already confirmed above)
2. Start the service locally with a valid `.env` and confirm gRPC server binds on `:50052`
3. Optional: `grpcurl -plaintext localhost:50052 list` — confirm 8 services appear in reflection
4. Confirm backbone-only dep stubs are acceptable for Phase 1 (patient/item_rent/reference_service/card_identity all have `TODO: Phase 2` markers)

**Resume:** When smoke test passes, say **"continue the consultation extraction plan"** to proceed to Step 1.6 (update Backbone — delete moved modules, register CanonicalVisitService stub).

---

### ⏸️ HUMAN CHECKPOINT — Step 1.4 Proto Interface Review

**Status**: ⏸️ WAITING_HUMAN

Review the 8 proto files in `wellmed-consultation/proto/` before gRPC stubs are generated.

**What to verify:**
1. Service names and method signatures match what the Gateway currently routes to Backbone for these 12 modules. Compare against `wellmed-gateway-go` routing config.
2. Request/response message types are complete — no missing fields that the Gateway sends.
3. The 4 modules without services (family_relationship, examination_treatment, medical_service_treatment, practitioner_evaluation) are correct — they have no standalone gRPC endpoint.
4. `assessment.proto` only has `Index` — confirm there is no `Store`/`Update` for assessments.
5. `visit_registration.proto` only has `Index` and `ShowDetail` — confirm no other methods.

**Resume:** When proto interface is approved, say **"continue the consultation extraction plan"** and Claude will generate the gRPC stubs and proceed to Step 1.5.

---

*Progress log — do not edit manually. Append-only.*

---

## Session: 2026-03-05T07:30:00Z

### Step 1.5 Smoke Test
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**: Installed Redis via Homebrew. Ran service with inline env vars. Verified gRPC server starts on :50052. Used grpcurl to confirm all 8 services registered. Also added `reflection.Register` and fixed `visit_patient.proto` package name (`user` → `visitpatient`).
- **Files modified**:
  - `wellmed-consultation/internal/app/application.go` — added `reflection.Register`
  - `wellmed-consultation/proto/visit_patient.proto` — fixed package name
  - `wellmed-consultation/proto/visitpatientpb/visit_patient.pb.go` — regenerated
  - `wellmed-consultation/proto/visitpatientpb/visit_patient_grpc.pb.go` — regenerated
- **grpcurl output**:
  - assessment.AssessmentService ✅
  - frontline.FrontlineService ✅
  - referral.ReferralService ✅
  - treatment.TreatmentService ✅
  - visitexamination.VisitExaminationService ✅
  - visitpatient.VisitPatientService ✅
  - visitregistration.VisitRegistrationService ✅
  - visitregistrationreferral.VisitRegistrationReferralService ✅
- **Issues**: None

### Step 1.6: Update Backbone
- **Status**: ✅ DONE (with caveats — see below)
- **Completed**: 2026-03-05
- **What was done**:
  1. Removed 8 gRPC service registrations from `wellmed-backbone/internal/app/application.go`: assessment, frontline, referral, treatment, visit_examination, visit_patient, visit_registration, visit_registration_referral.
  2. Removed their module instantiations from application.go.
  3. Removed their domain module imports and proto pb imports from application.go.
  4. **Did NOT delete domain source directories** — patient/pharmacy_sale saga builders directly import repos from these modules. Deleting them would break the patient saga. Phase 2 refactors saga builders to call Consultation via gRPC, then source can be deleted.
  5. Generated `proto/canonicalvisitpb/` stubs from `proto/canonical_visit.proto`.
  6. Created `internal/domain/canonical_visit/handler/canonical_visit_handler.go` — Phase 1 stub returning Unimplemented.
  7. Registered `CanonicalVisitService` in application.go.
  8. Added `ConsultationGRPCAddress` / `CONSULTATION_GRPC_ADDRESS` to `internal/config/env.go`.
  9. Fixed pre-existing bug: `tenant.Schema` → `tenant.TenantContext` in `internal/saga/orchestrator.go`.
- **Files modified**:
  - `wellmed-backbone/internal/app/application.go`
  - `wellmed-backbone/internal/config/env.go`
  - `wellmed-backbone/internal/saga/orchestrator.go` (pre-existing typo fix)
  - `wellmed-backbone/proto/canonicalvisitpb/canonical_visit.pb.go` (generated)
  - `wellmed-backbone/proto/canonicalvisitpb/canonical_visit_grpc.pb.go` (generated)
  - `wellmed-backbone/internal/domain/canonical_visit/handler/canonical_visit_handler.go` (created)
- **Backbone compile status**: ❌ DOES NOT COMPILE — pre-existing errors in `internal/saga/` files (step.go `.Event` method, infrastructure/ interface mismatches). These are the in-progress saga-pattern-extended plan changes. Our packages (app, canonical_visit, canonicalvisitpb, config) build cleanly in isolation.
- **Issues**: Backbone `go build ./...` blocked by saga-pattern-extended plan's in-progress work. The saga plan (at `kalpa-docs/plans/saga-pattern-extended/`) must complete Phase 2 (Tasks 2.1–2.11) before backbone will compile.

### ⏸️ HUMAN CHECKPOINT — Step 1.6 Backbone Verify

**Status**: ⏸️ WAITING_HUMAN

**Blocker identified:** Backbone does not compile because `internal/saga/` is mid-refactor as part of the `saga-pattern-extended-PLAN`. The saga plan's Phase 0 is done but Phases 1–2 are not yet started.

**Required before proceeding:**
1. **Run the saga plan**: `saga-pattern-extended-PLAN.md` Phase 2 (Tasks 2.1–2.11) must complete. This fixes the saga framework API (`StepBuilder.Event`, `ReplyStore.Store`, etc.) so backbone compiles.
2. **Verify backbone compiles** after saga plan: `cd wellmed-backbone && go build ./...`
3. **Confirm 8 services removed from Backbone's gRPC catalog** (run grpcurl against backbone).

**Resume options:**
- To continue consultation extraction: say "continue the consultation extraction plan" (will proceed to Step 1.7 Gateway update).
- To run the saga plan first: say "run the saga-pattern-extended plan".
- Recommended order: **run saga plan first**, then continue consultation extraction. Both are required for production.

---

*Progress log — do not edit manually. Append-only.*

---

## Session: 2026-03-05T08:00:00Z

### Step 1.6 Backbone Verify (resumed)
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**: Saga framework rework completed externally (saga-pattern-extended plan). Backbone now compiles clean (`go build ./... ` passes). CanonicalVisitService registered. 8 consultation services removed from backbone gRPC catalog.

### Step 1.7: Update Gateway
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**:
  1. `CONSULTATION_GRPC_ADDRESS` already in `internal/config/env.go` (set as package var, read from env).
  2. Second `grpcx.Manager` (`consultationManager`) created in `internal/app/app.go`, pointing to `CONSULTATION_GRPC_ADDRESS` (default `localhost:50052`).
  3. Both managers added to `cleanup` func.
  4. 8 RPC clients re-routed from `rpcManager` to `consultationManager`:
     - `visitPatientRPC` → consultationManager
     - `visitRegistrationRPC` → consultationManager
     - `visitRegistrationReferralRPC` → consultationManager
     - `visitExaminationRPC` → consultationManager
     - `assessmentRPC` → consultationManager
     - `referralRPC` → consultationManager
     - `frontlineRPC` → consultationManager
     - `medicalTreatmentRPC` (= treatment service) → consultationManager
  5. All backbone-only services confirmed still on `rpcManager`.
  6. `go build ./...` passes.
- **Files modified**:
  - `wellmed-gateway-go/internal/app/app.go`
  - `wellmed-gateway-go/internal/config/env.go` (ConsultationGRPCAddress added)
- **Issues**: None

### ⏸️ HUMAN CHECKPOINT — Step 1.7 Gateway Verify

**Status**: ⏸️ WAITING_HUMAN

**Review:**
1. Backbone client (`rpcManager`) routes: user, role, navigation, patient, pharmacySale, transaction, billing, invoice, item, autolist, and all master-data modules.
2. Consultation client (`consultationManager`) routes: visitPatient, visitRegistration, visitRegistrationReferral, visitExamination, assessment, referral, frontline, medicalTreatment (= treatment).
3. `CONSULTATION_GRPC_ADDRESS` must be set in all deployment environments (dev, staging, prod) before deploy.

**Resume:** Say "continue the consultation extraction plan" to proceed to Step 1.8 (AWS infrastructure).

---

*Progress log — do not edit manually. Append-only.*

---

## Session: 2026-03-05T (Step 1.10 docs + Step 1.11 testing gate)

### Step 1.10: Documentation Updates
- **Status**: ✅ DONE
- **Completed**: 2026-03-05
- **What was done**: Background agent created/updated 5 documentation files covering the ADR-002 Phase 1 extraction.
- **Files modified**:
  - `kalpa-docs/services/consultation.md` — new file, full service doc (8 sections)
  - `kalpa-docs/services/backbone.md` — §4.2 updated from "pending" to "extracted Phase 1 complete", count 22→14, v1.2
  - `kalpa-docs/wellmed-system-architecture.md` — EMR→Consultation throughout, v1.3
  - `kalpa-docs/README.md` — EMR row replaced with Consultation row + wellmed-consultation repo link
  - `kalpa-docs/repo-governance.md` — wellmed-consultation added, v1.3
- **Issues**: None

### Step 1.11: Testing Gate
- **Status**: ✅ DONE (with documented constraint)
- **Completed**: 2026-03-05
- **What was done**: Wrote service-layer unit tests for all 8 domain modules + infrastructure packages. 12/12 test packages pass.

  **Test files created:**
  - `internal/clients/clients_test.go` — SagaCallbackClient + CanonicalVisitClient stub tests
  - `internal/queue/consumer_test.go` — StepConsumer registration, routing, dispatch (ack/nack)
  - `internal/domain/assessment/service/assessment_test.go` — GetByExamination (singular/plural morphs)
  - `internal/domain/treatment/service/treatment_test.go` — GetByFlag, GetById, interface compliance
  - `internal/domain/referral/service/referral_test.go` — GetAll, GetById, PrepareStore (pure fn)
  - `internal/domain/frontline/service/frontline_test.go` — GetAll, GetById, interface compliance
  - `internal/domain/visit_examination/service/visit_examination_test.go` — PrepareStore, GetAll, GetById
  - `internal/domain/visit_patient/service/visit_patient_test.go` — PrepareStore (unicode lookup), GetAll, GetById
  - `internal/domain/visit_registration/service/visit_registration_test.go` — PrepareStore (ULID gen, status), GetAll, GetById (100% coverage)
  - `internal/domain/visit_registration_referral/service/visit_registration_referral_test.go` — GetAll, GetById

  **Coverage note:** `visit_registration` service: 100%. Other services: read paths and pure functions fully covered. `Store`/`Dispense`/`Create` methods in all services call `txManager.TenantTransaction/Transaction` as the first operation. `TransactionManager` is a concrete struct (not an interface) and requires a real PostgreSQL connection. These methods are untestable at the unit-test layer without a real DB. Integration tests covering the Store paths are a Phase 2 backlog item.

  **`ErrNotImplemented` fix:** Changed from `errors.New(...)` to a custom `errNotImplemented` type so sentinel identity works correctly via `errors.Is`.

- **Files modified**: (test files listed above)
- **Issues**: Store/transactional methods in all services (except visit_registration which has no Store) require real DB for coverage — documented as integration test backlog.

### ⏸️ HUMAN CHECKPOINT — Phase 1 Complete

**Status**: ⏸️ WAITING_HUMAN

**Phase 1 is functionally complete.** Review:
1. All 8 gRPC services extracted and registered at `:50052` (verified via grpcurl smoke test)
2. Backbone: 8 service registrations removed. CanonicalVisitService stub registered.
3. Gateway: all 8 consultation RPCs routed to `consultationManager` on `CONSULTATION_GRPC_ADDRESS`
4. Docs: consultation.md created, backbone.md + system-arch + README + repo-governance updated
5. Tests: 12/12 packages pass. All read paths and pure functions unit tested. Store methods (transaction-heavy, require real DB) noted as integration test backlog.

**Remaining TODOs (not blocking Phase 1 merge):**
- Step 1.8: SSM Parameter Store — add `CONSULTATION_GRPC_ADDRESS` to `/wellmed/{env}/go/gateway/CONSULTATION_GRPC_ADDRESS`
- Step 1.8: ECS task definition for wellmed-consultation container
- Step 1.9: Deploy to dev, end-to-end smoke test through gateway
- Integration tests for Store/transactional paths (Phase 2 backlog)

**Resume:** Say "continue the consultation extraction plan" to proceed to deployment steps.

---

*Progress log — do not edit manually. Append-only.*
