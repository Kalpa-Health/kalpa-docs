# CI/CD Guide

**Version:** 0.2
**Date:** 02 March 2026
**Previous Version:** 0.1 (01 March 2026) — initial stub
**Status:** In Progress — bootstrap and pipeline documented; deployment details pending
**Maintained by:** Alex, Hamzah

### Key Changes v0.1 → v0.2
- Added §5: New Repo Bootstrap — documents bootstrap-repo.sh, what it does, and where templates live
- Updated §3 Workflow Files table to reflect current four-workflow standard (added main-approval-check.yml)
- Corrected ci.yml description (unit tests only at PR stage; integration tests are a future addition)

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
| `.github/workflows/ci.yml` | PR + push to main/develop | Lint → unit tests → build verification (required status checks) |
| `.github/workflows/pr-review.yml` | PR opened / updated | Posts Claude AI review comment |
| `.github/workflows/main-approval-check.yml` | PR merged to main | Notifies if required reviewer did not approve |
| `.github/workflows/pr-doc-check.yml` | PR opened | Advisory doc and coverage check (does not block merge) |

---

## 4. Local Pre-Push Gate

```bash
make lint     # Must pass — golangci-lint
make test     # Must pass — unit tests
make build    # Must compile
```

---

## 5. New Repo Bootstrap

5.1.1 Every new Kalpa-health repo is brought up to org standard using `bootstrap-repo.sh`, which lives in `wellmed-infrastructure` (currently at `infrastructure/scripts/` until that folder becomes its own repo). It creates the three-branch structure, pushes the workflow templates, and applies branch protection rules — all in one command.

5.1.2 Full usage instructions are in [`infrastructure/HOW-TO.md`](https://github.com/Kalpa-Health/wellmed-infrastructure/blob/main/HOW-TO.md). The short version:

```bash
# dry run first
./bootstrap-repo.sh Kalpa-health/<repo-name> --dry-run

# execute
./bootstrap-repo.sh Kalpa-health/<repo-name>
```

5.1.3 The workflow templates live in `infrastructure/templates/workflows/`. When the org CI standard changes, update the template there and re-run the script — it is idempotent.

5.1.4 Two manual steps are always required after bootstrap: update the `TODO:ARCH` architecture context in `pr-review.yml`, and add `ANTHROPIC_API_KEY` as a repo secret (`gh secret set ANTHROPIC_API_KEY -r Kalpa-health/<repo-name>`).

---

## 6. Deployment

- [ ] TODO: Document deployment process for each environment (dev, staging, production)
- [ ] TODO: Document ECS task definition update process
- [ ] TODO: Document how to monitor a deployment (CloudWatch, service health endpoints)
- [ ] TODO: Document rollback procedure

---

## 7. Accessing Logs

- [ ] TODO: Document CloudWatch log group naming convention per service
- [ ] TODO: Document how to query logs for a specific tenant or request ID

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 01 Mar 2026 | Alex + Claude | Stub with pipeline overview and workflow file list |
| 0.2 | 02 Mar 2026 | Alex + Claude | Added §5 New Repo Bootstrap. Updated §3 workflow table (added main-approval-check.yml, corrected ci.yml description). Renumbered §6 Deployment → §6, Accessing Logs → §7. |
