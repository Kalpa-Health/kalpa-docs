# ADR-002: SAGA Orchestrator Pattern for Multi-Step Transactions

**Status:** Accepted
**Date:** 2025-01-20
**Author:** Alex
**Reviewers:** Raka (CTO)

> **Note:** This ADR documents a decision that was made and confirmed. The full stub is pending expansion with implementation details. See [`wellmed-system-architecture.md §3.2.3`](../wellmed-system-architecture.md) and [`wellmed-testing-architecture.md §4.1`](../wellmed-testing-architecture.md) for current implementation notes.

---

## 1. Context

1.1 WellMed has multi-step operations that span multiple services — for example, creating an MCU visit requires creating the visit record, reserving lab slots, reserving radiology slots, creating an invoice, and notifying departments. If any step fails, all previously completed steps must be reversed to maintain data integrity.

1.2 Without a structured approach, partial failures leave the system in inconsistent states (e.g., an invoice created but no visit record, or lab slots reserved for a visit that was never created).

---

## 2. Decision

2.1 Use the **SAGA orchestrator pattern** (not choreography) for all multi-step operations that span services. A central orchestrator explicitly coordinates each step and its compensation. On failure, the orchestrator calls `Compensate` on all previously completed steps in reverse order.

2.2 The orchestrator lives in `/pkg/saga/`. Every step must implement both `Execute` and `Compensate`. See [`wellmed-testing-architecture.md §4.1`](../wellmed-testing-architecture.md) for the interface definition and test patterns.

---

## 3. Consequences

## 3.1 Positive

- Data integrity guaranteed — no partial failures leave the system in an inconsistent state
- Centralized visibility — the orchestrator is the single source of truth for transaction state
- Easier debugging — one place to trace what happened and what was compensated

## 3.2 Negative

- Every step requires implementing both `Execute` and `Compensate`
- Compensation logic must be correct — a failed compensation is a serious problem

## 3.3 Risks

- Compensation failures must be logged and alerted on — **mitigation:** orchestrator logs all compensation failures and continues compensating remaining steps
- [ ] TODO: Document idempotency requirements for compensation functions

---

## 4. Alternatives Considered

| Alternative | Pros | Cons | Why Not Chosen |
|-------------|------|------|----------------|
| Choreography (event-driven) | Services are more independent | Requires event bus, harder to track saga state, complex debugging | Overkill for current scale; orchestration gives us clearer visibility |
| Database transactions (2PC) | ACID guarantees | Doesn't work across service boundaries; distributed 2PC is fragile | Multi-service architecture makes this impractical |

---

## 5. Implementation Notes

5.1 [ ] TODO: Document `/pkg/saga/` package structure once implemented.

5.2 [ ] TODO: Add guidance on idempotency requirements for Execute and Compensate functions.

---

## 6. References

- [`wellmed-system-architecture.md §3.2.3`](../wellmed-system-architecture.md)
- [`wellmed-testing-architecture.md §4.1`](../wellmed-testing-architecture.md)

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 01 Mar 2026 | Alex + Claude | Stub — decision captured, implementation notes pending |
