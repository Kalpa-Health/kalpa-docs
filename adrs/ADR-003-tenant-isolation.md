# ADR-003: Tenant Isolation via Separate Databases

**Status:** Accepted
**Date:** 2025-01-20
**Author:** Alex
**Reviewers:** Raka (CTO), Hamzah (infrastructure)

> **Note:** This ADR documents a decision that was made and confirmed. The stub is pending expansion with operational details. See [`wellmed-system-architecture.md §4.1`](../wellmed-system-architecture.md) for the current architecture description.

---

## 1. Context

1.1 WellMed is a multi-tenant SaaS — multiple clinics and hospitals share the same codebase and infrastructure. We need a tenant isolation strategy that prevents data leakage between tenants and eliminates accidental cross-tenant queries.

1.2 The most common multi-tenant database strategy is a shared database with a `tenant_id` column on every table. This is simple to set up but requires every query to include a tenant filter — a single missing WHERE clause can leak data across tenants.

---

## 2. Decision

2.1 Use **separate PostgreSQL databases per tenant**. Each clinic/hospital gets its own database (e.g., `tenant_clinic_a_db`). Within each tenant database, each microservice gets its own schema (e.g., `emr`, `cashier`, `pharmacy`).

2.2 Tenant context is extracted from JWT claims at the Gateway (`tenant_db`, `tenant_schema`) and propagated to all downstream services via gRPC metadata. Services connect to the correct tenant database based on these values.

2.3 HQ has its own standalone database (`hq_db`) separate from all tenant databases.

---

## 3. Consequences

## 3.1 Positive

- **Complete data isolation** — A query in `tenant_clinic_a_db` physically cannot return data from another tenant. No application-level tenant filtering is needed.
- **Simpler queries** — Developers don't need to remember to add `WHERE tenant_id = ?` to every query
- **Independent backup and restore** — Each tenant database can be backed up and restored independently
- **Compliance** — Easier to demonstrate data isolation to enterprise customers

## 3.2 Negative

- **Operational complexity** — More databases to manage, monitor, and back up as the tenant count grows
- **Schema migrations** — Running migrations across all tenant databases requires tooling (run migration on each tenant DB sequentially or in parallel)
- **Cross-tenant analytics** — Aggregate reporting across all tenants requires querying multiple databases

## 3.3 Risks

- Migration orchestration at scale — **mitigation:** migration tooling must be built before tenant count becomes large
- [ ] TODO: Document the migration orchestration approach for running schema migrations across all tenant databases

---

## 4. Alternatives Considered

| Alternative | Pros | Cons | Why Not Chosen |
|-------------|------|------|----------------|
| Shared DB, `tenant_id` column | Simple, single schema migration | Every query needs tenant filter; one missing WHERE leaks data; audit burden | Too risky for healthcare data; developer error too easy |
| Schema per tenant (same DB) | Easier to manage than separate DBs | Still shares DB resources; row-level security complexity | Insufficient isolation for enterprise compliance |

---

## 5. Implementation Notes

5.1 [ ] TODO: Document the migration orchestration tooling approach.

5.2 [ ] TODO: Document how cross-tenant analytics queries work (direct DB queries from HQ service? ElasticSearch aggregates?).

5.3 The `tenant_db` and `tenant_schema` values in the JWT claims are the source of truth for routing database connections. Services must never infer or hardcode these values.

---

## 6. References

- [`wellmed-system-architecture.md §4.1`](../wellmed-system-architecture.md)

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 01 Mar 2026 | Alex + Claude | Stub — decision captured, operational details pending |
