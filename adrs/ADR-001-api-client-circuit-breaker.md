# ADR-001: API Client Pattern with Circuit Breaker

**Status:** Accepted
**Date:** 2025-01-20
**Author:** Alex
**Reviewers:** Hamzah (CTO)

---

## 1. Context

1.1 WellMed integrates with multiple external APIs (SATU SEHAT, P-Care, Xendit, etc.) that have varying reliability. Government APIs in particular are notoriously unstable and poorly documented.

1.2 Without a standardized approach, each integration team would implement retry logic, error handling, and failure capture differently. This creates inconsistent resilience, duplicated code, and makes it difficult to reason about system behavior under failure conditions.

1.3 We need a standardized approach to: handle transient failures with intelligent retry, prevent cascade failures when external services are down, capture failed requests for later retry, and maintain consistent logging and metrics across all integrations.

---

## 2. Decision

2.1 Implement a reusable API client package (`/pkg/apiclient`) with policy-based configuration that all external integrations must use. The package provides:

1. **Policy-based configuration** — Different retry and timeout settings per API category
2. **Circuit breaker** — Prevents hammering failing services; opens after threshold failures
3. **Dead letter queue** — Failed requests (all retries exhausted) published to RabbitMQ DLQ for manual review and replay
4. **Structured logging** — JSON logs with correlation IDs on every request and response
5. **Metrics interface** — Pluggable metrics collection

2.2 **Policy groups** govern retry and timeout behavior per API category:

| Policy Group | APIs | Retries | Circuit threshold | Timeout |
|-------------|------|---------|-------------------|---------|
| critical-gov | SATU SEHAT, P-Care | 10 | 10 failures | 60s |
| standard-gov | Mobile JKN | 5 | 5 failures | 45s |
| payments | Xendit | 3 | 3 failures | 30s |
| business | Jurnal, Xero, Talenta | 5 | 5 failures | 30s |
| notifications | Zoho Desk, Kyoo | 2 | 3 failures | 15s |

---

## 3. Consequences

## 3.1 Positive

- Consistent error handling and resilience across all external integrations
- Automatic retry and circuit breaking without per-integration implementation
- Failed requests are never lost — DLQ captures everything for replay
- Easy to add new integrations by selecting an existing policy group
- Observable by default — every API interaction is logged with correlation IDs

## 3.2 Negative

- Initial implementation effort (~3–5 days)
- Team needs to learn the new patterns before building integrations
- Slight overhead on every external call (circuit breaker state check)

## 3.3 Risks

- Circuit breaker thresholds may need tuning per API — **mitigation:** make all thresholds configurable via policy groups
- DLQ could accumulate silently if not monitored — **mitigation:** dashboard alert when DLQ depth exceeds threshold

---

## 4. Alternatives Considered

| Alternative | Pros | Cons | Why Not Chosen |
|-------------|------|------|----------------|
| Per-integration retry logic | Simple to understand in isolation | Inconsistent, duplicated code, impossible to reason about at system level | Doesn't scale; every integration team solves the same problem differently |
| Third-party library (go-resilience) | Battle-tested | Less control, external dependency, doesn't integrate with our DLQ/RabbitMQ setup | Need custom DLQ integration; easier to own the code |
| No retry, fail fast | Simple | Lost requests, poor UX, compliance risk | Unacceptable for healthcare data — SATU SEHAT sync failures cannot be silently dropped |

---

## 5. Implementation Notes

5.1 Key files in `/pkg/apiclient/`:

- `config.go` — Policy group definitions and defaults
- `circuit_breaker.go` — Circuit breaker implementation (closed → open → half-open states)
- `dead_letter.go` — RabbitMQ DLQ integration for permanent failures
- `logging.go` — Structured JSON logging with correlation IDs
- `metrics.go` — Pluggable metrics interface
- `client.go` — Main client that composes all of the above

5.2 The circuit breaker follows the standard three-state model: **Closed** (normal operation) → **Open** (fast-fail all requests after threshold failures) → **Half-Open** (allow one test request to check if service has recovered) → back to Closed on success.

5.3 The DLQ is the last resort — it fires only after all retry attempts are exhausted. Every message in the DLQ contains the full original request context for replay.

5.4 All new external API integrations must use this client. Direct `http.Client` calls to external APIs are not permitted outside of this package.

---

## 6. References

- [`wellmed-system-architecture.md §3.3`](../wellmed-system-architecture.md) — API client in system context
- [`archive/wellmed-api-client-pattern-v1.md`](../archive/wellmed-api-client-pattern-v1.md) — Detailed implementation spec (future project)
- Circuit Breaker pattern: Martin Fowler, [martinfowler.com/bliki/CircuitBreaker.html](https://martinfowler.com/bliki/CircuitBreaker.html)

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-20 | Alex + Claude | Initial ADR — accepted by CTO (Hamzah) |
| 1.1 | 01 Mar 2026 | Alex + Claude | Reformatted to kalpa-docs ADR standard |
