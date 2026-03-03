# Laravel → Go Pattern Map: What Transfers, What Doesn't

**Version:** 1.0
**Date:** 03 March 2026
**For:** CTO and senior engineers
**Tone:** This isn't a code review — it's a map. Laravel is a well-designed framework and the decisions that led to the current patterns were rational. The goal here is to identify which Laravel defaults need to be actively replaced as we grow, and what the Go/PostgreSQL equivalent looks like.

---

# 1. The Core Shift: Document Model vs. Relational + Search

## 1.1 What the current system does

Laravel's Eloquent makes deep eager loading trivial:

```php
// One line, returns patient → visit → examination → assessment → medications
$transaction->load('consument.visitReference.visitPatient.patient.people.cardIdentity');
```

The API response becomes a fully self-contained document — everything the frontend might need, nested together. This is the **document model**: responses are designed to be read in a single shot, like a MongoDB document.

The problem is that the data lives in a relational database, not a document store. So you end up with the same patient's name stored in:

- `peoples.name` — the source
- `consuments.props.name` — a snapshot from when the consument was created
- `transaction_has_consuments.props.name` — a snapshot from when the transaction was created
- `invoices` response → `payer.name` — another snapshot
- The search index — another projection

When you update `peoples.name`, 4 other representations don't update. The frontend team doesn't know which one is authoritative.

## 1.2 What the target looks like

Two distinct systems with one clear job each:

```
PostgreSQL (normalized, source of truth)
  ├── peoples: id, name, nik, dob, sex, ... (typed columns, FK constraints)
  ├── patients: id, people_id FK, medical_record, ...
  ├── transactions: id, patient_id FK, visit_registration_id FK, ...
  └── billings: id, transaction_id FK, amount, status, ...

OpenSearch (denormalized, read projections)
  └── transaction_index: {id, patient_name, patient_nik, visit_date, amount, status, ...}
      → updated by event/queue when any source record changes
      → used for LIST endpoints, search, filters
      → never used for writes
```

**Write path:** client → gateway → backbone → PostgreSQL → event → search index update
**Read (list/search):** client → gateway → search index
**Read (single entity):** client → gateway → backbone → PostgreSQL (JOIN on explicit columns)

The frontend gets fast, rich list responses from the search index. Single-entity detail pages query normalized data. Neither path needs props.

---

# 2. Priority 1 — Stop Embedding, Start Referencing (Architectural)

## 2.1 The pattern

Current backbone responses for entity-heavy modules embed full sub-objects:

```
Invoice response (actual structure):
  invoice.id, invoice.invoice_code
  invoice.payment_summary (total, debt, discount, ...)
  invoice.author (id, name)
  invoice.billing.id
  invoice.billing.billing_code
  invoice.billing.author (id, name)       ← Author appears again
  invoice.billing.cashier (id, name)
  invoice.payer.id
  invoice.payer.name
  invoice.payer.payment_summary (...)     ← PaymentSummary appears again
  invoice.payer.user_wallet (...)
```

PaymentSummary appears twice in one invoice response. Author appears twice. This is not a Go problem or a gateway problem — it's a proto contract design problem. The backbone is serializing the document model into gRPC messages.

## 2.2 The Go/PostgreSQL pattern

DB schema owns the relationships via foreign keys. API responses carry IDs, not embedded objects:

```go
// What the invoice response should look like
type InvoiceResponse struct {
    ID            string    `json:"id"`
    InvoiceCode   string    `json:"invoice_code"`
    BillingID     string    `json:"billing_id"`      // ← reference, not embed
    PatientID     string    `json:"patient_id"`      // ← reference, not embed
    AuthorID      string    `json:"author_id"`       // ← reference, not embed
    Amount        int64     `json:"amount"`
    Debt          int64     `json:"debt"`
    Status        string    `json:"status"`
    CreatedAt     time.Time `json:"created_at"`
}
```

If the frontend needs author name alongside the invoice, that's a join — either done at the query layer or pre-projected in the search index. It's not embedded in the response.

**Why this matters for data integrity:** When you embed, you snapshot. When you reference, you always read current data. A patient name change should propagate to invoice displays automatically. With embedded snapshots, it doesn't.

## 2.3 The exception: deliberate audit snapshots

There is one valid reason to embed and snapshot: **billing at time of service**. SATU SEHAT requires that you record what the patient's NIK, address, and insurance status were *at the time of the visit*, not what they are today. If someone moves or changes their BPJS number, the historical billing record shouldn't change.

This is valid — but it should be **explicit**, not a side effect of using `interface{}`:

