# ADR-008: Props Field Classification Policy

**Status:** Accepted
**Date:** 2026-03-06
**Author:** Alex Knecht
**Reviewers:** Kalpa Engineering Team

---

## 1. Context

1.1 All Go services in the Kalpa ecosystem (wellmed-backbone, wellmed-consultation, wellmed-pos) use a `props` JSONB column on EMR entities. This column was introduced as a Laravel-era pattern via `helper.BuildProps()` / `helper.BuildPropsWithFilter()`, which serialises related entity snapshots into JSON at write time to avoid N+1 queries.

1.2 Over time, three distinct categories of data have accumulated in `props`:

- **Snapshot copies** of related entities (patient, medic service, visit patient, etc.), auto-prefixed with `prop_` by the helper. These duplicate data already available via relational FK/JOIN.
- **Business fields** with stable types that belong in real columns (status, tenant_id, name). These ended up in props as a side-effect of the helper pattern, not by design.
- **Genuinely variable data** — rows where the shape legitimately differs per record (assessment exam types, prescription payloads, external referral content). This is the only valid use case for JSONB.

1.3 In Go, GORM's `Preload()` and explicit JOINs solve the N+1 problem cleanly without data duplication. The snapshot pattern is therefore unnecessary except for cases with genuine per-row schema variation.

1.4 A props expression index exists on `visit_registrations` (`props->'prop_visit_patient'->'patient'->>'id'`) — a leading indicator that snapshot data was being treated as queryable state, which should instead come from a relational join on an indexed FK column.

1.5 The POS service (wellmed-pos) is being onboarded and has an equivalent props accumulation problem. This policy is intended to apply at intake, preventing recurrence.

---

## 2. Decision

2.1 All `props` fields across all Kalpa Go services are classified into exactly one of three categories. Classification determines the required action.

### 2.1.1 [K] Keep — Legitimately Variable Schema

**Rule:** The shape of the data varies per row by design and cannot be normalised to a fixed schema. These stay in `props`.

**Criteria:**
- No two rows of the same model have the same structure for this key
- The data is not queried by key in a WHERE clause or JOIN condition
- The variable shape reflects real business variance (not laziness)

**Current examples:**
- `assessment` — anthropometry, vital signs, symptoms, morphs (shape varies by assessment type)
- `frontline` assessments — prescription payload (shape varies by prescription type: medicine, tool, mix)
- `referral` — `prop_external_referral` payload (shape varies by referral destination type)

**Requirements for [K] fields:**
- The schema for each variant MUST be documented in a metadata struct or inline comment
- Field names MUST NOT use the `prop_` prefix (that prefix is reserved for legacy snapshots being removed)

### 2.1.2 [J] Join — Snapshot of Relational Data

**Rule:** Data that duplicates a related entity reachable via a FK/JOIN. Stop writing it. Replace reads with GORM `Preload()` or explicit JOINs.

**Criteria:**
- Field was written by `BuildProps()` / `BuildPropsWithFilter()` with a `prop_` prefix
- The underlying data lives in another table with a FK relationship
- OR: a stable scalar (status, name, tenant_id) that is already a real column on the table

**All `prop_*` prefixed keys are [J] by definition.**

Action:
- Remove `BuildProps` / `BuildPropsWithFilter` calls that produce these keys
- Replace `gjson.Get(props, "prop_*")` reads with relational queries
- Drop the expression index; ensure FK column has a standard B-tree index

### 2.1.3 [C] Columnize — Stable Business Field Requiring Its Own Column

**Rule:** A business field with a stable type that must be queryable and is not reachable via JOIN from an existing FK.

**Current state:** No [C] items were identified during the 2026 review. All candidate fields (tenant_id, name, status, patient_id) were found to already exist as columns or to be reachable via FK. This category is included for completeness and future reviews.

**If a [C] item is found:** Add a typed column via migration, backfill from props JSON, add index if needed, then remove from props.

---

## 3. Consequences

### 3.1 Positive

- Eliminates data duplication across snapshot-heavy tables (visit_registrations, visit_patients, visit_registration_referral, referral)
- Props columns shrink to genuine variable-schema data only — smaller rows, less IO
- Read paths become explicit and type-safe (GORM relations vs. gjson string paths)
- Policy is portable: POS and any future service can reference this ADR at onboarding

### 3.2 Negative

- Service code must be rewritten in multiple domains across two repos simultaneously (backbone + consultation)
- Existing prod rows retain dead `prop_*` keys until a cleanup migration runs — brief inconsistency window
- GORM Preload adds query overhead compared to single-row JSON read; must ensure FK indexes are in place before deploying

### 3.3 Risks

- **Risk:** A `prop_*` key is read somewhere not caught by the code review (frontend, reporting query, external integration)
  **Mitigation:** Grep all repos for `prop_visit_patient`, `prop_medic_service`, etc. before removing. Run in parallel with a shadow read from the relational path before cutting over.

- **Risk:** cleanup migration strips keys that were [K] by mistake
  **Mitigation:** Cleanup migration targets only known `prop_` prefixed keys (the legacy prefix). [K] fields do not use this prefix.

---

## 4. Alternatives Considered

| Alternative | Pros | Cons | Why Not Chosen |
|-------------|------|------|----------------|
| Columnize frequently-queried props | Strong type safety, fast WHERE clauses | Schema migrations, backfills for every new field | No [C] items found; overkill for current query patterns |
| Keep all props as-is | Zero migration risk | Compounds with POS onboarding; expression indexes scale poorly; no type safety | Deferred cost grows with each new service |
| Materialised views over props JSON | Read performance without code changes | Views are not writable; doesn't fix write duplication | Addresses symptom, not cause |

---

## 5. Implementation Notes

5.1 **`prop_` prefix is the legacy marker.** Any existing key starting with `prop_` is [J] and must be removed. Do not create new keys with this prefix.

5.2 **Helper functions.** `helper.BuildProps()`, `helper.BuildPropsWithFilter()`, `helper.BuildPropsV2()` in `internal/helper/virtual_column.go` are the source of all snapshot writes. After migration, calls to these functions should only pass [K] data. If no [K] data exists for a model, the props column write can be removed entirely.

5.3 **`gjson` reads.** Any `gjson.Get(entity.Props, "prop_*")` call is a [J] read and must be replaced with a relational query or preloaded association.

5.4 **Applies to both repos.** wellmed-backbone and wellmed-consultation contain mirrored domain service implementations. All changes must be applied to both. See `props-migration-PLAN.md` for sequencing.

5.5 **POS onboarding.** When wellmed-pos is integrated, apply this classification at intake before any new `BuildProps` calls are introduced. The same [K]/[J]/[C] framework applies.

5.6 **Unused props columns.** Tables where props was never written (`practitioner_evaluations`, `prescriptions`) should have the column dropped after confirming no reads exist.

---

## 6. References

- Props module review: `kalpa-docs/plans/props-module-review/props-module-review-PLAN.md`
- Migration execution plan: `kalpa-docs/plans/props-migration/props-migration-PLAN.md`
- `internal/helper/virtual_column.go` — BuildProps helper functions
- `internal/schema/metadata/` — existing [K] metadata structs

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-06 | Alex Knecht | Initial ADR from props module review findings |
