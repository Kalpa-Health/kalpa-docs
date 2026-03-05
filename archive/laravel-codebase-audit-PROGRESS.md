# Progress Log — Kalpa Go Codebase Audit

## Session: 2026-03-03T00:00:00Z

### Investigation 2.1: Postman Collection Source
- **Status**: ✅ DONE
- **Completed**: 2026-03-03
- **What was done**: Checked go.mod for Swaggo/OpenAPI libraries. Searched all .go files for `@Router/@Summary/@Param/@Success` Swaggo annotation comments. Checked for docs/swagger directory.
- **Files searched**: `go.mod`, all `*.go` files in wellmed-gateway-go/
- **Finding**: No Swaggo, no OpenAPI library, no annotation comments. The Postman collection is hand-authored — there is no code-to-Postman link. Garbage in Postman request bodies is a documentation problem, not a code generation problem.
- **Issues**: None (clean finding)

### Investigation 2.2: `props` Field Problem (code + DB)
- **Status**: ✅ DONE
- **Completed**: 2026-03-03
- **What was done**: Searched all Go structs and proto files for `props` field usage. DB scope query run against production schema.
- **Files searched**: `internal/**/*.go`, `proto/**/*.go`, `proto/*.proto`
- **DB finding**: **49 unique tables** have `props` columns. 12 use `jsonb` (performance-optimized binary JSON), the rest `json` text. This is systemic — props is architectural DNA across the entire backbone, not an isolated pattern.
- **Highest-risk tables**: `submissions` (SATU SEHAT), `peoples` (patient demographics), `transactions`/`billings`/`invoices`/`payment_details`/`payment_summaries` (financial stack — all jsonb), `users`, `tenants`
- **Code-side**: 5 proto contracts expose props to the gateway — unicode (22 modules, decoded to `map[string]interface{}`), visit_registration (partially decoded), visit_examination (passed through), patient (field 99, "backward compat", not mapped), transaction (input blob)
- **Issues**: `submissions` table having a props column is the highest-priority concern — if SATU SEHAT submission details are in that blob, that's regulatory-level unstructured data.

### Investigation 2.3: Model = API Response
- **Status**: ✅ DONE
- **Completed**: 2026-03-03
- **What was done**: Read handler files for blob-pass-through pattern. Searched for `interface{}` and `map[string]interface{}` in domain handlers.
- **Key files**: `frontline/handler/frontline.go`, `visit_examination/handler/visit_examination.go`, `pharmacy_sale/handler/pharmacy_sale.go`, `referral/handler/referral.go`, `item/handler/item.go`, `visit_patient/handler/visit_patient.go`, `medical_treatment/handler/medical_treatment.go`, `visit_registration_referral/handler/visit_registration_referral.go`
- **Finding**: The gateway-specific analog of "model as response" is confirmed in ~8 modules. These handlers receive `result.Data` (a raw JSON string from gRPC) and unmarshal it into `var response interface{}` or `var response map[string]interface{}`, then re-serialize it. The API contract is whatever the backbone returns — the gateway has zero knowledge of the response shape. The remaining ~31 modules (patient, user, transaction, all unicode modules) use typed resource/DTO structs correctly.
- **Issues**: None (found what we expected)

### Investigation 2.4: Service Layer Thinness
- **Status**: ✅ DONE
- **Completed**: 2026-03-03
- **What was done**: Read service files for blob-pass-through modules. Checked route/api.go for validation middleware usage.
- **Key files**: `visit_examination/service/visit_examination.go`, `frontline/service/frontline.go`, `route/api.go`
- **Finding**: Service layer EXISTS structurally (correct handler→service→rpc separation) but carries no business logic for blob-pass-through modules. Services are timeout wrappers: `Store(ctx, rawBody, id)` → `client.Store(ctx, rawBody, id)`. Only 1 of ~15 mutation routes uses validation middleware: `roleGroup.Post("", validation.RequestValidator[*roleDto.RoleRequest](), ...)`. All other mutation endpoints (frontline.Save, visitExamination.Save/Update, patient.Save, referral.Save, etc.) accept and forward raw request bodies with zero gateway-level validation. Debug log left in production: `visit_examination/service/visit_examination.go:37` — `log.Infof("RESP (raw): %+v", resp.Data)`.
- **Issues**: The no-validation-on-mutations finding is more severe than the plan anticipated.

### Investigation 2.5: Error Handling
- **Status**: ✅ DONE
- **Completed**: 2026-03-03
- **What was done**: Searched for `_, err` patterns without use, `fmt.Println(err)`, `log.Println(err)` without return.
- **Finding**: Error handling is GOOD. All handlers use `helper.HandleGrpcErrorResponse(ctx, err)` consistently. The two `_, err` instances in production code are correct: `validation.go:28` uses `err` as `return err == nil`; transaction service test files are correct in test context. No swallowed errors. No unguarded `fmt.Println`.
- **Issues**: One code-hygiene issue: debug log in visit_examination service.go:37 should be removed.

### Investigation 2.6: Configuration
- **Status**: ✅ DONE
- **Completed**: 2026-03-03
- **What was done**: Read `internal/config/env.go`. Searched all Go files for `os.Getenv` outside the config package.
- **Key files**: `internal/config/env.go`, `internal/route/api.go:79`
- **Finding**: Configuration is 95% well-structured. 13 env vars are centralized in `config/env.go` with a single `Initialize()` called at startup, stored as package-level vars. One stray outlier: `route/api.go:79` reads `os.Getenv("BACKBONE_GRPC_ADDRESS")` directly, bypassing the config package. This var is missing from `env.go`.
- **Issues**: One stray var; low risk but should be unified.

### Investigation 2.7: Dependency Injection vs. Global State
- **Status**: ✅ DONE
- **Completed**: 2026-03-03
- **What was done**: Searched for package-level `var` of pointer types in non-main files. Read config package. Examined wires.go pattern.
- **Finding**: The DI pattern is GOOD. gRPC client, Redis client, and all domain handlers/services are correctly constructed and injected via `NewXxx()` constructors and assembled in `route/api.go`. The config package uses package-level string vars (not pointers) — idiomatic for simple Go servers. No dangerous mutable global DB or gRPC connections. `validation.Validate` is a package-level singleton — acceptable.
- **Issues**: None material.

### Investigation 2.8: ORM Usage
- **Status**: ✅ DONE (N/A)
- **Completed**: 2026-03-03
- **What was done**: Searched for `gorm`, `AutoMigrate`, `Preload` in all Go files.
- **Finding**: No ORM present. The gateway has no direct DB connection — it's a pure gRPC→HTTP adapter. All data access goes through the backbone via gRPC. ORM anti-patterns are non-applicable for this service.
- **Issues**: None.

---

### CTO Pattern Guide
- **Status**: ✅ DONE
- **Completed**: 2026-03-03
- **Files created**: `kalpa-docs/laravel-go-pattern-map.md`
- **Coverage**: 9 sections — document model vs relational+search, props graduation decision tree, proto contract fixes, consument pattern, request/response symmetry, quick wins, DB design philosophy, OpenSearch boundary, transition sequence.

## Audit Complete

All investigation areas complete. Two output documents ready for team meeting:
1. `kalpa-docs/audit-findings.md` — findings brief (team)
2. `kalpa-docs/laravel-go-pattern-map.md` — pattern guide (CTO)
