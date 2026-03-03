# WellMed — Gateway Architectural Questions & Anomalies

Observations from codebase analysis. Discovered during: Gateway Documentation & Testing Plan (Feb 2026).

---

## Resolved

| # | Item | Resolution | Date |
|---|------|-----------|------|
| 1.1 | Two JWT secrets with unclear names | Renamed: `JWT_SECRET_KEY` → `JWT_SESSION_TOKEN_SECRET`, `JWT_SECRET` → `JWT_CREDENTIAL_TOKEN_SECRET`. Comment block added to `config/env.go` explaining the two-secret design. | 03 Mar 2026 |
| 1.2 | JWT claims extracted but not stored in Fiber locals | Added `c.Locals()` for `user_id`, `tenant_id`, `tenant_db`, `tenant_schema` in `authenticate.go` after claim validation. | 03 Mar 2026 |
| 1.3 | No timeout on `user.Login` gRPC call | Added `context.WithTimeout(ctx, 3*time.Second)` in `user/service/user.go` — consistent with all other service methods. | 03 Mar 2026 |
| 2.1 | Module path referenced old personal GitHub account | Updated `go.mod` to `module github.com/kalpa-health/wellmed-gateway`. Global find-replace across 185 files (504 occurrences). | 03 Mar 2026 |
| 2.2 | `gorm` imported but not used | Removed via `go mod tidy`. Gateway has no direct DB access. | 03 Mar 2026 |
| 2.3 | Dead `conn` field and `Close()` method on all 39 RPC structs | Removed field, method, and unused `google.golang.org/grpc` import from all RPC files. `grpcx.Manager.Close()` is the correct cleanup path. | 03 Mar 2026 |
| 2.4 | `fmt.Printf` used for error logging in `user/service/user.go` | Replaced with `log.Errorf(...)` using the existing structured logger. | 03 Mar 2026 |
| 2.5 | `generateTokenID` function defined but never called | Not found in codebase — already removed prior to this audit. | 03 Mar 2026 |
| 3.1 | Gateway architecture for external API calls | Decided: external integrations stay in backbone, not gateway. Gateway stays thin. | Mar 2026 |
| 4.5 | Backbone decomposition & service routing | Decided: Consultation extracted to `wellmed-consultation` (ADR-002). Gateway gets `CONSULTATION_GRPC_ADDRESS`. Full execution plan in `kalpa-docs/plans/consultation-extraction/PLAN.md`. | 03 Mar 2026 |

---

## Open

No open items.

---

*Last updated: 03 Mar 2026*
