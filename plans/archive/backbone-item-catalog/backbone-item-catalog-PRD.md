# Backbone Item Catalog Package — PRD

**Status:** Draft
**Author:** Alex
**Date:** 2026-03-04
**File:** `kalpa-docs/development/prds/backbone-item-catalog-PRD.md`

---

## 1. Problem Statement

1.1. There is no canonical owner for billable item definitions (services, obat, consumables, equipment, admin fees, packages) in the current Kalpa/WellMed architecture. Without an explicit owner, item definitions and pricing risk being embedded in domain MS (Consultation, Lab, Radiology) in incompatible ways.

1.2. The Saga orchestrator requires both item identity and pricing data as explicit steps when processing transactions. There is currently no stable, isolated structure for the orchestrator to read from.

1.3. A future Payer/Invoice MS will need not just custom pricing but a custom *view* of the catalog per payer — renamed items, approved/excluded items, payer-specific billing rules. This requires item identity and pricing to be managed together in a single domain, not split across two packages with a foreign-key dependency between them.

1.4. Item Catalog is a planned future MS extraction target. The package boundary must be clean from the start to make that extraction low-friction.

---

## 2. Users and Goals

2.1. **Saga Orchestrator (Backbone)** — needs to resolve item identity and price for any billable item or composite item set during transaction processing. Must support batch resolution (e.g., 43-item MCU encounter in a single call).

2.2. **Clinic Admin (via future settings UI)** — needs to define, view, and manage items and their prices. Out of scope for this PRD — UI deferred to a later milestone.

2.3. **Domain MS (Consultation, Lab, Radiology)** — register their service definitions into the Item Catalog at setup time. At runtime, they reference `item_id` only — they never read or write pricing data directly.

2.4. **POS MS** — out of scope as a pricing authority. POS applies prices resolved by the Saga; it does not own or query catalog data directly.

2.5. **Payer Module (future)** — will extend Item Catalog with payer-specific item overrides: custom display names, approved/excluded item lists, payer pricelist assignments, and billing rules per item. Schema must support this extension without restructuring.

2.6. **HQ Controller (future)** — will provide cross-tenant catalog controls and rollup visibility. Schema must not foreclose this but does not implement it now.

---

## 3. Requirements

### 3.1. Functional — Item Definition

3.1.1. The system must provide an `itemcatalog` package within the Backbone Go module, isolated from all other Backbone packages — no cross-package imports from other Backbone domains into Item Catalog internals.

3.1.2. The Item Catalog package must own its own database tables with no shared tables or foreign keys to tables owned by other Backbone packages.

3.1.3. The package must store a canonical item record per billable entity with the following fields: `item_id` (UUID), `item_type` (enum: service, obat, consumable, equipment, admin_fee, package), `clinic_id`, `display_name`, `canonical_code` (internal reference code), `is_active`, `created_at`, `updated_at`.

3.1.4. Item records must support a `parent_item_id` self-reference for composite items (panels, packages, MCU bundles). A composite item may have child items; children may themselves be composites (tree structure, not flat list).

3.1.5. The package must expose a service-layer interface (`CatalogService`) through which all external callers interact — no direct repository access from outside the package.

3.1.6. The `CatalogService` must support batch item lookup by `[]item_id` returning full item records in a single call.

### 3.2. Functional — Pricing

3.2.1. The package must store a base price record per item: `item_id`, `clinic_id`, `pricelist_id`, `amount`, `currency` (default IDR), `unit_of_measure`, `effective_from`.

3.2.2. The package must support named pricelists. A pricelist groups item prices under a named policy (e.g., "Standard", "BPJS", "Corporate"). A default/standard pricelist is designated per clinic via an `is_default` flag on the pricelist record.

3.2.3. The `CatalogService` interface must support **batch price resolution**: given `[]PriceRequest{item_id, pricelist_id *optional*, as_of_date}` and `clinic_id`, return `[]PriceResult{item_id, amount, currency, unit_of_measure, err}`.

3.2.4. When `pricelist_id` is nil, the resolver returns the default pricelist price for that clinic.

3.2.5. Effective dating must be enforced — a price record is only valid if `effective_from <= as_of_date`. When multiple records match, the most recent `effective_from` wins.

3.2.6. When no price record exists for a given item in a batch, that item's `PriceResult` must carry a typed error without failing the rest of the batch.

3.2.7. For composite items with a parent-level price, the resolver must return the parent price, not the sum of children. If no parent price exists, the resolver must return a typed error (sum-of-children logic is deferred — it is a business rule requiring explicit future decision).

3.2.8. `unit_of_measure` is required on all obat and consumable price records. For services, it defaults to "per-encounter."

### 3.3. Functional — Payer Item Overrides (Schema Only)

3.3.1. The schema must include a `payer_item_overrides` table with the following fields: `payer_id`, `item_id`, `clinic_id`, `custom_display_name` (nullable), `is_approved` (boolean), `billing_note` (nullable). This table is created now but no business logic is implemented against it in this phase.

3.3.2. No service-layer logic reads from `payer_item_overrides` in this phase. The table exists to support future Payer Module implementation without a schema migration.

### 3.4. Non-Functional

3.4.1. The package boundary is designed as a future MS extraction boundary: own models, own repository layer, own service interface, own migrations — no shared structs with other Backbone packages.

