# Service: Gateway

**Version:** 0.1 (stub)
**Date:** 01 March 2026
**Status:** Stub — link to service repo README for full details
**Maintained by:** Alex

---

## 1. Overview

1.1.1 The Gateway is the single HTTP entry point for all WellMed API calls. It receives requests from the frontend BFFs, authenticates via JWT + Redis session store, and proxies to internal microservices via gRPC.

1.1.2 For the full system context, see [`wellmed-system-architecture.md §2.3`](../wellmed-system-architecture.md).

---

## 2. Repository

| Item | Value |
|------|-------|
| **Repo** | [Kalpa-Health/wellmed-gateway-go](https://github.com/Kalpa-Health/wellmed-gateway-go) |
| **Language** | Go |
| **HTTP Framework** | Fiber v3 |
| **Module path** | `github.com/ingarso/kalpa-gateway` (pending update — see QUESTIONS.md §2.1) |

---

## 3. Key Characteristics

- **39 domain modules** — each follows handler → service → RPC client pattern
- **No database access** — all data flows through gRPC to internal services
- **Single backbone connection** (current state) — target state is direct per-service gRPC connections (see QUESTIONS.md §4.5)
- **JWT + Redis auth** — session tokens validated against Redis on every request
- **Rate limiting** — applied at the login endpoint (4 req/min per IP+path)

---

## 4. Service-Specific Documentation

The following docs live in the service repo at `wellmed-gateway-go/docs/`:

| Document | What it covers |
|----------|----------------|
| `gateway-structure.md` | Full directory map, file counts |
| `gateway-modules.md` | All 39 module service interfaces |
| `gateway-integrations.md` | gRPC integration, circuit breaker, interceptors |
| `gateway-middleware.md` | Auth/JWT/gRPC middleware chain |
| `gateway-test-baseline.md` | Test baseline and reference patterns |
| `gateway-test-coverage.md` | Current test coverage state |
| `repo-governance-transfer.md` | Checklist for transferring repo to org |

---

## 5. Open Questions

See [`QUESTIONS.md`](../QUESTIONS.md) for open architectural questions flagged during the gateway codebase analysis, including the two-JWT-secret design, middleware claim forwarding, and the routing model (all-via-backbone vs. direct per-service).

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 01 Mar 2026 | Alex + Claude | Initial stub |
