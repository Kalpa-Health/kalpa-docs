# Architecture Docs Sync: Opus Session Brief

**Version:** 1.0
**Date:** 04 March 2026
**Status:** Ready
**Maintained by:** Alex

---

# 1. Purpose

1.1 This brief is the input for a new Claude Opus session. The Opus session will produce a technical PRD titled **"Architecture Docs Sync: ADR-005 and ADR-006 Integration"** and save it to `kalpa-docs/development/prds/arch-docs-sync-PRD.md`.

1.2 The PRD will specify exact changes to two existing documents — `wellmed-system-architecture.md` and `services/backbone.md` — and resolve all open TODO items from ADR-005 and ADR-006. The PRD must be ready for Hamzah (CTO) review and approval before any changes are made.

1.3 Opus has access to the full ADR files and existing docs. This brief is a scoped instruction set, not a full context dump — reference the source documents rather than reproduce them.

---

# 2. Architectural Decisions Driving the Changes

2.1 Three ADRs were accepted in sequence. ADR-002 establishes the structural boundary. ADR-005 and ADR-006 define how services communicate across that boundary. The two existing docs predate ADR-005 and ADR-006 and are inconsistent with both.

## 2.1 ADR-002: Consultation Service Extraction (accepted 2026-03-03)

2.1.1 13 modules move from Backbone to `wellmed-consultation`. The `patient` module stays in Backbone as a canonical identity record.

2.1.2 On doctor final sign-off, Consultation writes a FHIR-shaped canonical visit record to Backbone. This is the one legitimate Consultation → Backbone call. All other dependencies flow downstream.

2.1.3 Full ADR: `kalpa-docs/adrs/ADR-002-consultation-service-extraction.md`

## 2.2 ADR-005: Saga Orchestration Pattern (accepted 2026-03-04)

2.2.1 Backbone is the sole saga orchestrator. No other service implements orchestration logic.

2.2.2 Backbone triggers each saga step by publishing a RabbitMQ message — it never blocks waiting for a response. The receiving module processes the step and responds back to Backbone via gRPC callback.

2.2.3 Module gRPC callback envelope: `{ saga_id, step_id, status, payload }`.

2.2.4 Saga state is persisted in a PostgreSQL saga audit table (source of truth) with Redis as fast state cache. Redis alone is never relied upon — if flushed, the PostgreSQL table enables recovery.

2.2.5 Backbone defines and executes all compensation sequences. Backbone must never absorb business logic from modules — it receives results, not decisions.

2.2.6 Saga step status values: `pending → in_flight → completed / failed / compensated`.

2.2.7 Full ADR: `kalpa-docs/adrs/ADR-005-saga-orchestration.md`

## 2.3 ADR-006: Domain and Service Boundary Decisions (accepted 2026-03-04)

2.3.1 Canonical domain models for shared subjects (`patient`, `user`, `employee`) live exclusively in Backbone. No module duplicates these models.

2.3.2 Modules reference canonical records by ID only. They never maintain their own copy of canonical domain data.

2.3.3 No shared contracts package. Backbone's saga payload composition is the inter-service contract per step.

2.3.4 Modules never call each other directly at runtime. All inter-service data flow is mediated by Backbone via saga.

2.3.5 Modules treat the saga step payload as complete and authoritative. If a module needs additional data, the fix is to update Backbone's payload composition for that step — not to add a direct call to another service.

2.3.6 Full ADR: `kalpa-docs/adrs/ADR-006-domain-service-boundaries.md`

---

# 3. Document 1: wellmed-system-architecture.md

3.1 Current file: `kalpa-docs/wellmed-system-architecture.md`. Current version: 1.1. The PRD must spec each change as a numbered item with exact proposed replacement text or unambiguous instructions.

## 3.1 Section 2.2.2 — Backbone role description

3.1.1 Current text says "Backbone owns tenant config and feature flags." This understates Backbone's role post-ADR-005/006. The PRD must specify replacement text that reflects: (a) canonical domain model owner for `patient`, `user`, `employee`, and other system-wide mastering data; (b) sole saga orchestrator for all cross-service write operations; (c) auth, tenant config, and feature flags as before.

