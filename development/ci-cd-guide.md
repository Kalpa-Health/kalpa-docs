# CI/CD Guide

**Version:** 0.1 (stub)
**Date:** 01 March 2026
**Status:** Stub — workflow overview captured; deployment details to be added
**Maintained by:** Alex, Hamzah

> For the authoritative CI/CD configuration including full GitHub Actions YAML, see [`wellmed-testing-architecture.md §5`](../wellmed-testing-architecture.md).
> For branch governance and merge rules, see [`wellmed-system-architecture.md §7`](../wellmed-system-architecture.md).

---

## 1. Pipeline Overview

```
feature branch
    └── PR to develop
            ├── lint (golangci-lint, 5 min limit)
            ├── unit-test (8 min limit)
            └── Claude PR review comment (auto-posted)

develop (on merge)
    └── integration-test (12 min limit, needs Postgres + Redis + RabbitMQ)

staging (on merge from develop)
    └── [same as develop + manual staging smoke test]

production (CTO only, Red-level merge)
    └── [deployment steps TBD]
```

---

## 2. GitHub Secrets Required

| Secret | Purpose | Where used |
|--------|---------|------------|
| `ANTHROPIC_API_KEY` | Claude PR review | `pr-review.yml` |
| `AWS_ACCESS_KEY_ID` | Future — deployment | Not yet configured |
| `AWS_SECRET_ACCESS_KEY` | Future — deployment | Not yet configured |

---

## 3. Workflow Files

| File | Trigger | What it does |
|------|---------|-------------|
| `.github/workflows/ci.yml` | PR + push to develop | Lint → unit tests → integration tests |
| `.github/workflows/pr-review.yml` | PR opened | Posts Claude review comment |
| `.github/workflows/pr-doc-check.yml` | PR opened | Advisory check for doc updates |

---

## 4. Local Pre-Push Gate

```bash
make lint     # Must pass — golangci-lint
make test     # Must pass — unit tests
make build    # Must compile
```

---

## 5. Deployment

- [ ] TODO: Document deployment process for each environment (dev, staging, production)
- [ ] TODO: Document ECS task definition update process
- [ ] TODO: Document how to monitor a deployment (CloudWatch, service health endpoints)
- [ ] TODO: Document rollback procedure

---

## 6. Accessing Logs

- [ ] TODO: Document CloudWatch log group naming convention per service
- [ ] TODO: Document how to query logs for a specific tenant or request ID

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 01 Mar 2026 | Alex + Claude | Stub with pipeline overview and workflow file list |
