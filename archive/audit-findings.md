# Kalpa Gateway: Laravel Pattern Audit — Findings Brief

**Version:** 1.1
**Date:** 03 March 2026
**Author:** Alex
**Status:** Complete

---

# 1. Summary

The gateway is structurally sound: correct service layering, clean DI, solid error handling. The two real risks are both rooted in the same cause — the backbone returns data as untyped JSON blobs (`props` strings, `Data` strings) and the gateway passes them through without understanding their shape. This is not a Go anti-pattern carried from Laravel. It's a protocol design decision that worked at small scale and is now becoming a constraint.

| # | Pattern | Risk | Confirmed |
|---|---------|------|-----------|
| 2.1 | Postman collection is hand-authored, not generated | Low | ✅ |
| 2.2 | `props` JSON blob fields — 49 tables, system-wide | **Critical** | ✅ |
| 2.3 | 8 modules pass backbone response through as untyped blob | High | ✅ |
| 2.4 | Mutation routes have no input validation at gateway | High | ✅ |
| 2.5 | Error handling | Low | ✅ (GOOD — no issue) |
| 2.6 | One stray `os.Getenv` outside config package | Low | ✅ |
| 2.7 | DI / global state | Low | ✅ (GOOD — no issue) |
| 2.8 | ORM patterns | N/A | ✅ (no ORM in gateway) |

---

# 2. Findings

## 2.1 Postman Collection Source

**Finding: Hand-authored. No code-generation link.**

`go.mod` has no Swaggo, no OpenAPI library, and no `@Router/@Summary` annotation comments exist in any `.go` file. The Postman collection was built and is maintained by hand.

**Implication:** The garbage in Postman request bodies is a documentation problem, not a code problem. Fixing Go structs will not update Postman automatically. The collection will drift from the code indefinitely unless a generation step is added.

**What it would take to fix:** Add `swaggo/swag` to go.mod, add annotations to handler function comments, wire `swag init` into the build, import the generated spec into Postman. One afternoon of work; high leverage.

---

## 2.2 The `props` Field Problem

**Finding: System-wide. 49 tables. This is the backbone's foundational data pattern.**

`props` is not an isolated field — it is how the Laravel backbone stores most non-trivial data. The DB scope query returned 49 unique tables with `props` columns:

**By domain:**

| Domain | Tables | JSON type |
|--------|--------|-----------|
| **Financial** | `transactions`, `transaction_items`, `transaction_has_consuments`, `billings`, `invoices`, `payment_details`, `payment_summaries`, `split_payments`, `refunds`, `wallet_transactions`, `deposits`, `user_wallets`, `withdrawals` | `jsonb` (most) |
| **Patient/Clinical** | `peoples`, `submissions`, `diseases`, `addresses`, `wellmed_addresses` | `json` |
| **Geographic** | `provinces`, `districts`, `subdistricts`, `villages` | `json` |
| **Access Control** | `users`, `roles`, `permissions`, `tenants`, `domains`, `workspaces`, `licenses`, `personal_access_tokens`, `api_accesses` | `json` |
| **Product/Services** | `unicodes`, `services`, `product_items`, `consuments`, `config_props`, `versions`, `installed_features`, `installed_product_items` | `json` / `jsonb` |
| **Relations/Misc** | `model_has_encodings`, `model_has_licenses`, `model_has_phones`, `model_has_relations`, `wellmed_model_has_encodings`, `user_references`, `activities`, `xendit_logs`, `submission` | `json` |

**The `submissions` table is the most urgent concern.** If SATU SEHAT submission payloads are stored in a `props` column, that means government health integration data — the fields the ministry validates, the fields that trigger billing — live in an unstructured blob with no schema enforcement. Any debugging, auditing, or compliance review of a submission requires parsing JSON by hand.

**How this reaches the gateway:**

5 proto contracts expose `props` to the gateway:

| Proto | Entity | Gateway handling |
|-------|--------|-----------------|
| `unicode.proto:31` | All 22 unicode entities | Decoded to `map[string]interface{}` — leaks to API clients |
| `visit_registration.proto:40` | VisitRegistration | Partially decoded via `ExtractProps()` — some structure, still fragile |
| `visit_examination.proto:46` | VisitExamination | Passed through as raw string |
| `patient.proto:206,219` | Patient, People (field 99) | "Keep for backward compat" — NOT exposed in API |
| `transaction.proto:38` | CreateTransactionRequest | Input blob forwarded to backbone |

