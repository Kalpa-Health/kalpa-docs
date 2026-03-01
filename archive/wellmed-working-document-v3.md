# WellMed Testing Strategy - Working Document
## Consolidated Decisions, ADR Template & Work Plan

**Version**: 2.0  
**Status**: Team decisions incorporated - ready for execution  
**Owner**: Alex (CMO)

---

# Part 1: Key Decisions (Confirmed)

This section captures all decisions made through team discussion. These are no longer open questions.

## 1.1 Architecture & Infrastructure Decisions

| Decision | Choice | Notes |
|----------|--------|-------|
| **Tenant isolation** | Separate DB per tenant, schemas per service | Eliminates tenant_id filter concerns |
| **SAGA pattern** | Orchestrator (not choreography) | Centralized coordination for reliability |
| **Proto/gRPC** | All-in (already built) | Type safety + auto-docs |
| **Request tracing** | AWS X-Ray | Cross-service request tracking |
| **Secrets management** | AWS Secrets Manager | For API tokens and credentials |
| **Log format** | JSON (uber-go/zap) | Including Redis; CloudWatch aggregation |
| **Log aggregation** | CloudWatch (start here) | Can revisit if costs escalate |
| **Infrastructure owner** | Hamzah | All AWS, Postgres, Redis, RabbitMQ, ES |

## 1.2 CI/CD & Process Decisions

| Decision | Choice | Notes |
|----------|--------|-------|
| **CI platform** | GitHub Actions | |
| **Unit test timeout** | 8 minutes | Revised from 4 min |
| **Integration test timeout** | 12 minutes | Revised from 6 min |
| **Total build timeout** | 15 minutes | Revised from 10 min; explore parallelization |
| **Branch merge levels** | Green/Yellow/Red | See below |
| **PR review** | Claude-assisted summary | Indonesian default |
| **ADR creation** | Claude drafts, CTO approves | Part of PR process |

### Branch & Merge Rules

| Level | Scope | Who Can Merge | Requirements |
|-------|-------|---------------|--------------|
| 🟢 Green | Minor changes, bug fixes | All developers | CI passes |
| 🟡 Yellow | Database migrations | Senior devs (Fajri, Hamzah) | Green + migration review |
| 🔴 Red | New modules, major features, prod deploy | CTO only | Yellow + architecture review |

**Flow**: `feature branch` → `dev` → `staging` → `production`  
Staging → Production is always 🔴 Red.

## 1.3 Testing Coverage Targets

| Layer | Minimum Coverage | Rationale |
|-------|------------------|-----------|
| Handler (Gateway, Frontend) | 30% | Mostly plumbing; integration tests catch bugs |
| Service Layer | 80% | Business logic lives here; bugs hurt most |
| Domain Layer (Utilities) | 50% | Important but less critical than business logic |
| Repository Layer | Via integration tests | Need real DB to test SQL meaningfully |

## 1.4 External API Policy Groups

| Policy Group | APIs | Retry | Circuit Threshold | Timeout |
|--------------|------|-------|-------------------|---------|
| critical-gov | SATU SEHAT, P-Care | 10 | 10 failures | 60s |
| standard-gov | Mobile JKN | 5 | 5 failures | 45s |
| payments | Xendit | 3 | 3 failures | 30s |
| business | Jurnal, Xero, Talenta | 5 | 5 failures | 30s |
| notifications | Zoho Desk, Kyoo | 2 | 3 failures | 15s |

## 1.5 AWS Backup Strategy

| Environment | Asset | Frequency | Retention |
|-------------|-------|-----------|-----------|
| Production | DB WAL | Continuous | 7 days |
| Production | DB snapshot | Daily | 14 days |
| Production | Full snapshot (app + frontend) | Weekly or pre-upgrade | Last 4 copies |
| Staging | DB snapshot | Weekly | 2 weeks |
| Staging | Config/structure | Git (IaC) | Forever |
| Dev | None | - | - |

## 1.6 Response Time SLOs

For CloudWatch monitoring thresholds:

| Threshold | Level | Action |
|-----------|-------|--------|
| < 200ms | 🟢 Green | Normal |
| 200-500ms | 🟡 Yellow | Monitor |
| 500-1000ms | 🟠 Orange | Investigate |
| > 1000ms | 🔴 Red | Alert |

## 1.7 Resolved Concerns

These were flagged as concerns but have been addressed:

| Original Concern | Resolution |
|------------------|------------|
| Single database with logical separation | **Resolved**: Separate DB per tenant |
| Proto/gRPC learning curve | **Resolved**: Already all-in |
| SAGA without orchestrator | **Resolved**: Orchestrator pattern confirmed |
| Who owns infrastructure | **Resolved**: Hamzah owns all |
| CTO review bottleneck | **Resolved**: Green-level self-merge after CI |
| Request tracing | **Resolved**: AWS X-Ray |
| Secrets management | **Resolved**: AWS Secrets Manager |
| Log aggregation | **Resolved**: CloudWatch (start here) |
| CI timing too aggressive | **Resolved**: Extended to 8/12/15 minutes |
| Tenant isolation in queries | **Resolved**: Separate DBs eliminate this |
| Backup retention | **Revised**: See updated backup strategy |

