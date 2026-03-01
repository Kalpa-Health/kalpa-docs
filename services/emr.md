# Service: EMR

**Version:** 0.1 (stub)
**Date:** 01 March 2026
**Status:** Stub — content to be added
**Maintained by:** TBD (service owner)

---

## 1. Overview

1.1.1 The EMR (Electronic Medical Record) service owns the core clinical data domain: patient visits, medical records, diagnoses, examinations, and referrals. It is part of the Lite tier and present in all WellMed deployments.

1.1.2 For the full system context, see [`wellmed-system-architecture.md`](../wellmed-system-architecture.md).

---

## 2. Repository

| Item | Value |
|------|-------|
| **Repo** | TBD |
| **Language** | Go |
| **Tier** | Lite (included in all deployments) |

---

## 3. Key Responsibilities

- [ ] TODO: Document EMR domain ownership
- Patient visit lifecycle (create, update, complete, cancel)
- Medical records and SOAP notes
- Diagnoses (ICD-10 coding)
- Examinations and clinical pathology
- Referrals (internal and external)
- Async sync to SATU SEHAT via RabbitMQ

---

## 4. External Dependencies

- **SATU SEHAT Integration** — Visit data synced asynchronously via RabbitMQ queue `satu_sehat_sync`
- **Cashier** — gRPC sync call to create invoice on visit completion

---

## 5. Open Questions

- Does EMR route through Backbone long-term, or does it get its own direct gRPC connection from Gateway? (See [`QUESTIONS.md §4.5`](../QUESTIONS.md))

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 01 Mar 2026 | Alex + Claude | Initial stub |
