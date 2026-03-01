# Incident Response

**Version:** 0.1 (stub)
**Date:** 01 March 2026
**Status:** Stub — content to be added
**Maintained by:** Hamzah (infrastructure owner), Alex

---

## 1. Severity Levels

1.1 Align with response time SLOs from [`wellmed-system-architecture.md §5.3`](../wellmed-system-architecture.md):

| Severity | Definition | Response target |
|----------|-----------|-----------------|
| P1 — Critical | Production down or data loss | Immediate |
| P2 — High | Major feature broken, >1000ms responses | Within 1 hour |
| P3 — Medium | Minor feature broken, degraded performance | Within 4 hours |
| P4 — Low | Cosmetic issue, no data risk | Next business day |

---

## 2. On-Call

- [ ] TODO: Document on-call rotation (who is on-call and when)
- [ ] TODO: Document how to reach the on-call engineer
- [ ] TODO: Document escalation path (engineer → Hamzah → Alex)

---

## 3. Incident Response Runbook

### 3.1 Initial Response

- [ ] TODO: Document triage steps (check CloudWatch, check service health endpoints, check DLQ depth)
- [ ] TODO: Document how to assess blast radius

### 3.2 Communication

- [ ] TODO: Document how to communicate an active incident to the team
- [ ] TODO: Document customer communication process (if tenant is affected)

### 3.3 Resolution

- [ ] TODO: Document hotfix deployment process (Red-level, CTO approval)
- [ ] TODO: Document rollback procedure

---

## 4. Post-Incident Review

- [ ] TODO: Document post-mortem template and process
- [ ] TODO: Define SLA for completing a post-mortem after a P1 or P2 incident

---

## 5. Key Observability Links

| Tool | Purpose | Link |
|------|---------|------|
| CloudWatch | Logs and metrics | TBD |
| AWS X-Ray | Distributed request traces | TBD |
| Service health endpoints | `/health` for each service | TBD |
| RabbitMQ management | DLQ depth monitoring | TBD |

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 01 Mar 2026 | Alex + Claude | Initial stub |
