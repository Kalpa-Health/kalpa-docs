# ADR-005: Saga Orchestration Pattern

**Status:** Accepted
**Date:** 2026-03-04
**Author:** Alex
**Reviewers:** Hamzah (CTO)

---

## 1. Context

1.1 WellMed uses a microservice architecture where multi-step business operations (e.g., creating a visitation) span multiple services — Backbone, module-consultation, module-pos, module-apotek, and others. These operations require coordinated writes across services with compensation logic if any step fails.

1.2 Two core tensions drove this decision: (a) where saga orchestration logic lives, and (b) how inter-service communication should work to preserve true microservice independence while maintaining data consistency.

1.3 The system stack includes Go microservices, PostgreSQL, Redis, RabbitMQ, and Elasticsearch. All services are reachable via gRPC. RabbitMQ is available for async messaging.

---

## 2. Decision

2.1 **Saga orchestration is centralised in `wellmed-backbone`.** No other microservice implements saga templates or orchestration logic. Backbone is the sole coordinator of multi-step cross-service operations.

2.2 **The async request/callback pattern is used for all saga steps:**
- Backbone triggers each saga step by publishing a message to RabbitMQ — Backbone does not block waiting for a response
- The receiving module processes the step and responds back to Backbone via gRPC when complete
- Backbone saga state table tracks the status of every step

2.3 **Backbone composes the full payload for each saga step.** Modules receive everything they need in the RabbitMQ message from Backbone. Modules never call each other directly — all inter-service data flow is mediated by Backbone.

2.4 **Saga state is persisted in a dedicated saga audit table (PostgreSQL).** Redis is used as a fast state cache for in-flight sagas. The PostgreSQL table is the source of truth for recovery and audit. Redis alone is not relied upon — if Redis is flushed, the saga table enables recovery.

2.5 **Compensation logic is defined in Backbone.** If a step fails or times out, Backbone executes the compensation sequence for that saga type based on which steps have already completed.

---

## 3. Consequences

### 3.1 Positive

- **True MS independence** — Backbone publishes and moves on. A slow or down module stalls only that saga instance, not Backbone itself or other sagas
- **Single source of truth for orchestration** — all saga logic, state, and compensation in one place; easier to debug and audit
- **Modules stay thin** — modules receive a composed payload and execute one step; no orchestration awareness required
- **Durability without RabbitMQ dependency** — saga audit table ensures no step is lost even if a module is down for an extended period; Backbone retries on recovery

### 3.2 Negative

- **Backbone is a single point of failure for saga flows** — if Backbone is down, no saga can be initiated or progressed
- **Backbone payload composition grows over time** — as modules need more data per step, Backbone must be updated to include it
- **Backbone can accumulate complexity** — strict discipline required to keep Backbone as orchestration-only, never absorbing business logic from other modules

### 3.3 Risks

- Backbone becoming a de facto monolith as saga types multiply — **mitigation:** each saga type is a discrete, independently testable unit; periodic review of Backbone scope
- Redis flush during in-flight saga — **mitigation:** PostgreSQL saga table is authoritative; Redis is cache only
- [x] ~~TODO: Define retry backoff strategy and max retry count per saga step type~~ — **Resolved** in saga-pattern-extended-PRD.md §3.2.2. Three categories: `internal-fast` (1s base, 5 retries), `external-integration` (5s base, 8 retries), `compensation` (2s base, 10 retries). Exponential backoff with 20% jitter.
- [x] ~~TODO: Define timeout thresholds per step before compensation is triggered~~ — **Resolved** in saga-pattern-extended-PRD.md §3.2.3. `internal-fast`: 30s, `external-integration`: 300s, `compensation`: 60s/step. `REJECTED` callback → retry (no compensation). `FAILED` or timeout → compensation.

---

## 4. Alternatives Considered

| Alternative | Pros | Cons | Why Not Chosen |
|---|---|---|---|
| Saga orchestration distributed per MS | True independence, blast radius limited to one MS | Each MS needs saga infrastructure; cross-service sagas require "saga of sagas" complexity | Too much duplication for current team size; coordination complexity outweighs benefits |
| gRPC for all saga step triggers | Simpler to build and debug; saga table provides durability | Backbone blocks on every gRPC call; slow/down module stalls Backbone thread | Undermines MS independence at runtime — the core reason for MS architecture |
| gRPC trigger with RabbitMQ fallback | Attempts best of both | Two code paths for same operation; 3s timeout before fallback creates guaranteed stall; state management complexity | Complexity without clean benefit; fallback path is harder to test and reason about |
| RabbitMQ for triggers AND responses | Full async decoupling | Backbone never gets a direct response; reconciliation becomes poll-based; harder to debug | Callback via gRPC gives Backbone a clean response signal without blocking on the trigger |

---

## 5. Implementation Notes

5.1 Each saga type is defined in Backbone as a discrete step sequence with named steps, payload schema per step, and compensation sequence.

5.2 Saga step status values: `pending` → `in_flight` → `completed` / `failed` / `compensated`.

5.3 Modules respond to Backbone via gRPC with a standard response envelope: `{ saga_id, step_id, status, payload }`.

5.4 Backbone must never include business logic belonging to another module's domain. If a step requires a business decision (e.g., whether to allow a visit given a patient's outstanding balance), that logic lives in the relevant module — Backbone only receives the result.

5.5 **Resolved** — Exchange: `wellmed.saga`. Routing key: `saga.{saga_type}.{step_name}.{trigger|compensate}`. Dead letter exchange: `dlx.wellmed.saga`. Dead letter queue: `dlx.{routing_key}`. See saga-pattern-extended-PRD.md §3.2.4.

5.6 **Resolved** — Single method `SagaCallbackService.ReportStepResult`. Fields: `saga_id`, `step_id`, `status` (COMPLETED/FAILED/REJECTED), `payload`, `error_code`, `error_message`, `idempotency_key`, `tenant`. Proto at `wellmed-backbone/proto/saga_callback.proto`. See saga-pattern-extended-PRD.md §3.2.5 and §3.3.1.

---

## 6. References

- [`ADR-004-tenant-isolation.md`](./ADR-004-tenant-isolation.md)
- [`wellmed-system-architecture.md`](../wellmed-system-architecture.md)
- [`ADR-006-domain-service-boundaries.md`](./ADR-006-domain-service-boundaries.md)

---

# Edit Log

| Version | Date | Author | Changes |
|---|---|---|---|
| 0.1 | 04 Mar 2026 | Alex + Claude | Initial draft — decisions captured from architecture alignment session with Hamzah |
| 1.0 | 05 Mar 2026 | Alex + Claude | Closed all four open TODOs: retry backoff strategy (§3.3), timeout thresholds (§3.3), RabbitMQ naming convention (§5.5), gRPC callback endpoint contract (§5.6). All resolved in saga-pattern-extended-PRD.md. Saga framework rework (Phase 3) complete — `go build ./...` and `go test ./internal/saga/...` pass cleanly. |
