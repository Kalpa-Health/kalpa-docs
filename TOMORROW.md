# WellMed — Deferred Work Items

**Purpose:** Items raised in architectural discussions that are not blocking go-live and not in the active dev workflow plan. Layer into workflow as capacity allows.

**Last updated:** 05 March 2026

---

# 1. Items

| # | Item | Context | Blocked By | Source |
|---|------|---------|------------|--------|
| 1.1 | Swagger/OpenAPI integration (Swaggo) | Generate API spec from typed Go structs, import into Postman/APIDog. High leverage but only useful after typed protos land. | Proto contract typing (dev-workflow-PLAN Section 2) | audit-findings 2.1 |
| 1.2 | APIDog migration from Postman | Postman licensing/policy changes make this worth evaluating. APIDog supports OpenAPI import. Independent of props work. | Nothing — can start anytime | Team discussion 05 Mar |
| 1.3 | Elasticsearch read path for list endpoints | Write to PostgreSQL via GORM, read lists from ES. Requires parsing ES results into same struct GORM returns. Start with transaction list as proof of concept. | Go-live (not needed at current scale); stable response structs | laravel-go-pattern-map 1.2 |
| 1.4 | Request/response symmetry audit | Fields sent in requests should be readable at the same path in responses. FamilyRelationship is the documented example (family_role_id sent as field, returned inside props). | Props resolution (dev-workflow-PLAN Section 2) | laravel-go-pattern-map 6.1 |
| 1.5 | Invoice/billing response normalization | PaymentSummary appears twice in invoice response. Author appears twice. Remove duplication once proto contracts are typed. | Proto contract typing | laravel-go-pattern-map 2.1 |
| 1.6 | FK constraints audit | Laravel-origin tables may lack actual FK constraints (Eloquent doesn't enforce unless `constrained()` is explicit). Add `FOREIGN KEY ... ON DELETE RESTRICT` for critical relationships. | Nothing — can be incremental | laravel-go-pattern-map 8.3 |
| 1.7 | `google.protobuf.Timestamp` for dates | Replace string timestamps in proto messages with native Timestamp type. Do incrementally when proto contracts are updated for other reasons. | Nothing — do opportunistically | laravel-go-pattern-map 7.4 |
| 1.8 | Non-clinical props documentation | For tables where props stays (unicodes, activities, config_props, geographic): document the JSON schema in code comments or metadata structs. Add GIN indexes where queried. | Props review completion | laravel-go-pattern-map 3.3 Tier 3 |
| 1.9 | Generated columns as migration bridge | PostgreSQL `GENERATED ALWAYS AS (props->>'field') STORED` can create typed, indexed columns without changing application write code. Useful as transitional step during props graduation. | Specific column decisions from props review | laravel-go-pattern-map 3.4 |

---

# 2. Decision Log

Items that were discussed and explicitly deferred (not forgotten):

| Item | Decision | Date | Rationale |
|------|----------|------|-----------|
| ES/GORM read-write split | Deferred post-launch | 05 Mar | Data volumes don't require it yet. Big lift to parse ES into GORM structs. |
| Swagger | Deferred until typed protos | 05 Mar | No value generating specs from `string data` fields. Prerequisite: proto contract typing. |
| Full props cleanup (all 49+ tables) | Incremental, not batch | 05 Mar | Focus on 8 clinical modules first (props-module-review-PLAN). Non-clinical tables are lower risk. |

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 05 Mar 2026 | Alex | Initial deferred items from audit review discussion |