```go
// GOOD: Explicit audit snapshot with a defined type
type BillingPatientSnapshot struct {
    PatientID     string `json:"patient_id"`      // FK for reconciliation
    NIK           string `json:"nik"`             // captured at billing time
    Name          string `json:"name"`            // captured at billing time
    BPJSNumber    string `json:"bpjs_number"`     // captured at billing time
    SnapshotAt    time.Time `json:"snapshot_at"`  // when was this captured
}

// BAD: Opaque blob that might or might not be a snapshot
Consument interface{} `json:"consument"`
```

The snapshot struct goes in a typed column (`billing_patient_snapshot jsonb` with a JSON schema comment), not in a catch-all `props`.

---

# 3. Priority 2 — Graduate `props` Fields Selectively (DB Design)

## 3.1 The Laravel origin

`$casts = ['props' => 'array']` in an Eloquent model means "this field is a JSON blob, cast to/from PHP array automatically." It was a productivity tool — add fields without migrations during rapid iteration. At 49 tables with `props` columns, the flexibility window has closed.

## 3.2 The decision framework

For each `props` column, ask three questions:

**Q1: Is this field searched or filtered?**
```sql
WHERE props->>'some_field' = ?
```
If yes → it must become a typed column with an index. Querying JSON is slower than querying a column and can't use a standard B-tree index without explicit extraction.

**Q2: Is this field structurally stable?** (Does it have the same keys for every row?)
If yes → it should be a typed column or a typed jsonb struct with a schema comment.
If no (genuinely variable per row, like feature flags or UI preferences) → it can stay as jsonb, but needs a documented schema.

**Q3: Does SATU SEHAT or any external system read or write this field?**
If yes → typed column, non-nullable, indexed. No exceptions. Government integrations cannot depend on JSON parsing.

## 3.3 Prioritized graduation list

Based on the 49 tables, suggested order:

**Tier 1 — Fix before next SATU SEHAT submission:**

| Table | Field to promote | Why |
|-------|-----------------|-----|
| `peoples` | `nik`, `bpjs_number` (from props) | SATU SEHAT patient matching |
| `submissions` | All fields describing the submission payload | Audit, regulatory compliance |
| `transactions` | `reference_type`, `reference_id` (if in props) | Core business logic |
| `billings` | `payment_status`, `amount_paid` (if in props) | Financial integrity |

**Tier 2 — Fix in next DB iteration:**

| Table | Action |
|-------|--------|
| `transaction_items` | Promote `item_id`, `quantity`, `price` to columns |
| `payment_details` | Promote `payment_method_id`, `amount` to columns |
| `peoples` | Promote `address_ktp_id`, `address_residence_id` to FK columns |

**Tier 3 — Document and leave:**

| Table | Action |
|-------|--------|
| `unicodes` | Document what keys exist in props per unicode type |
| `activities` | Activity log — props as arbitrary event data is valid; add GIN index |
| `config_props` | Configuration by design — document the schema in code |
| Geographic tables (provinces, districts, villages) | These are rarely queried on specific prop keys; jsonb is fine, add GIN index if needed |

## 3.4 The PostgreSQL tooling available

You're not choosing between "rigid columns" and "flexible props." PostgreSQL gives you a spectrum:

```sql
-- Option 1: Typed column (fully queryable, indexed, constrained)
ALTER TABLE peoples ADD COLUMN nik VARCHAR(16) NOT NULL;
CREATE UNIQUE INDEX idx_peoples_nik ON peoples(nik);

-- Option 2: jsonb with GIN index (flexible, queryable on any key)
CREATE INDEX idx_unicodes_props ON unicodes USING GIN(props);
-- Now: WHERE props @> '{"category": "drug"}' uses the index

-- Option 3: jsonb with extracted generated column (best of both)
ALTER TABLE peoples
  ADD COLUMN nik VARCHAR(16) GENERATED ALWAYS AS (props->>'nik') STORED;
CREATE UNIQUE INDEX idx_peoples_nik ON peoples(nik);
-- No migration to the app layer needed; extraction is automatic
```

Option 3 (generated columns) is the least disruptive migration path — you get a typed, indexed, queryable column without changing the application code that writes to `props`. It's a pure DB-layer change and can be done incrementally.

---

# 4. Priority 3 — Fix the Proto Contracts (Protocol Design)

## 4.1 The `data: string` problem

Several modules have proto contracts like this:

```protobuf
// visit_examination.proto — actual current state
message ListResponse {
  repeated VisitExamination visit_examinations = 1;
  int32 page = 2;
  int32 per_page = 3;
  int32 total = 4;
  string data = 5;   // ← JSON string carrying the actual payload
}
```

