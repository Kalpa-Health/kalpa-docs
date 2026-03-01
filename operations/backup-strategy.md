# Backup Strategy

**Version:** 0.1 (stub)
**Date:** 01 March 2026
**Status:** Stub — current backup table captured here; runbook detail to be added
**Maintained by:** Hamzah (infrastructure owner)

---

## 1. Current Backup Schedule

For the authoritative backup table, see [`wellmed-system-architecture.md §5.4`](../wellmed-system-architecture.md).

| Environment | Asset | Frequency | Retention |
|-------------|-------|-----------|-----------|
| Production | DB WAL | Continuous | 7 days |
| Production | DB snapshot | Daily | 14 days |
| Production | Full snapshot (app + frontend) | Weekly or pre-upgrade | Last 4 copies |
| Staging | DB snapshot | Weekly | 2 weeks |
| Staging | Config/structure | Git (IaC) | Forever |
| Dev | None | — | — |

---

## 2. Disaster Recovery Targets

| Metric | Target |
|--------|--------|
| RTO (Recovery Time Objective) | 4 hours |
| RPO (Recovery Point Objective) | 1 hour |

---

## 3. Recovery Procedures

- [ ] TODO: Document step-by-step database restore procedure from RDS snapshot
- [ ] TODO: Document step-by-step restore from WAL
- [ ] TODO: Document full application restore procedure
- [ ] TODO: Add contact information and escalation path
- [ ] TODO: Document quarterly DR test schedule

---

## 4. Multi-Tenant Considerations

- [ ] TODO: Document how per-tenant database backups are organized and named
- [ ] TODO: Document how to restore a single tenant's database without affecting others

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 01 Mar 2026 | Alex + Claude | Stub with current backup table from system architecture |