```go
// internal/schema/resource/unicode.go — what API clients receive
type UnicodeResource struct {
    // ...
    Props map[string]interface{} `json:"props,omitempty" map:"Props"`
}
```

Any field inside `Props` is invisible to Go's type system. Adding a field in the backbone silently appears in 22+ API responses. Removing one silently disappears. No compile error, no test failure, no warning.

**Why this pattern exists:** In Laravel, `$casts = ['props' => 'array']` on an Eloquent model is a common pattern for fields that weren't fully modeled during early development. It's a productivity shortcut — skip the migration, put it in props. At small scale, this is pragmatic. At 49 tables and a government health integration, the flexibility has become a liability: the schema is in the application code (or nowhere), not in the database.

**The right framing for the team:** This is not wrong code — it's a Laravel convention that was the right call at the time. The question now is which `props` fields have stabilized enough to graduate into first-class columns, starting with the ones that SATU SEHAT touches.

---

## 2.3 Backbone Response as Untyped Blob (8 modules)

**Finding: Confirmed. High risk for clinical/write endpoints.**

Eight domain modules receive `result.Data` — a raw JSON string from the backbone — and pass it straight through to the API response:

```go
// Pattern found in: frontline, visit_examination, pharmacy_sale,
// referral, item, visit_patient, medical_treatment, visit_registration_referral

func (h *FrontlineHandler) Show(ctx fiber.Ctx) error {
    result, err := h.service.GetById(helper.GetGrpcContext(ctx), ctx.Params("id"))
    // ...
    var response map[string]interface{}          // ← no type, no contract
    json.Unmarshal([]byte(result.Data), &response)
    return ctx.Status(http.StatusOK).JSON(jsonResponse)
}
```

**The service layer is equally opaque:**

```go
// visit_examination/service/visit_examination.go
func (s *visitExaminationService) Store(ctx context.Context, requestBody, visitRegistrationId string) (*visitexaminationpb.ShowResponse, error) {
    // ...
    resp, err := s.client.Store(cctx, requestBody, visitRegistrationId)
    return resp, err  // ← zero transformation, zero knowledge of shape
}
```

**What this means:** For these 8 modules, the gateway has no knowledge of what the backbone's response looks like. Any backbone change — field rename, field removal, type change — silently propagates to clients. There is no compile-time or test-time check.

**Contrast with typed modules:** Patient, user, transaction, and all 22 unicode modules correctly use typed resource structs (`PatientResource`, `UserResource`, `TransactionDTO`) with explicit proto→DTO mapping. Those are fine.

**The 8 affected modules are the most clinically sensitive ones:** visit examination, visit registration, frontline pharmacy, referral, pharmacy sale.

---

## 2.4 Missing Input Validation on Mutation Routes

**Finding: Most POST/PUT routes have no gateway-level validation.**

This is the most actionable finding. Only one mutation route uses the existing validation middleware:

```go
// route/api.go:226 — ONLY validated mutation route
roleGroup.Post("", validation.RequestValidator[*roleDto.RoleRequest](), roleModule.Handler.Save)
```

Every other POST/PUT goes directly to the handler:

```go
// route/api.go:374 — NO validation
visitRegistrationGroup.Post("/:visit_registration_id/visit-examination", visitExaminationModule.Handler.Save)
visitRegistrationGroup.Put("/:visit_registration_id/visit-examination", visitExaminationModule.Handler.Update)

// route/api.go:400 — NO validation
frontlineGroup.Post("", frontlineModule.Handler.Save)

// route/api.go:364 — NO validation
patientGroup.Post("", patientModule.Handler.Save)
```

Inside these handlers, the raw body is forwarded as-is:

```go
// visit_examination/handler/visit_examination.go:58
func (h *VisitExaminationHandler) Save(ctx fiber.Ctx) error {
    requestBody := ctx.Body()
    result, err := h.VisitExaminationService.Store(helper.GetGrpcContext(ctx), string(requestBody), ...)
    // ← no parsing, no validation, no DTO
}
```

**What this means:** The backbone is the only validation layer. A malformed request, a missing required field, or a field with the wrong type reaches the backbone and either fails with an opaque gRPC error or — worse — succeeds and corrupts data.

**The validation infrastructure already exists.** `internal/validation/validation.go` has a working `RequestValidator[T any]()` generic middleware with Zod-style struct tag validation. The role DTO proves it works. The gap is that no one wired it up for the clinical modules.

