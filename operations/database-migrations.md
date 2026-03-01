# Database Migrations

**Version:** 0.1 (stub)
**Date:** 01 March 2026
**Status:** Stub — content to be added
**Maintained by:** Hamzah (infrastructure), Fajri (senior dev)

---

## 1. Overview

1.1.1 Database migrations at WellMed require special care because of the multi-tenant architecture. A migration must run against every tenant database, not just a single database. This makes migrations a Yellow-level change requiring senior developer review before merging.

1.1.2 For the merge governance rules, see [`wellmed-system-architecture.md §7.2`](../wellmed-system-architecture.md).

---

## 2. Migration Tool

- [ ] TODO: Document which migration tool is used (current status?)
- [ ] TODO: Document migration file naming convention
- [ ] TODO: Document how to create a new migration

---

## 3. Running Migrations

### 3.1 Staging

- [ ] TODO: Document how migrations are applied to staging tenant databases

### 3.2 Production

- [ ] TODO: Document the production migration process — who runs it, when, how to verify

### 3.3 Multi-Tenant Orchestration

- [ ] TODO: Document how migrations are run across all tenant databases (sequential? parallel? tooling?)

---

## 4. Rollback

- [ ] TODO: Document rollback procedure for a failed migration
- [ ] TODO: Document what to do if a migration partially succeeds across tenants

---

## 5. Review Checklist (Yellow-Level)

Before merging any migration PR, the senior reviewer must verify:

- [ ] Migration is idempotent (safe to run twice)
- [ ] Rollback migration exists
- [ ] Migration has been tested against a copy of production data on staging
- [ ] No locking operations that could cause downtime (e.g., full table rewrites on large tables)
- [ ] The migration is backward-compatible with the currently deployed application version

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 01 Mar 2026 | Alex + Claude | Initial stub |
