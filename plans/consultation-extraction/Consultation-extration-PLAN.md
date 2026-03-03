# Plan: wellmed-consultation Service Extraction

**Version:** 1.0
**Date:** 03 March 2026
**Author:** Alex
**ADR:** [`adrs/ADR-002-consultation-service-extraction.md`](../../adrs/ADR-002-consultation-service-extraction.md)
**Status:** Ready to execute

---

## Overview

Extract 13 clinical consultation modules from `wellmed-backbone` into a new standalone microservice `wellmed-consultation`. After Phase 1, both services run independently. Phase 2 (saga wiring) is planned separately using Phase 1 as operational context.

**Repos involved:**
- `wellmed-backbone` — source of modules, will be modified
- `wellmed-gateway-go` — routing update required
- `wellmed-consultation` — new repo (created in this plan)
- `kalpa-docs` — documentation updates

**Local paths:**
- Backbone: `/Users/alexknecht/Projects/WellMed/wellmed-backbone`
- Gateway: `/Users/alexknecht/Projects/WellMed/wellmed-gateway-go`
- Docs: `/Users/alexknecht/Projects/WellMed/kalpa-docs`

---

## Modules Moving to wellmed-consultation

`visit_patient`, `visit_registration`, `visit_registration_referral`, `visit_examination`, `assessment`, `treatment`, `examination_treatment`, `medical_service_treatment`, `referral`, `practitioner_evaluation`, `frontline`, `data_sync`, `family_relationship`

## Modules Staying in Backbone

`patient`, `tenant`, `user`, `employee`, `role`, `permission`, `menu`, `unicode`, `autolist`, `address`, `reference_service`, `consumer`, `card_identity`

---

## Phase 1: Structural Extraction

### 1.1 Pre-extraction Audit

- [ ] **Audit cross-module imports in the 13 Consultation modules.** For each module being moved, grep for any imports of Backbone-only modules (e.g., `patient`, `tenant`, `user`). Document every discovered dependency. Any module that calls into a non-Consultation module needs a gRPC client in Consultation to reach it after extraction.
  ```
  grep -rn "backbone\|/patient\|/tenant\|/user" internal/domain/visit_patient/ --include="*.go"
  ```
  Repeat for each of the 13 modules. Log findings before proceeding.

- [ ] **Audit saga steps that touch Consultation modules.** In `internal/saga/`, identify every saga that references a Consultation module. Document: step name, what it does, what compensation it requires. This is the input for Phase 2.

- [ ] **Note any shared utilities used by Consultation modules** (DB connection manager, Redis client, config loader). These patterns need to be replicated in the new service.

> **HUMAN CHECKPOINT:** Review audit findings. Confirm no blocking dependencies were missed before creating the new repo and starting the move.

---

### 1.2 Bootstrap wellmed-consultation Repo

- [ ] **Run bootstrap script** from `wellmed-infrastructure`:
  ```bash
  cd ~/Projects/WellMed/infrastructure/scripts
  ./bootstrap-repo.sh Kalpa-health/wellmed-consultation \
    --binary-name wellmed-consultation \
    --cmd-path ./cmd
  ```

- [ ] **Update go.mod** module path: `github.com/Kalpa-Health/wellmed-consultation`

- [ ] **Add ANTHROPIC_API_KEY secret** to the new repo:
  ```bash
  gh secret set ANTHROPIC_API_KEY -r Kalpa-health/wellmed-consultation
  ```

- [ ] **Update `.github/workflows/pr-review.yml`** — fill in the `TODO:ARCH` section with a description of the Consultation service.

---

### 1.3 Scaffold the Consultation Service

- [ ] **Copy the service skeleton** from Backbone: `cmd/`, `internal/app/`, `config/`, `.env.example`. Rename binary references from `wellmed-backbone` to `wellmed-consultation`.