**Why this matters for SATU SEHAT:** Any patient or visit examination write that makes it to the backbone with bad data becomes a SATU SEHAT submission problem. The government integration cannot be the first validation layer.

---

## 2.5 Error Handling

**Finding: Good. No material issues.**

All handlers use `helper.HandleGrpcErrorResponse(ctx, err)` consistently, which correctly maps gRPC status codes to HTTP codes. No swallowed errors, no bare `fmt.Println`, no `_, err` without use.

One minor hygiene issue: `visit_examination/service/visit_examination.go:37` has a debug log that should be removed before production:

```go
log.Infof("RESP (raw): %+v", resp.Data)  // ← development artifact
```

---

## 2.6 Configuration

**Finding: One stray `os.Getenv` outside the config package.**

All 13 env vars are correctly centralized in `internal/config/env.go` and loaded at startup via `config.Initialize()`. One variable was added later and bypassed this:

```go
// internal/route/api.go:79 — should be in config/env.go
backboneAddr := os.Getenv("BACKBONE_GRPC_ADDRESS")
```

`BACKBONE_GRPC_ADDRESS` is missing from `config/env.go`. Minor, but means this var is invisible in config audits.

---

## 2.7 Dependency Injection and Global State

**Finding: Good. No material issues.**

The DI pattern is correctly implemented throughout. gRPC clients, Redis, and all domain dependencies are constructed in `route/api.go` and injected via `NewXxx()` constructors. `wires.go` files assemble domain modules cleanly. No global DB connections, no global gRPC clients.

Config vars are package-level strings, which is idiomatic for simple Go servers. Not a concern.

---

## 2.8 ORM Usage

**Finding: Not applicable.**

The gateway is a pure gRPC→HTTP adapter with no direct database connection. All data access is via the backbone. No GORM, no AutoMigrate, no Preload chains.

---

# 3. Priority Order

Based on risk to the SATU SEHAT integration and the team's near-term work:

**Fix first (before any new feature work on clinical modules):**

1. [ ] **Input validation on mutation routes** (Section 2.4) — DTOs and `RequestValidator` wiring for visitExamination, patient, frontline, referral. The infrastructure exists — it's one DTO struct + one line per route. Highest immediate risk because bad input reaching the backbone means bad data in SATU SEHAT.

2. [ ] **Audit the `submissions.props` column** (Section 2.2) — Inspect what fields are actually being stored in `submissions.props`. If SATU SEHAT payload fields are in there, the backbone team needs a migration plan to promote them to first-class columns. This is a backbone task, not a gateway task, but the gateway team needs to understand the contract.

**Fix as the clinical modules stabilize:**

3. [ ] **Typed response structs for the 8 blob-through modules** (Section 2.3) — Define typed Go structs in the gateway for visitExamination, frontline, pharmacy_sale, referral, visit_patient, item, medical_treatment, visit_registration_referral responses. This is the larger project but it's how the gateway eventually owns its API contract.

4. [ ] **Incrementally graduate `props` to typed columns in the backbone** (Section 2.2) — Start with `peoples.props` (patient demographics that SATU SEHAT validates) and `submissions.props` (government submission data). Don't try to fix all 49 tables — pick the ones with regulatory exposure. This is a backbone/migration effort.

**One-line fixes (do in the next PR):**

5. [ ] **Move `BACKBONE_GRPC_ADDRESS` into `config/env.go`** (Section 2.6)
6. [ ] **Remove debug log from `visit_examination/service/visit_examination.go:37`** (Section 2.5)

**Non-urgent:**

7. [ ] **Add Swaggo for Postman generation** (Section 2.1) — Quality-of-life improvement, high leverage once the DTO work is done.

---

# 4. What This Is Not

This is not a critique of the Laravel-to-Go decision or the team's work. The gateway's structure is correct — proper layering, working DI, consistent error handling, 91.8% test coverage on tested modules. The issues here are specific to how the backbone's data model was translated into the gateway's API contracts. That's a hard problem and the team got the architecture right. These are targeted gaps.

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 03 Mar 2026 | Alex | Initial findings from codebase investigation. DB scope query pending. |
| 1.1 | 03 Mar 2026 | Alex | Added DB scope results — 49 tables confirmed. Escalated 2.2 to Critical. Updated priority order. |