3.4.2. `currency` column must be present on all price records from day one, defaulting to IDR. Multi-currency business logic is not implemented in this phase but the schema must not require a migration to add it later.

3.4.3. All records are `clinic_id`-scoped to support multi-tenant data isolation and future HQ rollup.

3.4.4. No patient data passes through this package — no SATU SEHAT compliance requirements apply.

3.4.5. Item Catalog tables must be included in the standard Backbone backup and migration pipeline.

---

## 4. Acceptance Criteria

4.1. An `itemcatalog` package exists under the Backbone Go module with no import dependencies on other Backbone packages.

4.2. Database migrations create tables (`catalog_items`, `pricelists`, `pricelist_entries`, `payer_item_overrides`) with no foreign keys to tables owned by other Backbone packages.

4.3. `CatalogService` batch item lookup accepts `[]item_id` and returns full item records including composite tree membership in a single database round-trip.

4.4. `CatalogService` batch price resolution accepts `[]PriceRequest` and returns `[]PriceResult` — a 43-item batch executes in minimal round-trips, not one query per item.

4.5. When `pricelist_id` is nil, resolver returns the clinic's default pricelist price (`is_default = true`).

4.6. A missing price record returns a typed error on that item without failing the batch.

4.7. Effective dating is enforced — a future-dated price is not returned for a past `as_of_date`.

4.8. A composite item with a parent price returns the parent price, not a sum of children.

4.9. All price records have non-null `currency`.

4.10. Obat and consumable price records have non-null `unit_of_measure`; DB constraint rejects insert without it for these types.

4.11. `payer_item_overrides` table exists and accepts inserts; no service logic reads from it in this phase.

4.12. The Saga calls `CatalogService` without importing any repository or model types from the `itemcatalog` package directly.

4.13. Seed data covers: one default pricelist per clinic, sample item records for each `item_type` enum value, and sample composite item with at least two children.

---

## 5. Out of Scope

5.1. **HTTP API / admin UI** — no REST endpoints in this package. UI is a future milestone.

5.2. **Voucher / discount logic** — transaction-level concern owned by POS or Saga.

5.3. **Payer-type rules and pricelist selection logic** — the Payer Module will implement the rules engine that selects the correct pricelist for a given payer. This PRD creates the data structures only.

5.4. **Payer item override business logic** — `payer_item_overrides` table is created; logic reading from it is deferred to the Payer Module PRD.

5.5. **UoM conversion and vendor packaging** — `unit_of_measure` field is required, but tablet ↔ strip ↔ box conversion and vendor-specific packaging variations are deferred to a Pharmacy/Inventory PRD.

5.6. **Multi-currency business logic** — currency column is created, defaults IDR, no conversion logic implemented.

5.7. **HQ cross-tenant controls** — `clinic_id` scoping is in place; HQ rollup and control-down patterns are a future milestone.

5.8. **Real-time catalog or price change events** — no pub/sub or event emission in this phase.

5.9. **Sum-of-children composite pricing** — composite items with no parent price return a typed error in this phase. The decision of whether to sum children, error, or apply a fallback rule is deferred and requires an explicit business decision.

---

## 6. Dependencies

- `wellmed-go-backbone` — confirm module path and `itemcatalog` package directory location before implementation.
- **Saga orchestrator calling convention** — the pattern by which Saga invokes Backbone service-layer interfaces must be confirmed before `CatalogService` interface is finalized. If this is the first example of that pattern, the interface design here sets the precedent. See Open Question 7.1.
- **Consultation MS extraction (in-flight)** — `itemcatalog` package should be created before or in parallel with Consultation MS extraction so that Consultation MS registers service definitions into Catalog and never encodes price data internally.
- **Domain MS registration pattern** — Lab, Radiology, and other future MS will register their services into Item Catalog. The registration mechanism (API call, migration seed, admin UI) is not defined in this PRD. See Open Question 7.2.
- **Migration toolchain** — confirm migration tool (golang-migrate, goose, etc.) and existing Backbone conventions before writing migration files.

---

## 7. Open Questions

7.1. **Saga calling convention** — does the Saga have an established pattern for calling Backbone service-layer interfaces, or will `CatalogService` be the first? If the latter, the interface design here sets the precedent for all future packages. `@Hamzah` — resolve before interface design.

7.2. **Domain MS service registration** — when Consultation MS or Lab MS defines a new service type, how does that service get registered as an `item_id` in Item Catalog? Options: (a) admin seeds it manually, (b) domain MS calls a Catalog registration endpoint at startup, (c) migration-time seeding only. This determines whether a write interface is needed in this phase. `@Alex + @Hamzah`

7.3. **Composite pricing fallback** — when a composite item has no parent-level price, should the resolver (a) return a typed error, (b) sum child prices, or (c) support a configurable fallback per composite type? Currently scoped as typed error. Confirm. `@Alex`

7.4. **Default pricelist per clinic** — confirmed as `is_default` flag on pricelist record. Constraint: only one pricelist per clinic may have `is_default = true`. DB unique partial index to enforce this. Confirm approach. `@Hamzah`

---

## 8. Feeds Into

**Plan:** `kalpa-docs/plans/backbone-item-catalog/backbone-item-catalog-PLAN.md` (to be created in Claude Code using this PRD + codebase inspection)