- [ ] **Copy DB connection manager** (`internal/db/` or equivalent) verbatim. Consultation needs its own `ConnectionManager` for multi-tenant PostgreSQL access. Do not import Backbone as a dependency.

- [ ] **Copy Redis client setup** verbatim.

- [ ] **Copy the saga framework** from Backbone's `internal/saga/` into Consultation's `internal/saga/`. This is a pattern copy — the two copies evolve independently until a shared module is extracted.

- [ ] **Copy `pkg/apiclient/`** from Backbone into Consultation if any Consultation module needs it (check audit findings from 1.1).

- [ ] **Set gRPC port.** Consultation runs on `:50052`. Update `cmd/main.go` and `.env.example`. Document in `services/consultation.md` after Phase 1.

- [ ] **Verify the service compiles with no modules yet:**
  ```bash
  cd ~/Projects/WellMed/wellmed-consultation
  go build ./...
  ```

---

### 1.4 Define Consultation Proto Interface

- [ ] **Create proto file** `proto/consultation/consultation.proto`. Define gRPC services for all 13 modules. Mirror the method signatures that currently exist in Backbone's proto for these modules. Example structure:
  ```protobuf
  service VisitPatientService {
    rpc GetAll(VisitPatientRequest) returns (VisitPatientListResponse);
    rpc GetById(VisitPatientByIdRequest) returns (VisitPatientResponse);
    rpc Store(VisitPatientStoreRequest) returns (VisitPatientResponse);
    rpc UpdateStatus(VisitPatientStatusRequest) returns (VisitPatientResponse);
  }
  // ... repeat for all 13 services
  ```

- [ ] **Add a stub `CanonicalVisitService`** in the proto for the sign-off write to Backbone (Phase 2 will implement this). Define the interface now so the direction of the call is documented.

> **HUMAN CHECKPOINT:** Review proto interface definitions. Confirm all method signatures are complete and match what the Gateway currently calls on Backbone. This is the contract — approve before generating stubs.

- [ ] **Generate gRPC stubs** from proto:
  ```bash
  protoc --go_out=. --go-grpc_out=. proto/consultation/consultation.proto
  ```

---

### 1.5 Move the 13 Modules

- [ ] **Copy each module** from `wellmed-backbone/internal/domain/{module}/` to `wellmed-consultation/internal/domain/{module}/`. Do this one module at a time and verify compilation after each move.

  Order (least cross-dependencies first, based on audit findings):
  1. `family_relationship`
  2. `data_sync`
  3. `referral`
  4. `assessment`
  5. `treatment`
  6. `examination_treatment`
  7. `medical_service_treatment`
  8. `practitioner_evaluation`
  9. `visit_examination`
  10. `frontline`
  11. `visit_registration_referral`
  12. `visit_registration`
  13. `visit_patient`

- [ ] **Update import paths** in each moved module from `github.com/Kalpa-Health/wellmed-backbone/...` to `github.com/Kalpa-Health/wellmed-consultation/...`.

- [ ] **For any module that calls a Backbone-only module** (found in audit 1.1): add a gRPC client call to Backbone's service instead of the direct in-process call. This is a known dependency that must be wired explicitly.

- [ ] **Register all 13 gRPC services** in `wellmed-consultation/internal/app/application.go`.

- [ ] **Verify Consultation compiles and all 13 services are registered:**
  ```bash
  go build ./...
  go test ./...
  ```

> **HUMAN CHECKPOINT:** Verify `wellmed-consultation` compiles cleanly. Start the service locally and confirm gRPC server registers all 13 services. Basic smoke test: one GetAll call per module via grpcurl or a test client.

---

### 1.6 Update Backbone

- [ ] **Delete the 13 module implementations** from `wellmed-backbone/internal/domain/`. Remove: `visit_patient`, `visit_registration`, `visit_registration_referral`, `visit_examination`, `assessment`, `treatment`, `examination_treatment`, `medical_service_treatment`, `referral`, `practitioner_evaluation`, `frontline`, `data_sync`, `family_relationship`.