The `string data` field means the backbone is serializing a JSON object into a string, sending it over gRPC, and the gateway is `json.Unmarshal`-ing it back. Protobuf's type safety is completely bypassed. The gateway cannot inspect, validate, or transform the response without parsing an opaque string.

This is the root cause of the `var response interface{}` pattern in 8 handler files. The gateway isn't being lazy — it genuinely cannot do better without the backbone changing the proto.

## 4.2 The fix

Define the actual message structure in proto. Replace every `string data` field that carries JSON with a properly typed message:

```protobuf
// Target: proper typed proto contract
message VisitExamination {
  string id = 1;
  string visit_registration_id = 2;
  string status = 3;
  string props = 4;              // ← Still exists for now, but the outer shape is typed
  google.protobuf.Timestamp created_at = 5;
  google.protobuf.Timestamp updated_at = 6;
}

message ListResponse {
  repeated VisitExamination data = 1;   // ← typed list, not string
  int32 page = 2;
  int32 per_page = 3;
  int32 total = 4;
}
```

Once the backbone proto is typed, the gateway can use typed structs, the handler goes from 30 lines to 10 lines, and Go's compiler catches any field mismatches at build time.

## 4.3 The `payload: string` input problem

The same issue appears on the write side:

```protobuf
// visit_examination.proto — actual current state
message VisitExaminationRequest {
  string payload = 1;  // ← raw JSON string sent from frontend
  string visit_registration_id = 2;
}
```