## 3.2 Section 2.3.2 — Gateway target state routing

3.2.1 Current text: "target state: Gateway routing directly to each domain service." This is correct for read requests but incomplete — it implies modules can be called directly for any operation. The PRD must spec a clarification: direct gRPC from Gateway is valid for reads and single-service writes. Cross-service writes that span domain boundaries are orchestrated by Backbone via saga. Gateway never triggers inter-module calls for write operations that span domains.

## 3.3 Section 3.1 — Protocol Matrix

3.3.1 The matrix is missing the saga communication pattern. The PRD must spec a new row: pattern name "RabbitMQ trigger + gRPC callback", use case "saga step coordination across service boundaries", example "visit creation → POS line item push → Backbone callback". This pattern is distinct from plain async RabbitMQ.

## 3.4 Section 3.2 — Internal Service Communication diagram

3.4.1 The current mermaid diagram contains the line `EMR →|gRPC sync| CASH`. This directly violates ADR-006 §2.4 (modules never call each other directly). The PRD must spec removal of this arrow and its replacement with a saga flow: Backbone publishes RabbitMQ trigger → module executes step → gRPC callback to Backbone.

3.4.2 The PRD must include a redrawn mermaid diagram showing the correct saga flow alongside the existing direct gRPC read path.

## 3.5 Section 3.2.3 — Saga orchestrator description

3.5.1 Current text is one vague paragraph. The PRD must spec a full replacement subsection describing ADR-005's pattern: Backbone as sole orchestrator, RabbitMQ trigger, gRPC callback envelope, PostgreSQL audit table, Redis state cache, compensation sequences, and the no-business-logic rule. Cross-reference ADR-005.

## 3.6 Section 6.3 — Shared Packages table

3.6.1 `/pkg/saga` is listed as a shared package, implying any service can import it. ADR-005 §5.2 states saga is internal to Backbone. ADR-002 §5.2 states Consultation copies `internal/saga/` for Phase 1 — it does not import Backbone as a Go module. The PRD must spec a correction to the `/pkg/saga` entry: clarify it is internal to Backbone, not a shared importable package. Consultation holds a local copy in Phase 1; extraction to a shared module is deferred.

---

# 4. Document 2: services/backbone.md

4.1 Current file: `kalpa-docs/services/backbone.md`. Current version: 1.0. Same instruction as above — PRD specifies exact changes section by section.

## 4.1 Section 1 — Overview

4.1.1 Current text: "Backbone is the core internal gRPC backend... all business logic... all direct database access." This is wrong post-ADR-005/006. Business logic belongs to domain services, not Backbone. The PRD must spec replacement overview text reflecting Backbone's three actual roles: (a) canonical domain model owner, (b) sole saga orchestrator, (c) auth and tenant config. One or two sentences — no inflation.

## 4.2 Section 3 — Key Responsibilities