---

# Part 2: ADR Template & Example

## 2.1 ADR Template

```markdown
# ADR-XXX: [Title]

**Status**: Proposed | Accepted | Deprecated | Superseded  
**Date**: YYYY-MM-DD  
**Author**: [Name]  
**Reviewers**: [Names]

## Context

[What is the issue? What forces are at play?]

## Decision

[What is the change being proposed/decided?]

## Consequences

### Positive
- [Benefit 1]
- [Benefit 2]

### Negative
- [Drawback 1]
- [Drawback 2]

### Risks
- [Risk 1 and mitigation]

## Alternatives Considered

| Alternative | Pros | Cons | Why Not Chosen |
|-------------|------|------|----------------|
| [Option A] | | | |
| [Option B] | | | |

## Implementation Notes

[Any specific guidance for implementation]

## References

- [Link to relevant docs]
```

## 2.2 ADR-001: API Client Pattern with Circuit Breaker

**Status**: Accepted  
**Date**: 2025-01-20  
**Author**: Alex  
**Reviewers**: Raka (CTO)

### Context

WellMed integrates with multiple external APIs (SATU SEHAT, P-Care, Xendit, etc.) that have varying reliability. We need a standardized approach to:
- Handle transient failures with intelligent retry
- Prevent cascade failures when external services are down
- Capture failed requests for later retry
- Maintain consistent logging and metrics

### Decision

Implement a reusable API client package (`/pkg/apiclient`) with:
1. **Policy-based configuration** - Different retry/timeout settings per API category
2. **Circuit breaker** - Prevents hammering failing services
3. **Dead letter queue** - Failed requests go to RabbitMQ DLQ for retry
4. **Structured logging** - JSON logs with correlation IDs
5. **Metrics interface** - Pluggable metrics collection

### Consequences

#### Positive
- Consistent error handling across all external integrations
- Automatic resilience without per-integration code
- Failed requests are never lost (DLQ capture)
- Easy to add new integrations following established pattern

#### Negative
- Initial implementation effort (~3-5 days)
- Team needs to learn new patterns
- Slight overhead on every external call (circuit breaker check)

#### Risks
- Circuit breaker thresholds may need tuning per API (mitigation: make configurable)
- DLQ could accumulate if not monitored (mitigation: dashboard alerts)

### Alternatives Considered

| Alternative | Pros | Cons | Why Not Chosen |
|-------------|------|------|----------------|
| Per-integration retry logic | Simple to understand | Inconsistent, duplicated code | Doesn't scale |
| Third-party library (go-resilience) | Battle-tested | Less control, dependency | Need custom DLQ integration |
| No retry, fail fast | Simple | Lost requests, poor UX | Unacceptable for healthcare |

### Implementation Notes

See `/pkg/apiclient` for full implementation. Key files:
- `config.go` - Policy group definitions
- `circuit_breaker.go` - Circuit breaker implementation
- `dead_letter.go` - RabbitMQ DLQ integration
- `client.go` - Main client with all features

### References

- WellMed API Client Pattern document
- Circuit Breaker pattern (Martin Fowler)

---

# Part 3: Work Plan

This section organizes all pending work into logical workstreams. Each item is designed to be farmable to either human developers or AI agents with clear scope and acceptance criteria.

## 3.1 Workstream: CI/CD Pipeline Setup

**Owner**: Alex  
**Dependencies**: None  
**Estimated effort**: 2-3 days

- [ ] Create GitHub Actions lint workflow — PRs blocked if `golangci-lint` fails *(AI agent)*
- [ ] Create GitHub Actions unit test workflow — PRs blocked if tests fail; 8-min timeout *(AI agent)*
- [ ] Create GitHub Actions integration test workflow — Runs on push to develop; 12-min timeout *(AI agent)*
- [ ] Set up Claude PR review workflow — Auto-comment with review summary on PRs *(AI agent)*
- [ ] Create Makefile template for services — `make lint`, `make test`, `make build` targets *(AI agent)*
- [ ] Document local dev requirements — README section on pre-push checks *(AI agent)*

## 3.2 Workstream: API Client Package

**Owner**: Dev team  
**Dependencies**: CI/CD pipeline (for testing)  
**Estimated effort**: 3-5 days

- [ ] Implement `config.go` — Policy groups defined per 1.4 *(Human dev)*
- [ ] Implement `errors.go` — Custom error types with context *(Human dev)*
- [ ] Implement `circuit_breaker.go` — Opens after threshold failures, half-open retry *(Human dev)*
- [ ] Implement `dead_letter.go` — Failed requests published to RabbitMQ DLQ *(Human dev)*
- [ ] Implement `logging.go` — JSON structured logs with zap *(Human dev)*
- [ ] Implement `metrics.go` — Interface for pluggable metrics *(Human dev)*
- [ ] Implement `client.go` — Main client integrating all above *(Human dev)*
- [ ] Write unit tests — 80% coverage on client package *(Human dev)*
- [ ] Create SATU SEHAT client — Using API client package *(Human dev)*
- [ ] Create in-process mock for SATU SEHAT — For unit testing dependent services *(Human dev)*