The entire request body is a JSON string tunneled through gRPC. This means:
- The gateway cannot validate individual fields (it can only check if it's valid JSON)
- The backbone receives an untyped blob and must parse it again
- Any field rename in the frontend breaks silently

Fix is the same: define the request structure in proto.

---

# 5. Priority 4 — The Consument Pattern

## 5.1 What consument is

`consument` (presumably "consumer" — the billing-side identity of the patient) appears as:
- `interface{}` in `TransactionMeta.Consument`
- Structured `PropConsument` with id, name, phone, payment_summary, user_wallet, reference
- Inside `Reference.Consument` — creating a circular reference

It represents "who is the payer for this service" — which may differ from the patient (a parent paying for a child, a company paying for an employee, an insurance payer, etc.).

## 5.2 The problem

Because `Consument` is typed as `interface{}` in Go, every caller must type-assert or JSON-decode it before use. There's no contract for what's inside. The `PropConsument` type in the meta package provides some structure but it's not consistently used.

## 5.3 The right model

Consument is a legitimate concept — it's the billing party, which may or may not be the patient. The fix is to make it a first-class entity with a typed contract:

```go
// Clear separation of concerns
type BillingParty struct {
    Type           string `json:"type"`             // "patient" | "insurance" | "company"
    ID             string `json:"id"`               // FK to the relevant entity
    Name           string `json:"name"`             // denormalized display name
    PaymentMethod  string `json:"payment_method"`   // "cash" | "bpjs" | "private_insurance"
    InsuranceID    *string `json:"insurance_id"`    // BPJS number, policy number, etc.
}
```

This is a typed column or a well-documented jsonb struct — not `interface{}`.

---

# 6. Priority 5 — Request/Response Symmetry

## 6.1 The asymmetry problem

`FamilyRelationship` is a concrete example of a broader pattern:

**Request (what you send):**
```protobuf
message FamilyRelationshipRequest {
  optional string id = 1;
  string family_role_id = 2;   // ← structured reference
  string name = 3;
  string phone = 4;
}
```

**Response (what you get back):**
```protobuf
message FamilyRelationship {
  string id = 1;
  string name = 2;
  string phone = 3;
  string props = 4;   // ← family_role_id is now buried in props
}
```

You send `family_role_id` as a structured field and get back a blob. You cannot round-trip your own data. The frontend has to know that `family_role_id` is inside `props` in the response even though it sent it as a top-level field.

## 6.2 The rule

**The response must be a superset of the request.** Any field the client can write, the client must be able to read back at the same path. This is table stakes for a REST or gRPC API.

---

# 7. Quick Wins (No Architecture Change Required)

These can be done in the next sprint without proto changes or DB migrations:

## 7.1 Move `BACKBONE_GRPC_ADDRESS` into config

```go
// internal/route/api.go:79 — current
backboneAddr := os.Getenv("BACKBONE_GRPC_ADDRESS")

// Fix: add to internal/config/env.go
var BackboneGrpcAddress string
// in Initialize(): BackboneGrpcAddress = os.Getenv("BACKBONE_GRPC_ADDRESS")
```

One line. Consolidates all config in one place for auditing.

## 7.2 Add input validation to mutation routes

The `validation.RequestValidator[T]()` middleware already exists and works (see role module). Wire it to the clinical mutation endpoints:

```go
// Current (no validation)
visitRegistrationGroup.Post("/:id/visit-examination", visitExaminationModule.Handler.Save)

// Target (with validation)
visitRegistrationGroup.Post("/:id/visit-examination",
    validation.RequestValidator[*visitExaminationDto.StoreRequest](),
    visitExaminationModule.Handler.Save,
)
```

Requires defining DTO structs for each endpoint. The infrastructure is already there.

## 7.3 Remove the production debug log

```go
// visit_examination/service/visit_examination.go:37 — delete this line
log.Infof("RESP (raw): %+v", resp.Data)
```

## 7.4 Use `google.protobuf.Timestamp` for dates

Timestamps sent as strings require runtime parsing and lose timezone information. Proto has a native timestamp type. Adopt it in all new proto messages; migrate existing ones when the contract is updated for other reasons.

---

# 8. DB Design Philosophy for the Transition

To summarize the above as a set of principles for new table design and existing table migration:

## 8.1 The decision tree for every field

```
Is this field queried/filtered directly?
  YES → typed column with appropriate index
  NO  →
    Is this field part of a SATU SEHAT or external contract?
      YES → typed column, non-nullable, indexed
      NO  →
        Is this field structurally the same across all rows?
          YES → typed column or documented jsonb struct
          NO  → jsonb field with GIN index and schema comment
```

## 8.2 The indexing strategy

| Data type | PostgreSQL tool |
|-----------|-----------------|
| Fields filtered individually (nik, status, patient_id) | B-tree index on typed column |
| Fields searched as JSON key-value (`props->>'category'`) | GIN index on jsonb column |
| Full-text search across multiple fields | OpenSearch/Elasticsearch |
| Geospatial data (if needed) | PostGIS |
| High-cardinality FK lookups | B-tree index on FK column |

Never index `props` with a B-tree — it won't be used. Either promote to a column or use GIN.

## 8.3 FK constraints that should exist but probably don't

The Laravel pattern of using string UUIDs for all IDs often comes without actual FK constraints (Eloquent doesn't enforce them unless you add `constrained()` explicitly). When re-speccing tables in PostgreSQL:

```sql
-- Add explicit FK constraints
ALTER TABLE transactions
  ADD CONSTRAINT fk_transactions_patient
  FOREIGN KEY (patient_id) REFERENCES patients(id)
  ON DELETE RESTRICT;  -- never silently delete a patient with transactions
```

This catches data integrity issues at the DB level before they reach the application.

## 8.4 The OpenSearch boundary

OpenSearch (or Elasticsearch) handles:
- List endpoints with complex filters across multiple entities
- Autocomplete and fuzzy search
- Pre-projected read models (patient name + transaction code + visit date in one document)
- Aggregate counts and stats dashboards

PostgreSQL handles:
- All writes
- Single-entity detail reads (GET /patient/:id)
- Joins that require transactional consistency
- Anything that touches SATU SEHAT (source of truth only)

The two systems are kept in sync by a queue worker that listens for DB changes and updates the search index. The search index is always considered stale by up to a few seconds. That's acceptable for list views. It's not acceptable for billing or government submissions.

---

# 9. Transition Sequence

In order, accounting for team capacity and dependencies:

1. [ ] **Quick wins** (Section 7) — One sprint, no proto changes needed
2. [ ] **Add input validation DTOs** for clinical modules (Section 7.2) — Protects SATU SEHAT write path
3. [ ] **Audit `submissions.props`** — Determine what SATU SEHAT fields are in there; promote to typed columns
4. [ ] **Add generated columns to `peoples`** for `nik` and `bpjs_number` (Section 3.4, Option 3) — No app change required
5. [ ] **Fix proto contracts** for the 8 blob-through modules (Section 4) — Backbone task; unblocks gateway type safety
6. [ ] **Formalize the Consument type** (Section 5) — Define the billing party contract explicitly
7. [ ] **Incremental props graduation** using the decision tree (Section 8.1) — Ongoing, prioritized per Section 3.3
8. [ ] **Normalize invoice/billing response** — Remove PaymentSummary duplication (Section 2.1)
9. [ ] **Switch list endpoints to OpenSearch** for modules with complex filters — Enables normalized DB schema without sacrificing frontend performance

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 03 Mar 2026 | Alex | Initial pattern guide based on codebase audit and DB schema review. |
