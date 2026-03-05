# ADR-006: Domain and Service Boundary Decisions

**Status:** Accepted
**Date:** 2026-03-04
**Author:** Alex
**Reviewers:** Hamzah (CTO)

---

## 1. Context

1.1 WellMed is a multi-module microservice system. As modules are added (consultation, POS, apotek, lab, SCM, and future modules), decisions are needed about where canonical domain models live, how shared data is accessed across services, and whether inter-service contracts require a shared library.

1.2 The risk of getting this wrong is either (a) duplicating canonical models across services creating consistency problems, or (b) centralising too much at runtime creating a bottleneck that undermines microservice independence.

1.3 This ADR covers three related decisions that were resolved together: domain model ownership, runtime data access pattern, and inter-service contract strategy.

---

## 2. Decision

2.1 **Canonical domain models for shared subjects live exclusively in `wellmed-backbone`.** This includes `patient`, `user`, `employee`, and other mastering data that is a subject of WellMed as a whole. No module duplicates these models.

2.2 **Modules reference canonical records by ID only.** Module-consultation stores `patient_id` as a reference. Module-POS stores `reference_id` and `reference_type`. Modules do not maintain their own copy of canonical domain data.

2.3 **No separate contracts package or shared repo is required.** Inter-service communication shape is defined by the payload Backbone composes and publishes per saga step (see ADR-005). Modules consume what Backbone sends — they do not need to import a shared type library or call each other directly.

2.4 **Modules never call each other directly at runtime.** All inter-service data flow is mediated by Backbone via saga orchestration. If module-POS needs `visit_id` from module-consultation, Backbone composes a payload containing `visit_id` and delivers it to module-POS as part of the saga step trigger.

2.5 **No shared data repo in Backbone at runtime.** Modules do not query Backbone as a data hub outside of saga flows. Data needed by a module is either (a) owned by that module, (b) a reference ID received via saga payload, or (c) harvested from the saga event payload composed by Backbone.

2.6 **Each module owns its domain vocabulary.** Module-SCM knows `item`. A future module-medical knows `medicine`, `medictool`, `lab_sample`. These domain terms are not shared or mapped to each other — Backbone composes cross-domain payloads when a saga step requires data from multiple domains.

---

## 3. Consequences

### 3.1 Positive

- **No data duplication** — canonical models have one owner, one source of truth
- **No runtime bottleneck from shared data hub** — modules never call Backbone for reads outside saga flows
- **No shared library versioning overhead** — no contracts package to update and coordinate across services on every model change
- **Clean domain boundaries** — each module's vocabulary is its own; cross-domain translation happens only in Backbone saga payload composition
- **Backbone payload is the contract** — changing what a module needs means updating Backbone's payload composition for that saga step, one place

### 3.2 Negative

- **Backbone payload composition is load-bearing** — if Backbone sends an incomplete payload to a module, that module cannot self-heal by calling another service
- **Less module autonomy on data** — modules cannot independently fetch data they need; they are dependent on receiving it via saga
- **Backbone scope discipline required** — as saga types grow, Backbone payload composition logic grows; must not absorb business logic

### 3.3 Risks

- Backbone payload for a saga step becomes stale as module data needs evolve — **mitigation:** module data requirements are reviewed when saga types are added or modified; Backbone payload is the explicit contract surface
- Module receives incomplete payload and fails silently — **mitigation:** modules must validate required fields on receipt and return a structured error to Backbone via the gRPC callback
- [ ] TODO: Define validation requirements for module step handlers — what constitutes a malformed payload and how it is reported back

---

## 4. Alternatives Considered

| Alternative | Pros | Cons | Why Not Chosen |
|---|---|---|---|
| Separate `wellmed-contracts` package | Consistent types, each MS imports independently, no runtime bottleneck | Version coordination on every model change; MS must manage imports; breaking changes force multi-service redeploy | Overhead not justified — Backbone saga payload already defines the communication shape per step |
| Shared data repo in Backbone (runtime) | One place to query shared data; simple for consumers | Backbone in every read path; runtime bottleneck; restart causes data inaccessibility for all modules | Directly contradicts MS independence goal |
| Domain models duplicated per module | Each module fully autonomous | Consistency nightmare; patient data diverges across services; migration complexity | Unacceptable for healthcare data integrity |
| MS-to-MS direct calls with contracts | Modules can fetch what they need on demand | N² contract surface; each new module adds contracts to all services it calls; Backbone not in read path but complexity explodes | Contract management overhead exceeds benefit; Backbone mediation is cleaner |

---

## 5. Implementation Notes

5.1 When a new saga type is designed, the payload schema for each step must be explicitly defined and reviewed — this is the primary place where cross-module data requirements are surfaced and resolved.

5.2 Modules must treat the saga step payload as the complete and authoritative data source for that step. If a module needs additional data, the fix is to update Backbone's payload composition for that step, not to add a direct call to another service.

5.3 Reference IDs in module models (e.g., `patient_id` in consultation, `reference_id` in POS) are opaque identifiers. Modules do not resolve these to full objects at runtime — they are used for correlation and reporting only.

5.4 [ ] TODO: Document the naming convention for reference fields across modules (`*_id` suffix, `reference_id` + `reference_type` pattern for polymorphic references).

5.5 [ ] TODO: Establish review process for when new canonical domain models are added to Backbone — criteria for what qualifies as a Backbone-level domain vs a module-local domain.

---

## 6. References

- [`ADR-005-saga-orchestration.md`](./ADR-005-saga-orchestration.md)
- [`ADR-004-tenant-isolation.md`](./ADR-004-tenant-isolation.md)
- [`wellmed-system-architecture.md`](../wellmed-system-architecture.md)

---

# Edit Log

| Version | Date | Author | Changes |
|---|---|---|---|
| 0.1 | 04 Mar 2026 | Alex + Claude | Initial draft — decisions captured from architecture alignment session with Hamzah |
