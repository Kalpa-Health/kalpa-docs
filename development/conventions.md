# Code Conventions

**Version:** 0.1 (stub)
**Date:** 01 March 2026
**Status:** Stub — foundational conventions documented; additional detail to be added
**Maintained by:** Alex

---

## 1. Overview

1.1.1 These conventions apply across all WellMed Go services. Consistency means any developer can navigate any service immediately. For code organization and project structure, see [`wellmed-system-architecture.md §6`](../wellmed-system-architecture.md).

---

## 2. Naming

| Scope | Convention | Example |
|-------|-----------|---------|
| Files | `snake_case` | `visit_handler.go` |
| Packages | `lowercase`, no underscores | `package emr` |
| Types/Structs | `PascalCase` | `VisitService` |
| Interfaces | `PascalCase` | `VisitRepository` |
| Functions/Methods | `camelCase` | `createVisit()` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRY_COUNT` |
| Database tables | `snake_case` | `visit_registrations` |
| Environment variables | `SCREAMING_SNAKE_CASE` | `BACKBONE_GRPC_ADDRESS` |

---

## 3. Project Structure

3.1 Every service follows the four-layer pattern:

```
/internal/domain/{module}/
├── handler/          # HTTP/gRPC entry point
├── service/          # Business logic (80% coverage target)
├── repository/       # Database access (integration tests only)
└── dto/              # Data transfer objects
```

3.2 Shared packages used across services belong in `/pkg/`. Service-specific code stays in `/internal/`.

---

## 4. Go Idioms

### 4.1 Error Handling

- Always handle errors explicitly — never ignore with `_`
- Wrap errors with context: `fmt.Errorf("creating visit: %w", err)`
- Use custom error types for domain errors (e.g., `ErrPatientInactive`)

### 4.2 Context

- All functions that do I/O (DB, gRPC, HTTP) must accept `context.Context` as the first parameter
- Always propagate context; never use `context.Background()` inside handlers

### 4.3 IDs

- Use `github.com/oklog/ulid/v2` for all entity IDs — never `uuid` or sequential integers
- See [`wellmed-testing-architecture.md §1.2.6`](../wellmed-testing-architecture.md) for rationale

### 4.4 Logging

- Use `uber-go/zap` with structured JSON logging
- Always log at the service boundary (handler or service layer), not inside loops
- Never use `fmt.Printf` or `log.Println` in production code

---

## 5. Proto / gRPC Conventions

- [ ] TODO: Document proto naming conventions
- [ ] TODO: Document versioning convention for proto files (`/proto/emr/v1/emr.proto`)

---

## 6. Database Conventions

- Parameterized queries always — never string concatenation
- Tables named in `snake_case`, prefixed by domain if needed
- High-volume tables partitioned by year (e.g., `visits_2025`, `visits_2026`)
- Migrations use the pattern documented in [`operations/database-migrations.md`](../operations/database-migrations.md)

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 01 Mar 2026 | Alex + Claude | Initial stub with foundational conventions |
