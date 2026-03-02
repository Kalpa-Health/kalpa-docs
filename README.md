# kalpa-docs

**Maintained by:** Kalpa Inovasi Digital engineering team
**Last updated:** 02 March 2026

Cross-service documentation for the WellMed platform. This repo contains no application code — only markdown documents, diagrams, and ADRs.

For guidance on what lives here vs. in service repos, see [`repo-governance.md`](repo-governance.md).

---

## 1. Start Here

| Document | What it answers |
|----------|----------------|
| [`wellmed-system-architecture.md`](wellmed-system-architecture.md) | What is WellMed, how do the services fit together, what tech do we use |
| [`repo-governance.md`](repo-governance.md) | What documentation lives here vs. in service repos |
| [`QUESTIONS.md`](QUESTIONS.md) | Open architectural questions and anomalies flagged for team discussion |

---

## 2. Testing & Quality

| Document | What it answers |
|----------|----------------|
| [`wellmed-testing-architecture.md`](wellmed-testing-architecture.md) | Testing values, patterns, coverage targets, CI/CD configuration |

---

## 3. Architecture Decision Records

ADRs document significant architectural decisions — why we made them, what alternatives we considered, and what the consequences are.

| ADR | Title | Status |
|-----|-------|--------|
| [`adrs/ADR-000-template.md`](adrs/ADR-000-template.md) | Template | — |
| [`adrs/ADR-001-api-client-circuit-breaker.md`](adrs/ADR-001-api-client-circuit-breaker.md) | API Client Pattern with Circuit Breaker | Accepted |
| [`adrs/ADR-002-saga-orchestrator.md`](adrs/ADR-002-saga-orchestrator.md) | SAGA Orchestrator Pattern | Stub |
| [`adrs/ADR-003-tenant-isolation.md`](adrs/ADR-003-tenant-isolation.md) | Tenant Isolation via Separate Databases | Stub |

---

## 4. Service Reference

Per-service overview documents (authored here, linked from service repos).

| Service | Document | Repo |
|---------|----------|------|
| Gateway | [`services/gateway.md`](services/gateway.md) | [wellmed-gateway-go](https://github.com/Kalpa-Health/wellmed-gateway-go) |
| Backbone | [`services/backbone.md`](services/backbone.md) | TBD |
| EMR | [`services/emr.md`](services/emr.md) | TBD |

---

## 5. Operations

| Document | What it covers |
|----------|----------------|
| [`operations/backup-strategy.md`](operations/backup-strategy.md) | Backup schedules, retention, disaster recovery |
| [`operations/database-migrations.md`](operations/database-migrations.md) | Migration process, review gates, rollback procedures |
| [`operations/incident-response.md`](operations/incident-response.md) | On-call runbook, escalation paths, post-mortems |

---

## 6. Development Guides

| Document | What it covers |
|----------|----------------|
| [`development/setup.md`](development/setup.md) | Local development environment setup |
| [`development/conventions.md`](development/conventions.md) | Code conventions, naming, project structure |
| [`development/testing-patterns.md`](development/testing-patterns.md) | How to write tests (quick reference with examples) |
| [`development/ci-cd-guide.md`](development/ci-cd-guide.md) | GitHub Actions workflows explained, new repo bootstrap |

---

## 7. Infrastructure & Tooling

Infrastructure configuration and org-wide scripts live in `wellmed-infrastructure` (currently `infrastructure/` locally, pending GitHub repo creation).

| Resource | What it covers |
|----------|----------------|
| [`HOW-TO.md`](https://github.com/Kalpa-Health/wellmed-infrastructure/blob/main/HOW-TO.md) | How to bootstrap a new repo to org standard (start here) |
| `scripts/bootstrap-repo.sh` | Creates branches, pushes CI workflows, applies branch protection |
| `templates/workflows/` | Source-of-truth CI workflow templates for all service repos |

For governance rules on what lives in infrastructure vs service repos vs kalpa-docs, see [`repo-governance.md §3.2`](repo-governance.md).

---

## 8. Diagrams

| File | What it shows |
|------|--------------|
| [`diagrams/system-topology.mermaid`](diagrams/system-topology.mermaid) | High-level system topology (all layers) |

---

## 9. Archive

[`archive/`](archive/) contains superseded document versions kept for historical reference. Do not link to archive documents from new content — link to the canonical versions above instead.