- [ ] **Remove their registrations** from `wellmed-backbone/internal/app/application.go`.

- [ ] **Add a gRPC client for Consultation** in Backbone. Backbone needs this to receive the canonical visit record on sign-off (Phase 2). In Phase 1, this is a stub client — the interface is defined, the implementation is `TODO: Phase 2`.

- [ ] **Add `CONSULTATION_GRPC_ADDRESS` env var** to Backbone's config and `.env.example` (for the sign-off write in Phase 2).

- [ ] **Verify Backbone compiles** after module removal:
  ```bash
  go build ./...
  go test ./...
  ```

> **HUMAN CHECKPOINT:** Verify Backbone compiles and its remaining tests pass. Confirm the 13 modules are gone from Backbone's gRPC service catalog.

---

### 1.7 Update Gateway

- [ ] **Add `CONSULTATION_GRPC_ADDRESS`** to Gateway's config, `.env.example`, and `README.md` env var section.

- [ ] **Add a new RPC client struct** for Consultation in Gateway (alongside the existing Backbone client). Follow the same pattern as the Backbone client.

- [ ] **Re-point routing** for all 13 module handlers: change the RPC client target from Backbone to Consultation. The handler code itself does not change — only the client it calls.

  Modules to re-route: `visit_patient`, `visit_registration`, `visit_registration_referral`, `visit_examination`, `assessment`, `treatment`, `examination_treatment`, `medical_service_treatment`, `referral`, `practitioner_evaluation`, `frontline`, `data_sync`, `family_relationship`

- [ ] **Verify Gateway compiles:**
  ```bash
  go build ./...
  go test ./...
  ```

> **HUMAN CHECKPOINT:** Verify Gateway compiles. Confirm: (a) Backbone client routes only Backbone modules, (b) Consultation client routes only Consultation modules. Check for any handler accidentally left pointing to the wrong service.

---

### 1.8 AWS Infrastructure

- [ ] **Open port 50052** (Consultation gRPC) in AWS Security Groups — same pattern as port 50051 for Backbone:
  - Production: Gateway fe SG → Consultation app SG, port 50052 inbound
  - Dev/Staging: Gateway fe SG → Consultation dev-app SG, port 50052 inbound

- [ ] **Add `CONSULTATION_GRPC_ADDRESS`** to the appropriate SSM Parameter Store path:
  `/wellmed/{env}/go/gateway/CONSULTATION_GRPC_ADDRESS`

- [ ] **Create ECS task definition** for `wellmed-consultation` (clone from Backbone task definition, update image and env vars).

---

### 1.9 Deploy and Smoke Test

- [ ] **Deploy to dev environment.** Both services must be running simultaneously.

- [ ] **End-to-end smoke test:**
  - [ ] User login (Backbone — unchanged)
  - [ ] Patient lookup (Backbone — unchanged)
  - [ ] Visit registration (Consultation — new routing)
  - [ ] Visit examination entry (Consultation — new routing)
  - [ ] Assessment store (Consultation — new routing)
  - [ ] Treatment store (Consultation — new routing)
  - [ ] Frontline workflow (Consultation — new routing)

> **HUMAN CHECKPOINT:** Full smoke test on dev. Confirm all 13 Consultation flows work through the new service. Confirm all Backbone flows still work. Sign off before proceeding to doc updates.

---

### 1.10 Documentation Updates

- [ ] **Create `kalpa-docs/services/consultation.md`** — new service doc following the same structure as `services/backbone.md`. Sections: overview, repository, key responsibilities, gRPC interface catalog (all 13 services), multi-tenant design, saga framework (Phase 2 stub), auth architecture, key design decisions.

- [ ] **Update `kalpa-docs/services/backbone.md`:**
  - Remove the 13 moved modules from §4 gRPC Interface Catalog
  - Update §3 Key Responsibilities (remove clinical logic, add canonical record ownership)
  - Update module count (37 → ~15)