## 3.3 Workstream: RabbitMQ Infrastructure

**Owner**: Hamzah  
**Dependencies**: None  
**Estimated effort**: 2-3 days

- [ ] Document current queue topology — Accurate registry of existing queues *(Human dev)*
- [ ] Create `queues.yaml` registry — All queues documented with purpose *(AI agent draft + Human validate)*
- [ ] Set up DLQ infrastructure — DLQ exists for each critical queue *(Human dev)*
- [ ] Implement DLQ monitoring — Alert when DLQ depth > threshold *(Human dev)*
- [ ] Add message idempotency pattern — Document pattern for handlers *(AI agent)*
- [ ] Audit existing handlers for idempotency — List which handlers need updates *(Human dev)*

## 3.4 Workstream: Monitoring & Observability

**Owner**: Alex (setup), Hamzah (infra)  
**Dependencies**: Services deployed  
**Estimated effort**: 3-4 days

- [ ] Set up CloudWatch log aggregation — All service logs queryable in CloudWatch *(Hamzah)*
- [ ] Configure AWS X-Ray tracing — Request traces visible across services *(Hamzah)*
- [ ] Create response time alarms — Alerts per SLO thresholds (1.6) *(AI agent config + Hamzah deploy)*
- [ ] Set up AWS Budgets alerts — Alert at 80% and 100% of monthly budget *(Alex)*
- [ ] Implement `/health` endpoint standard — All services expose same format *(AI agent template + Human implement)*
- [ ] Create monitoring dashboard v1 — Service health + deployment log visible *(Human dev)*

## 3.5 Workstream: Documentation & Standards

**Owner**: Alex  
**Dependencies**: None  
**Estimated effort**: 2-3 days (ongoing)

- [ ] Set up `/docs` directory structure — Matches strategy doc taxonomy *(AI agent)*
- [x] Create ADR template — Per section 2.1 *(Done)*
- [x] Write ADR-001 (API Client) — Per section 2.2 *(Done)*
- [ ] Write ADR-002 (SAGA Orchestrator) — Documents orchestrator decision *(AI agent)*
- [ ] Write ADR-003 (Tenant Isolation) — Documents separate DB per tenant *(AI agent)*
- [ ] Create `permissions.yaml` matrix — RBAC roles and resources *(AI agent draft + Human validate)*
- [ ] Document database migration process — Best practice process documented *(AI agent)*
- [ ] Create E2E manual testing checklist — QA can follow for regression testing *(AI agent draft + Human validate)*

## 3.6 Workstream: Testing Infrastructure

**Owner**: Dev team  
**Dependencies**: CI/CD pipeline  
**Estimated effort**: 3-4 days

- [ ] Create unit test examples — Reference tests for each layer *(AI agent)*
- [ ] Create integration test patterns — Reference tests with real DB *(AI agent)*
- [ ] Implement permission test matrix — Tests generated from `permissions.yaml` *(Human dev)*
- [ ] Create SAGA test patterns — Test compensation on failure *(AI agent pattern + Human implement)*
- [ ] Set up contract verification script — Daily check of external API schemas *(Human dev)*
- [ ] Deploy contract checker to webhost — Public endpoint for verification *(Human dev)*
- [ ] Enable Postgres slow query logging — Queries >100ms logged *(Hamzah)*
- [ ] Create smoke test script — Post-deployment health verification *(AI agent)*

## 3.7 Workstream: Security & Compliance

**Owner**: Hamzah  
**Dependencies**: None  
**Estimated effort**: 2 days

- [ ] Migrate secrets to AWS Secrets Manager — No hardcoded credentials in code *(Human dev)*
- [ ] Implement rate limiting at Gateway — Prevents runaway external API calls *(Human dev)*
- [ ] Audit OAuth2 token refresh for SATU SEHAT — Document current flow, identify gaps *(Human dev)*

---

# Part 4: Deferred Items (Future Backlog)

These items are acknowledged but not prioritized for immediate work:

- [ ] Database connection pooling audit — *Revisit if connection errors appear*
- [ ] Load testing with realistic data — *Revisit before major release*
- [ ] Graceful shutdown (SIGTERM handlers) — *Revisit before 24/7 operations*
- [ ] Synthetic test data strategy — *Revisit after core testing in place*
- [ ] Proto documentation generation — *Revisit when onboarding new devs*
- [ ] Mobile JKN integration — *Q3 roadmap; revisit after BAZNAS deal complete*

---

*Document Owner: Alex (CMO)*  
*Version: 2.0*  
*Last Updated: January 2025*  
*Status: Ready for iterative execution*