4.2.1 "Business logic for all 19 domain services" — wrong. 13 modules are extracting. The PRD must spec a rewrite of the responsibilities list to reflect: canonical domain model ownership, saga orchestration (sole coordinator, composes payloads, manages state, executes compensation), canonical visit record reception from Consultation on sign-off, auth, multi-tenant database access, and POS pass-through (for Transaction, Billing, Invoice — these are Backbone's downstream calls as saga orchestrator, not business logic it owns).

## 4.3 Section 4 — gRPC Interface Catalog

4.3.1 The current catalog lists 19 services, 13 of which are moving to `wellmed-consultation`. The PRD must spec a split of this catalog into three subsections:

4.3.2 Staying in Backbone: `UserService`, `UnicodeService`, `MenuService`, `AutoListService`, `RoleService`, `PatientService` (canonical identity), `ItemService`.

4.3.3 Moving to `wellmed-consultation`: `AssessmentService`, `TreatmentService`, `VisitPatientService`, `VisitRegistrationService`, `VisitRegistrationReferralService`, `VisitExaminationService`, `ReferralService`, `FrontlineService`. These should be marked as "pending extraction" in the updated doc, not deleted — they are still in Backbone until Phase 1 of ADR-002 completes.

4.3.4 New Backbone gRPC interfaces to add: `CanonicalVisitService` (receives sign-off write from Consultation per ADR-002 §5.4) and `SagaCallbackService` (standard callback endpoint that all modules call to respond to saga steps per ADR-005 §5.3).

## 4.4 Section 6 — Saga Framework

4.4.1 Current section describes SYNC and ASYNC blocking execution modes. ADR-005 replaces both with a single non-blocking pattern. The PRD must spec a full rewrite of this section covering: RabbitMQ publish to trigger (never blocks), gRPC callback to receive step result, standard callback envelope, state machine (`pending → in_flight → completed / failed / compensated`), PostgreSQL audit table as source of truth, Redis as fast state cache, compensation execution on timeout or failure, and the no-business-logic rule. Cross-reference ADR-005 in the section header.

---

# 5. Open TODOs — Opus Must Propose Concrete Answers

5.1 The following TODOs appear in ADR-005 and ADR-006 and must be resolved in the PRD. For each, Opus proposes a concrete answer. Hamzah approves or modifies at PRD review. No TODO should remain open after the PRD is approved.

## 5.1 From ADR-005

5.1.1 **Retry backoff strategy and max retry count per saga step type.** Propose defaults by step category. Consider: fast internal steps (Backbone → Consultation, Backbone → POS), slow external integration steps (Backbone → SATU SEHAT async), and compensation steps (which may need different limits than forward steps).

5.1.2 **Timeout thresholds per step before compensation triggers.** Propose values per category. A step that times out is different from a step that returns a failure — compensation logic should distinguish them.

5.1.3 **RabbitMQ exchange and routing key naming convention for saga step triggers.** Propose a consistent pattern. The pattern should encode enough context (saga type, step name, service target) to be unambiguous in RabbitMQ management UI and in logs.

5.1.4 **gRPC callback endpoint contract in Backbone.** Propose the full proto definition shape for `SagaCallbackService`. Must include: service name, method name, request message fields (`saga_id`, `step_id`, `status`, `payload` — with types), response message, and error handling contract (how a failed step vs a malformed callback is distinguished).

## 5.2 From ADR-006

5.2.1 **Validation requirements for module step handlers.** Define what constitutes a malformed payload. Define what the module must return to Backbone via the gRPC callback when payload validation fails (vs when business logic fails — these need different handling at the orchestrator level).

5.2.2 **Naming convention for reference fields across modules.** Confirm or refine: `*_id` suffix for simple foreign key references (e.g., `patient_id`, `visit_id`), `reference_id` + `reference_type` pair for polymorphic references (e.g., POS line items that can reference a visit or a walk-in). Propose a Go struct tag or proto field annotation convention to make this discoverable.

5.2.3 **Criteria for what qualifies as a Backbone-level canonical domain model.** Propose a decision rule. The rule should be concrete enough that a developer can apply it to a new entity without asking. Consider: does the entity need to be referenced by more than one service? Is it a subject of the business as a whole (not a subject of one module's workflow)? Is it something that must remain consistent across modules?

---

# 6. Output Format for the PRD

6.1 The PRD must follow the kalpa-docs PRD format (`kalpa-docs/development/prds/arch-docs-sync-PRD.md`) per the markdown style guide Section 9.

6.2 Required PRD sections in order: Problem Statement, Users and Goals, Requirements (Functional + Non-functional), Acceptance Criteria, Out of Scope, Dependencies, Open Questions (should be empty after Opus resolves the TODOs), Feeds Into.

6.3 The Out of Scope section must explicitly name: EMR/Consultation naming alignment (separate plan), consultation extraction Phase 1 execution (separate plan), Phase 2 saga wiring design, any new service documentation beyond backbone.md updates.

6.4 The Dependencies section must reference ADR-002, ADR-005, and ADR-006 as resolved dependencies.

6.5 The Feeds Into field must point to the plan that will execute the doc updates: `kalpa-docs/plans/arch-docs-sync/arch-docs-sync-PLAN.md` (to be created after PRD approval).

6.6 The document will be reviewed and approved by Hamzah before any changes to the existing docs are made.

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 04 Mar 2026 | Alex + Claude | Initial brief — scopes Opus PRD session for arch docs sync following ADR-005 and ADR-006 acceptance. |
