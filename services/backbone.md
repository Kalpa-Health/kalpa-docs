# Service: Backbone

**Version:** 0.1 (stub)
**Date:** 01 March 2026
**Status:** Stub — content to be added
**Maintained by:** TBD (service owner)

---

## 1. Overview

1.1.1 The Backbone is the core internal service that handles tenant management, feature flags, environment configuration, and — in the current architecture — acts as the single gRPC target for the Gateway.

1.1.2 For the full system context, see [`wellmed-system-architecture.md §2.1`](../wellmed-system-architecture.md).

---

## 2. Repository

| Item | Value |
|------|-------|
| **Repo** | TBD |
| **Language** | Go |
| **gRPC address** | `BACKBONE_GRPC_ADDRESS` (configured in gateway) |

---

## 3. Key Responsibilities

- [ ] TODO: Document backbone domain ownership
- Tenant management and provisioning
- Feature flags per tenant
- Environment configuration and co-dependency resolution
- (Current state) Proxies all 39 Gateway domain modules to their respective business logic

---

## 4. Service-Specific Documentation

- [ ] TODO: Link to backbone service repo once created under `Kalpa-Health` org

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 01 Mar 2026 | Alex + Claude | Initial stub |