- [ ] **Update `kalpa-docs/wellmed-system-architecture.md`:**
  - Add `wellmed-consultation` to the service topology diagram in §2.1
  - Update the Internal Microservices subgraph to include Consultation
  - Update §2.2.2 to describe Consultation's role
  - Update §1.2 product tier table — rename "EMR" to "Consultation" in Lite tier

- [ ] **Update `kalpa-docs/README.md`** — add `services/consultation.md` to the navigation.

- [ ] **Update `kalpa-docs/repo-governance.md`** if it lists service repos.

- [ ] **Update `MEMORY.md`** — add `wellmed-consultation` to the kalpa-health org repos section and note that ADR-002 is Accepted.

> **HUMAN CHECKPOINT:** Review all doc updates for accuracy before committing.

---

### 1.11 Commit and PR

- [ ] **wellmed-consultation repo:** Initial commit — "feat: bootstrap wellmed-consultation service (Phase 1 extraction)"
- [ ] **wellmed-backbone repo:** PR to develop — "refactor: extract consultation modules to wellmed-consultation service"
- [ ] **wellmed-gateway-go repo:** PR to develop — "feat: add consultation gRPC client and re-route consultation module handlers"
- [ ] **kalpa-docs repo:** PR to main — "docs: add consultation service doc and update architecture for Phase 1 extraction"

---

## Phase 2: Saga Coordination (Scope Only — Separate Plan)

Phase 2 is not designed here. It begins after Phase 1 is in production and the team has direct experience with the two-service architecture. Phase 2 planning starts with a scenario session using Phase 1 as context.

**Phase 2 scope items (to be detailed):**

- [ ] **Forward saga — visit creation:**
  - Consultation saga step: push initial service line items to POS (Cashier gRPC)
  - Consultation saga step: pharmacy prescription hook (Pharmacy gRPC, optional)
  - Consultation saga step: additional services added mid-visit (update POS + inventory)
  - Pattern for future linked services entering the visit saga

- [ ] **Backward saga — visit cancellation:**
  - Compensation: void POS transaction for service line items
  - Compensation: release pharmacy prescription hold
  - Compensation: notify future linked services
  - Rollback trigger: user-initiated (wrong doctor, wrong department)

- [ ] **Sign-off one-way doors:**
  - Consultation → Backbone: write canonical FHIR visit record
  - Consultation → RabbitMQ: publish SATU SEHAT sync event
  - Consultation → Cashier: trigger closed invoice creation (with doctor + diagnosis)
  - These are non-compensatable — sign-off is final

- [ ] **Update-visit saga:**
  - Adding a service after visit creation = new, smaller saga (not nested inside original)
  - Original visit saga is closed; update saga opens independently
  - Compensation: void the added service from POS

- [ ] **Shared saga framework:** Plan extraction of `internal/saga/` to a shared Go module (`wellmed-shared` or similar) so Backbone, Consultation, Cashier, Pharmacy all use the same implementation.

- [ ] **Patient data boundary:** Cashier and reporting need patient name/contact on invoices. Define `PatientService.GetById` gRPC call from Cashier → Backbone. This call path does not exist today.

---

## Reference Documents

| Document | Location |
|----------|----------|
| ADR-002 | `kalpa-docs/adrs/ADR-002-consultation-service-extraction.md` |
| Indonesian team brief | `kalpa-docs/meetings/consultation-decomp-team-brief-id.md` |
| Backbone service doc | `kalpa-docs/services/backbone.md` |
| System architecture | `kalpa-docs/wellmed-system-architecture.md` |
| Backbone source | `/Users/alexknecht/Projects/WellMed/wellmed-backbone` |
| Gateway source | `/Users/alexknecht/Projects/WellMed/wellmed-gateway-go` |

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 03 Mar 2026 | Alex + Claude | Initial plan — Phase 1 and Phase 2 scope. |
