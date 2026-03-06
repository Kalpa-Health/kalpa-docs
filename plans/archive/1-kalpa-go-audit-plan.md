# Kalpa Go Codebase: Laravel Pattern Audit Plan

**Version:** 1.0
**Date:** 03 March 2026
**Maintained by:** Alex

---

# 1. Purpose

1.1 The Kalpa gateway was refactored from Laravel to Go (Fiber). The refactor carried over architectural habits from Laravel that don't map cleanly to Go — some are benign, some are actively harmful at scale. This document guides a solo investigation of the codebase and database before presenting findings to the team.

1.2 Two outputs are expected: a findings brief for the team (what's wrong and why it matters) and a companion pattern guide for the CTO (what good looks like in Go, mapped to the Laravel equivalent he knows).

1.3 Scope is **patterns**, not every instance. One confirmed example of each pattern is enough to name it and assess risk.

---

# 2. Investigation Areas

## 2.1 How the Postman Collection Was Built

2.1.1 Check `go.mod` for `swaggo/swag`, `fiber-swagger`, or any OpenAPI generation library. This is the most likely source — Swaggo annotations in handler files auto-generate a spec that gets imported to Postman.

2.1.2 Search for Swaggo annotation comments in handler files: `grep -r "@Router\|@Summary\|@Param\|@Success" --include="*.go" .`

2.1.3 If Swaggo is confirmed, the garbage in the Postman request body is coming from example values in struct tags or annotation comments — the collection is a direct reflection of the code structs. Fix the struct, fix Postman automatically.

2.1.4 If no Swaggo, check for a `docs/` or `swagger/` directory with a hand-written spec file.

## 2.2 The `props` Field Problem (High Risk — Start Here)

2.2.1 In the database, run this query to establish scope:

```sql
SELECT table_name, column_name, data_type
FROM information_schema.columns
WHERE column_name LIKE '%props%'
   OR (column_name = 'props' AND data_type IN ('json', 'jsonb', 'text'));
```

2.2.2 In the codebase, find Go structs that map to this column: `grep -rn "props" --include="*.go" .` — look for `json:"props"` struct tags, especially typed as `json.RawMessage`, `map[string]interface{}`, or `interface{}`.

2.2.3 The Laravel origin: Eloquent models often use a `$casts = ['props' => 'array']` pattern — a catch-all JSON blob for fields that weren't fully modeled. In Go this becomes an untyped blob that leaks straight into API responses.

2.2.4 Risk assessment: **High.** Any consumer of the API is now coupled to an unstructured blob. Adding fields to `props` silently breaks clients. Removing fields is invisible. No type safety. No validation.

2.2.5 What to look for beyond just `props`: any column typed `jsonb` or `json` that stores multiple logical fields. These are the same problem with different names.

## 2.3 Model = API Response (High Risk)

2.3.1 In Laravel, it's normal to return an Eloquent model directly from a controller — `return response()->json($model)`. The model is the response. In Go, this pattern means the DB struct is being marshalled directly to JSON.

2.3.2 Search for handler files returning DB structs directly: look for patterns where the same struct appears in both a database query result and a `c.JSON()` call.

2.3.3 Risk assessment: **High.** Exposes internal DB field names to API consumers. Any DB schema change becomes a breaking API change. Password fields, internal flags, timestamps get leaked unless manually excluded.

2.3.4 What good looks like in Go: a separate response struct (DTO) that is explicitly populated from the DB model. The DB model and the API shape are decoupled.

## 2.4 Missing or Thin Service Layer (Medium Risk)

2.4.1 Laravel enforces nothing but convention encourages Controllers → Services → Models. Go enforces nothing either, but without deliberate layering the Laravel refactor often lands as fat handlers — business logic living directly in HTTP handler functions.

2.4.2 Look at handler files: if a handler is doing database queries, business logic, and response formatting all in one function, the service layer is missing or thin.

2.4.3 Risk assessment: **Medium.** Not an immediate bug risk, but makes the codebase untestable and causes logic duplication as features are added. Becomes High as the team grows.

## 2.5 Error Handling (Medium Risk)

2.5.1 Laravel uses exceptions with automatic HTTP response mapping. Go errors are values — they must be checked and handled explicitly at every step. A Laravel-trained developer's instinct is to let errors bubble, which in Go means they silently get swallowed or returned as 500s with no context.

2.5.2 Search for unhandled errors: `grep -n "_, err" --include="*.go" -r .` — any `_, err :=` where `err` is never checked is a ticking clock.

2.5.3 Also look for bare `fmt.Println(err)` or `log.Println(err)` without returning an error response — the request continues after logging an error.

2.5.4 Risk assessment: **Medium.** Silent failures in a healthcare API are not acceptable at scale. SATU SEHAT integration errors in particular need structured error responses.

## 2.6 Configuration and Environment Handling (Medium Risk)

2.6.1 Laravel's `.env` + `config()` pattern is well-established. In Go the equivalent is reading env vars directly or using a library like `viper`. The risk is env vars being read inline throughout the codebase rather than loaded once at startup into a typed config struct.

2.6.2 Search for `os.Getenv` scattered through handler or service files rather than centralized in a config package.

2.6.3 Risk assessment: **Medium.** Makes configuration auditing hard and increases the chance of a missing env var causing a runtime panic rather than a clean startup failure.

## 2.7 Dependency Injection vs. Global State (Low-Medium Risk)

2.7.1 Laravel uses its service container for DI. A common Go anti-pattern from Laravel converts is using package-level global variables (a global DB connection, global config) instead of injecting dependencies through constructors or function parameters.

2.7.2 Look for `var db *gorm.DB` or similar at package level in non-`main.go` files.

2.7.3 Risk assessment: **Low-Medium.** Works fine until you need to test anything or run multiple instances with different configs. Technical debt rather than immediate risk.

## 2.8 ORM Usage Patterns (Low Risk)

2.8.1 If the team is using GORM (common Laravel-to-Go path), check for over-reliance on GORM's magic: `Preload` chains, `AutoMigrate` in production, soft deletes via `DeletedAt` without understanding the performance implications.

2.8.2 Also check whether raw SQL queries exist — if there are both GORM and raw SQL patterns in the same codebase, that's a signal the team lost confidence in the ORM at some point and started working around it, which creates inconsistency.

2.8.3 Risk assessment: **Low** for now, worth noting for the pattern guide.

---

# 3. Investigation Sequence

3.1 **Day 1 (today — pre-meeting):** Run all database queries to establish props scope. Run all grep searches in the repo. Note the specific files and line numbers — don't fix anything yet, just document.

3.2 Check `go.mod` and confirm how Postman was generated. One command: `cat go.mod | grep -i "swagger\|swag\|openapi"`.

3.3 Pick one confirmed example of each pattern found. That's all you need for the presentation — one clear example is more persuasive than a count.

---

# 4. Presentation Frame (Team Meeting)

4.1 **Lead with the good:** The Laravel-to-Go refactor is real work and the right direction. This is about making the Go code idiomatic Go, not relitigating the decision.

4.2 **Frame as patterns, not blame.** These are habits that follow any experienced Laravel developer into Go. They're not errors — they're defaults that need updating.

4.3 **Prioritize by risk to the SATU SEHAT integration.** That's the near-term constraint — if the API is leaking untyped blobs to a government health system, that's the conversation opener.

4.4 **Proposed ask of CTO:** Not a refactor sprint. A one-page pattern guide (Go equivalents of the top 3 Laravel patterns) that both the CTO and junior dev work from. The junior dev needs air cover — he's getting contradictory direction and needs a reference he can point to.

---

# 5. Companion Pattern Guide (for CTO)

5.1 This is a separate document to produce after findings are confirmed. Working title: **Laravel → Go Pattern Map**. Structure: for each pattern, show the Laravel way, explain why it doesn't transfer, show the Go way with a minimal code example from the actual codebase.

5.2 Tone: peer-to-peer, not corrective. He's smart and will respond to the reasoning, not a checklist.

5.3 Patterns to cover (pending investigation confirmation): props/JSON blob columns → typed structs, model-as-response → DTO pattern, exception bubbling → explicit error values, fat controllers → handler/service split.

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 03 Mar 2026 | Alex | Initial investigation plan. |
