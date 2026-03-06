# ADR-009: Canonical Archive Pattern

**Status:** Accepted
**Date:** 2026-03-06
**Author:** Alex
**Reviewers:** Hamzah (CTO)

---

## 1. Context

1.1 WellMed operates a microservice architecture where clinical and financial data is produced across multiple services (consultation, cashier, backbone, and others) during the lifecycle of a patient visit. At any point during an active visit, this data is mutable, distributed, and service-owned. There is currently no single authoritative, immutable record that represents the closed state of either a clinical encounter or a financial transaction.

1.2 Two distinct archival requirements have been identified:

- **Medical records** — a closed, tamper-evident record of what happened clinically during a visit, suitable for legal, regulatory, and continuity-of-care purposes.
- **Financial records** — a closed, tamper-evident record of what was charged, what was applied, and how an invoice was settled, suitable for audit, reporting, and future payer/consumer integrations.

1.3 WellMed integrates with SATU SEHAT (Indonesia's national health data platform, FHIR R4 profile) for clinical data submission and is subject to Indonesia's emerging PEPPOL e-invoicing requirements for financial records. However, both external systems have significant limitations that make them unsuitable as internal sources of truth:

- **SATU SEHAT:** Patient identity matching failures (KTP errors, duplicates), duplicate location records with inconsistent spelling, no support for foreign patients. SATU SEHAT data quality cannot be relied upon for the canonical record.
- **PEPPOL/DJP e-Faktur:** Still maturing in Indonesia; the canonical financial record must not be coupled to the current state of external compliance requirements.

1.4 WellMed serves a significant population of foreign patients (expats, tourists) in Bali who have no Indonesian national ID (NIK/KTP) and therefore cannot be submitted to SATU SEHAT. These patients must be fully supported in WellMed's canonical model regardless of their submittability to external systems.

1.5 The system needs a clear boundary between **operational data** (mutable, service-owned, in-flight) and **archived data** (immutable, canonical, closed). Without this boundary, services will couple to each other's mutable state, reporting becomes unreliable, and audit trails are difficult to guarantee.

---

## 2. Decision

### 2.1 The Canonical Archive Pattern

Two immutable archives are established, each owned by a dedicated canonical service:

| Domain | Canonical Service | Immutable Archive | Standard Basis |
|---|---|---|---|
| Clinical / Visit | `CanonicalVisitService` | `immutable_visit` | FHIR R4 concepts |
| Financial / Transaction | `CanonicalTransactionService` | `immutable_transaction` | UBL 2.x concepts |

Each archive is **append-only and write-once per saga closure**. No service other than the owning canonical service may write to these tables.

### 2.2 Write Conditions

An archive record is written **only when the saga that produced it closes**. In-progress sagas — regardless of how much activity has occurred — do not produce an archive record.

| Archive | Write Trigger | Saga Close Event |
|---|---|---|
| `immutable_visit` | Doctor sign-off | `CreateVisit` saga completes with practitioner signature |
| `immutable_transaction` | Invoice settlement | Payment application brings invoice to `SETTLED` or `WRITTEN_OFF` |

This means:
- A SATU SEHAT Encounter may exist and be active while a visit is in progress. The `immutable_visit` record is not written until sign-off.
- An open invoice may have partial payments applied. The `immutable_transaction` record is not written until the invoice reaches a terminal status.

### 2.3 Standards as Conceptual Models, Not Storage Formats

FHIR R4 and UBL 2.x are used as **conceptual models** for the archive schema — their resource types, field names, and relationships inform the data structure. They are **not** stored as raw FHIR JSON or UBL XML internally.

This separation means:
- The archive schema is stable and queryable as relational data.
- SATU SEHAT and PEPPOL-specific profiles, extensions, and quirks are absorbed by **output adapters** at submission time — not baked into the archive.
- Schema migrations are under WellMed's control, not coupled to external standard revisions.

### 2.4 External Identity Mapping

The canonical model uses WellMed-internal IDs for all entities. Mappings to external system identifiers are maintained in dedicated mapping tables, separate from the archive.

**Patient identity mapping:**

```
patient_identity
  id
  canonical_patient_id
  identity_type        -- NIK | PASSPORT | KITAS | KITAP
  identity_number
  satusehat_ihs_id     -- nullable
  match_status         -- UNMATCHED | MATCHED | FAILED | SKIPPED_FOREIGN | PENDING_REVIEW
  last_attempted_at
  failure_reason
```

Foreign patients: `identity_type = PASSPORT`, `match_status = SKIPPED_FOREIGN`. No retry, no error. When Indonesia adds foreign patient support, a mapping record is added and retroactive submission becomes possible — the canonical archive record was always complete.

**Location and practitioner mappings follow the same pattern:** canonical ID as primary key, SATU SEHAT ID nullable, match status explicit.

**Financial identity mapping:**

```
party_tax_identity
  id
  canonical_party_id   -- patient, corporate payer, or clinic
  party_type           -- PATIENT | PAYER | SUPPLIER
  npwp                 -- nullable (foreign patients may not have one)
  peppol_endpoint_id   -- nullable
  tax_status           -- TAXABLE | EXEMPT | FOREIGN | PENDING_REVIEW
```

### 2.5 Invoice as the Atomic Financial Unit

The canonical financial unit is the **Invoice**. A "bill" for a simple cash visit is an Invoice with `PaymentMeans = cash` and `PaymentTerms = immediate` — there is no separate "bill" type.

Invoice settlement status is derived from the sum of payment applications:

| Status | Meaning |
|---|---|
| `OPEN` | No payment applied |
| `PARTIAL` | Some payment applied, balance remaining |
| `SETTLED` | Fully covered by any combination of payment types |
| `WRITTEN_OFF` | Remaining balance credited/waived |
| `VOIDED` | Cancelled before settlement |

WellMed owns the settlement status. Dispute resolution, AR aging, and payer negotiation are **accounting concerns** — their outcome is received by WellMed as a payment application or credit note, not modelled internally.

Payment application types against an invoice:
- Cash / card payment
- Insurance / corporate payer payment
- Credit note applied
- Wallet or deposit draw-down _(WellMed records the application event only; balance ledger lives in external finance/accounting)_

### 2.6 Supersession Rules

The immutable archive never updates or deletes. Post-write mutations are handled through supersession: a new record is created that references the original, and the original is marked as superseded.

```
Record A  →  superseded_by: Record B ID  →  status: SUPERSEDED
Record B  →  supersedes: Record A ID     →  status: ACTIVE
```

The current truth is always the latest non-superseded record. All history is preserved.

**2.6.1 Visit supersession — addendum only (v1)**

Post-sign-off, only an addendum is permitted. The successor record contains identical clinical data to the original, with an addendum annotation appended.

```
immutable_visit (Record B)
  supersedes:       Record A ID
  clinical_data:    identical to Record A
  addendum:         { note, author_id, timestamp }
  status:           ACTIVE
```

This is an administrative annotation, not a clinical correction. In FHIR terms this corresponds to `relatesTo.code = appends`, not `replaces`.

**True clinical supersession** (reopening a signed visit to change clinical content) is **structurally supported** but **not permitted in v1**. Any future implementation requires an explicit authorization workflow (approver identity, reason code, audit trail). The ADR constraint is: _addendum is the only permitted post-sign-off operation in v1._

**2.6.2 Transaction supersession — correction permitted**

Invoice corrections are legitimate business events. The successor record may contain different content from the original (corrected amounts, line items, tax values).

In UBL/PEPPOL terms: a CreditNote is issued to zero the original Invoice, and a new Invoice is issued. Both reference the original via `BillingReference`. In the archive: the original record is marked `SUPERSEDED`, the successor record carries `supersedes` and `BillingReference` to the original.

### 2.7 Service Ownership Evolution

`wellmed-cashier` (formerly `wellmed-pos`) currently owns all transaction data. As the system evolves:

- Item catalog will become its own microservice (via Backbone)
- Payer / consumer / wallet features will be added
- Reporting and analytics will read from `immutable_transaction`

The canonical archive is designed to support these readers now. When services are extracted, they read from the immutable archive rather than coupling to cashier's mutable internal state.

---

## 3. Terminology Mapping

### 3.1 WellMed → FHIR R4

| WellMed Term | FHIR Resource / Concept |
|---|---|
| Visit | Encounter |
| Result / Finding | Observation |
| Diagnosis | Condition |
| Doctor / Clinician | Practitioner |
| Clinic / Room | Location |
| Patient | Patient |
| Prescription | MedicationRequest |
| Procedure / Treatment | Procedure |
| Referral | ServiceRequest |
| Addendum | DocumentReference (`relatesTo.code = appends`) |
| Clinical supersession | Encounter (`replaces`) |

### 3.2 WellMed → UBL 2.x

| WellMed Term | UBL Concept |
|---|---|
| Bill / Invoice | Invoice |
| Line item (service or product) | InvoiceLine |
| Payment event | PaymentMeans |
| Payment application | BillingReference + applied amount |
| Patient (cash payer) | AccountingCustomerParty |
| Insurance / corporate payer | PayerParty |
| Write-off / credit | CreditNote |
| Deposit draw-down | PaymentMeans (type = deposit) |
| Voided invoice | Invoice superseded via CreditNote + new Invoice with BillingReference |
| Tax exemption (medical) | TaxCategory (`E` — exempt) |
| Foreign patient (no NPWP) | AccountingCustomerParty with no TaxScheme |

---

## 4. Consequences

### 4.1 Positive

- **Single source of truth per domain** — downstream services (reporting, payers, external integrations) read from one stable, immutable record
- **External systems decoupled** — SATU SEHAT and PEPPOL failures, data quality issues, or profile changes do not affect the canonical archive
- **Foreign patients fully supported** — no KTP/NIK required for a complete canonical record; SATU SEHAT submission is an optional, asynchronous adapter concern
- **Audit trail by design** — supersession preserves full history; no soft-deletes or update-in-place
- **PEPPOL-ready** — UBL-concept schema positions WellMed for Indonesian e-invoicing compliance without a data model rework
- **Reporting stability** — analytics and finance reporting read from an immutable table; no risk of mutable operational data affecting historical numbers

### 4.2 Negative

- **Two canonical services to build and maintain** — `CanonicalVisitService` and `CanonicalTransactionService` are new infrastructure
- **Deferred record creation** — the archive record does not exist until saga closure; reporting on in-progress visits/transactions requires reading from mutable service data
- **Supersession complexity** — any UI or API that displays a record must resolve the supersession chain to show the current version

### 4.3 Risks

- **Saga never closes** — a visit that is never signed off or an invoice that is never settled produces no archive record. **Mitigation:** Backbone saga monitoring and timeout escalation (defined in ADR-005) will surface stalled sagas. A periodic reconciliation job should flag visits/invoices open beyond expected duration.
- **Mapping table staleness** — if SATU SEHAT IHS IDs or NPWP data changes externally, mapping tables may become stale. **Mitigation:** Match status field with `last_attempted_at` enables periodic re-validation without blocking operations.
- **CanonicalTransactionService scope creep** — as payer, wallet, and catalog features are added, pressure may arise to add business logic to the canonical service. **Mitigation:** The canonical service is a writer and schema owner only — it must not absorb business rules from cashier or other modules.

---

## 5. Alternatives Considered

| Alternative | Pros | Cons | Why Not Chosen |
|---|---|---|---|
| Use SATU SEHAT as the medical record source of truth | Reduces duplication | Data quality failures, no foreign patient support, cannot be trusted as source of truth | Violates data integrity requirements for a significant portion of the patient population |
| Store raw FHIR JSON / UBL XML natively | Standards-compliant storage, easier export | Verbose, not queryable, schema migrations require FHIR/UBL version management, performance poor for reporting | Standards as conceptual model only; adapters handle serialization at output boundary |
| Write archive record at visit creation, update at sign-off | Archive exists earlier, easier partial reporting | Violates immutability; forces update-in-place; archive loses meaning | Immutability is non-negotiable for audit and legal compliance |
| Separate ADRs for visit and transaction archives | Simpler individual documents | The pattern is identical; separate ADRs risk divergent implementations of the same architecture | Single ADR with two instances of the same pattern ensures consistent implementation |

---

## 6. Implementation Notes

6.1 **CanonicalVisitService** is the sole authorized writer to `immutable_visit`. It is triggered by the `DoctorSignOff` step in the `CreateVisit` saga (see ADR-005, ADR-006).

6.2 **CanonicalTransactionService** is the sole authorized writer to `immutable_transaction`. It is triggered when an invoice transitions to `SETTLED` or `WRITTEN_OFF` in `wellmed-cashier`, communicated via the saga callback pattern.

6.3 **Supersession fields** required on both archive tables:
```
supersedes_id       UUID nullable  -- points to the record this one replaces
superseded_by_id    UUID nullable  -- points to the successor record
supersession_type   ENUM           -- ADDENDUM | CORRECTION | VOID
superseded_at       TIMESTAMPTZ nullable
archive_status      ENUM           -- ACTIVE | SUPERSEDED
```

6.4 **Queries must always filter** `archive_status = ACTIVE` unless explicitly retrieving history. Helper views or repository methods should enforce this by default.

6.5 **SATU SEHAT adapter** reads from `immutable_visit` + `patient_identity` + `practitioner_satusehat_mapping` + `location_satusehat_mapping` to build the FHIR R4 bundle. It does not read from mutable visit state in `wellmed-consultation`. Submission is asynchronous and its failure does not affect the canonical record.

6.6 **PEPPOL/e-Faktur adapter** (future) reads from `immutable_transaction` + `party_tax_identity` to build the UBL Invoice document. Same principle: adapter failure does not affect the canonical record.

6.7 **True clinical supersession** (v1 constraint): any code path that attempts to create a superseding visit record with different clinical content must be rejected at the service layer with error code `CLINICAL_SUPERSESSION_NOT_PERMITTED`. This constraint is intentional and must not be bypassed without a formal ADR amendment defining the authorization workflow.

---

---

## A.1 Amendment: Pharmacist Dispense Visit (2026-03-06)

**Context:** The pharmacy extraction (ADR-010) introduced an `AtomicPharmacistVisit` saga. When a pharmacist confirms a dispense for a registered patient with no open clinical visit saga, backbone must write a record to the patient file. This record is not a clinical consultation — it is a dispense event — but it is still a canonical visit record in the sense that it is immutable, patient-linked, and audit-relevant.

**Amendment:**

A.1.1 `immutable_visit` gains a `visit_type` field (VARCHAR, NOT NULL, default `CLINICAL`):

| Value | Meaning | Write Path |
|-------|---------|------------|
| `CLINICAL` | Standard clinical encounter, doctor-signed | `CreateVisit` saga, doctor sign-off step (§2.2 — unchanged) |
| `PHARMACIST_DISPENSE` | Pharmacist-confirmed dispense, no open clinical visit | `AtomicPharmacistVisit` saga, pharmacist dispense confirmation |

A.1.2 The `CLINICAL` write path is **completely unchanged**. A pharmacist cannot write a `CLINICAL` record. The `PHARMACIST_DISPENSE` record has null `diagnosis_codes` and `treatment_summary` is set to `"pharmacist_dispense:{dispense_id}"`.

A.1.3 `PHARMACIST_DISPENSE` records exist to create an audit trail in the patient file when medication is dispensed outside of an active clinical visit saga. This covers: OTC dispenses for registered patients, post-visit dispense where the clinical visit saga has already closed.

A.1.4 The `AtomicPharmacistVisit` saga is **only triggered** by backbone's `PharmacySale.RouteDispenseResult` local step when:
- `patient_id` is set on the pharmacy sale (non-anonymous patient)
- No `in_flight` saga exists for that patient

A.1.5 If an `in_flight` clinical visit saga exists for the patient at dispense time, the dispense is attached to that saga's context (`dispense_id` set as nullable context key). No separate `PHARMACIST_DISPENSE` record is written — the clinical visit's `immutable_visit` will be written at doctor sign-off per §2.2.

A.1.6 Migration: `canonical_visit/migrations/002_add_visit_type.sql`. Existing rows default to `CLINICAL`.

---

## 7. References

- [`ADR-005-saga-orchestration.md`](./ADR-005-saga-orchestration.md)
- [`ADR-006-domain-service-boundaries.md`](./ADR-006-domain-service-boundaries.md)
- [FHIR R4 Encounter Resource](https://hl7.org/fhir/R4/encounter.html)
- [OASIS UBL 2.1 Invoice](https://docs.oasis-open.org/ubl/os-UBL-2.1/UBL-2.1.html)
- [PEPPOL BIS Billing 3.0](https://docs.peppol.eu/poacc/billing/3.0/)
- [SATU SEHAT API — Staging](https://api-satusehat-stg.kemkes.go.id)

---

# Edit Log

| Version | Date | Author | Changes |
|---|---|---|---|
| 0.1 | 06 Mar 2026 | Alex + Claude | Initial draft — captured from architecture alignment session covering canonical archive pattern, supersession rules, FHIR/UBL conceptual model approach, foreign patient support, and operational/accounting boundary |
| 0.2 | 06 Mar 2026 | Alex + Claude | Amendment A.1: added `visit_type` field (CLINICAL / PHARMACIST_DISPENSE) to `immutable_visit`; defined `AtomicPharmacistVisit` as secondary edge-case write path; doctor sign-off path unchanged. ADR-010 cross-reference. |
