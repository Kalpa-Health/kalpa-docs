# Repo Governance — What Lives Where

**Version:** 1.2
**Date:** 02 March 2026
**Previous Version:** 1.1 (01 March 2026) — added §4.4 docs-only branch strategy
**Maintained by:** Alex

### Key Changes v1.1 → v1.2
- Added §3.3: `wellmed-infrastructure` repo — defines infrastructure as a first-class repo type alongside docs and service repos.
- Updated §4.1 decision matrix to include infrastructure tooling row.
- Added §4.5: Infrastructure repo governance — when changes to templates and scripts require review.

---

# 1. The Problem

1.1.1 WellMed is a multi-repo architecture — each service (gateway, backbone, EMR, cashier, etc.) has its own Git repository. System-level documentation (architecture overviews, ADRs, cross-service patterns, testing strategy) doesn't naturally belong in any single service repo.

1.1.2 Without a clear home for umbrella documentation, architecture knowledge stays in Google Docs, Slack threads, or people's heads. It drifts out of sync. New team members can't find it. AI agents can't reference it in context.

---

# 2. The Solution: `kalpa-docs` Repo

2.1.1 A dedicated `kalpa-docs` repository holds all cross-service and system-level documentation. It contains no application code — only markdown documents, mermaid diagrams, ADRs, and reference material.

2.1.2 Each service repo's README links to `kalpa-docs` for system-level context. Service-specific documentation (service README, service-level PLAN files, test coverage reports) stays in the service repo.

---

# 3. Repository Structure

## 3.1 `kalpa-docs` Repo

```
kalpa-docs/
├── README.md                              # Repo overview, how to navigate
├── wellmed-system-architecture.md         # System architecture (the root document)
├── wellmed-testing-architecture.md        # Testing strategy, patterns, values
├── wellmed-api-client-pattern.md          # /pkg/apiclient design + ADR
├── wellmed-working-document.md            # Consolidated decisions, work plan
├── kalpa-documentation-testing-prd.md     # Meta-plan: docs/testing culture
├── repo-governance.md                     # This document
│
├── adrs/                                  # Architecture Decision Records
│   ├── ADR-000-template.md
│   ├── ADR-001-api-client-circuit-breaker.md
│   ├── ADR-002-saga-orchestrator.md
│   └── ADR-003-tenant-isolation.md
│
├── services/                              # Per-service reference docs (authored here)
│   ├── gateway.md                         # Gateway service overview + links
│   ├── backbone.md
│   ├── emr.md
│   └── ...
│
├── operations/                            # Operational runbooks
│   ├── backup-strategy.md
│   ├── database-migrations.md
│   └── incident-response.md
│
├── development/                           # Developer guides
│   ├── setup.md                           # Local dev environment setup
│   ├── conventions.md                     # Code conventions, naming, patterns
│   ├── testing-patterns.md                # How to write tests (examples)
│   └── ci-cd-guide.md                     # GitHub Actions workflows explained
│
└── diagrams/                              # Standalone mermaid files (if needed)
    └── system-topology.mermaid
```

## 3.2 `wellmed-infrastructure` Repo

```
wellmed-infrastructure/
├── HOW-TO.md                          # How to use the bootstrap script (start here)
├── README.md                          # Repo overview
│
├── scripts/                           # Operational scripts (run manually or via CI)
│   ├── bootstrap-repo.sh              # Bootstraps a new repo to org standard
│   ├── create-iam-resources.sh        # AWS IAM setup
│   ├── install-cloudwatch-agent.sh
│   └── install-xray-daemon.sh
│
├── templates/                         # Source-of-truth templates applied by scripts
│   └── workflows/                     # GitHub Actions workflow templates
│       ├── ci.yml
│       ├── main-approval-check.yml
│       └── pr-review.yml
│
├── iam/                               # AWS IAM policy and role definitions
│   ├── policies/
│   └── roles/
│
└── cloudwatch/                        # CloudWatch agent configuration
```

3.2.1 This repo is the source of truth for anything that applies uniformly across all service repos: CI templates, branch protection standards, AWS instance configuration. If the CI standard changes, the template changes here, and `bootstrap-repo.sh` re-applies it.

3.2.2 The `wellmed-infrastructure` repo is currently a local folder (`infrastructure/`) pending creation as a standalone GitHub repo. See `HOW-TO.md §7` for the pending actions.

## 3.3 Service Repos (e.g., `kalpa-gateway`)

```
kalpa-gateway/
├── README.md                # Service-specific: what this service does, how to run it
├── PLAN.md                  # Service-specific execution plan (phases, tasks, checkpoints)
├── Makefile                 # lint, test, build targets
├── .github/workflows/       # CI/CD workflows for THIS service
│
├── docs/                    # Service-specific documentation
│   ├── gateway-structure.md           # Directory map
│   ├── gateway-modules.md             # Module inventory + interfaces
│   ├── gateway-integrations.md        # gRPC integration map
│   ├── gateway-middleware.md          # Auth/JWT/gRPC middleware chain
│   ├── gateway-architecture.mermaid   # Request flow diagram
│   ├── gateway-test-baseline.md       # Test coverage state
│   └── services/                      # Per-module service docs
│       ├── transaction.md
│       ├── user.md
│       └── ...
│
├── internal/                # Application code
├── proto/                   # Proto definitions
└── ...
```

---

# 4. What Lives Where — The Rules

## 4.1 Decision Matrix

| Document Type | Lives In | Rationale |
|-------------|----------|-----------|
| System architecture overview | `kalpa-docs` | Spans all services |
| Architecture Decision Records (ADRs) | `kalpa-docs/adrs/` | Decisions affect multiple services |
| Testing strategy & values | `kalpa-docs` | Organization-wide policy |
| API client pattern | `kalpa-docs` | Shared package used by all services |
| Cross-service working doc / decisions | `kalpa-docs` | Affects multiple teams |
| Operational runbooks | `kalpa-docs/operations/` | Not tied to one service |
| Developer setup guide | `kalpa-docs/development/` | Applies across repos |
| CI workflow templates | `wellmed-infrastructure/templates/` | Applied to all service repos |
| AWS / infra configuration | `wellmed-infrastructure/` | Not tied to one service's code |
| Bootstrap and setup scripts | `wellmed-infrastructure/scripts/` | Operate across repos, not within one |
| Service README | Service repo | Specific to that service |
| Service PLAN.md | Service repo | Execution plan for that service |
| Service-specific docs (modules, middleware, etc.) | Service repo `/docs/` | Too granular for umbrella |
| CI/CD workflows (`.github/`) | Service repo | Runs against that service's code (generated from templates) |
| Makefile | Service repo | Builds that service |
| Test files (`*_test.go`) | Service repo | Tests that service's code |

## 4.2 The Linking Pattern

4.2.1 Every service repo README includes a "System Context" section at the top that links back to `kalpa-docs`:

```markdown
## System Context

This service is the WellMed API Gateway. For system-wide architecture, see:
- [System Architecture](https://github.com/{org}/kalpa-docs/blob/main/wellmed-system-architecture.md)
- [Testing Strategy](https://github.com/{org}/kalpa-docs/blob/main/wellmed-testing-architecture.md)
- [ADRs](https://github.com/{org}/kalpa-docs/tree/main/adrs)
```

4.2.2 `kalpa-docs` cross-references service repos where relevant. For example, the system architecture doc references `kalpa-gateway/PLAN.md` for Gateway-specific execution plans.

## 4.3 When to Create a New Document vs. Update an Existing One

4.3.1 If the information affects more than one service, it goes in `kalpa-docs`. Update the relevant existing document if one covers the topic. Create a new document only if it's a genuinely new topic area.

4.3.2 If the information is specific to one service's internals, it stays in that service repo. The service's `/docs/` folder is the right place.

4.3.3 ADRs are always created in `kalpa-docs/adrs/` even if the decision primarily affects one service, because architectural decisions have system-wide implications and should be discoverable in one place.

## 4.5 Infrastructure Repo Governance

4.5.1 Changes to `wellmed-infrastructure` templates affect every repo that gets bootstrapped afterward. Treat template changes as Yellow-level: they require at least one reviewer before merge, even if the CI check is green.

4.5.2 The `bootstrap-repo.sh` script is idempotent — re-running it against an existing repo updates workflow files and overwrites protection rules. This is intentional and safe, but running it against production repos should be a deliberate action, not an accident. The `--dry-run` flag exists for this reason.

4.5.3 When Go version or approval count standards change org-wide, update the template in `wellmed-infrastructure`, document the change in its edit log, and schedule a re-bootstrap pass across all active repos.

---

## 4.4 Branch Strategy for Documentation Changes in Service Repos

4.4.1 Documentation changes in service repos do not need to travel through `develop` → `staging` → `main`. README and `docs/` updates PR **directly to `main`** from a `docs/*` branch.

4.4.2 **The rule:**

| Change type | Branch strategy |
|-------------|----------------|
| `README.md` or `docs/*.md` only — no code changed | `docs/<slug>` → PR to `main` directly (Green level) |
| Code change with coupled doc update | Doc travels with the code PR through normal flow (`develop` → `staging` → `main`) |

4.4.3 **Rationale.** A README describes what the repo *is*, not what is deployed to a specific environment. Routing docs through `develop` and `staging` creates three stale copies of the same content with no benefit. The exception is docs that are tightly coupled to unreleased code — those should travel with the code so they go live together.

4.4.4 **GitHub branch protection note.** Branch protection rules cannot distinguish docs-only from code changes. This convention is enforced by review, not by automation. The existing protection on `main` (PR required + CI passing) still applies — this rule only clarifies *which branch to target*, not that the PR gate is bypassed.

---

# 5. Maintenance

## 5.1 Who Updates What

| Repo | Primary Maintainer | Contributor Access |
|------|-------------------|-------------------|
| `kalpa-docs` | Alex | All team members can PR |
| `kalpa-gateway` | Alex (docs), Dev team (code) | Standard PR process |
| Other service repos | Respective service owners | Standard PR process |

5.1.1 `kalpa-docs` follows the same Green/Yellow/Red merge rules as code repos. Most doc updates are Green-level (CI passes → merge). ADR creation is Yellow-level (requires CTO review).

5.1.2 AI agents (Claude) can draft PRs to `kalpa-docs` for ADRs, documentation updates, and diagram generation. Human review is always required before merge.

## 5.2 Keeping Docs in Sync

5.2.1 When a service's architecture changes (new module, new integration, changed data flow), the developer must also update the relevant `kalpa-docs` document. This is enforced socially, not by CI — the PR review process should catch missing doc updates.

5.2.2 The system architecture document (`wellmed-system-architecture.md`) is the most critical to keep current. It should be reviewed quarterly or whenever a new service is added.

5.2.3 Version numbers on all documents must be incremented on every change (per the markdown style guide). This makes it easy to spot stale documents.

---

# 6. Getting Started

6.1.1 Create the `kalpa-docs` GitHub repository under the Kalpa organization.

6.1.2 Copy the existing documents into the structure defined in Section 3.1. The current files map as follows:

| Current File | New Location in `kalpa-docs` |
|-------------|------------------------------|
| `wellmed-testing-architecture-v7.md` | `wellmed-testing-architecture.md` |
| `wellmed-api-client-pattern-v1.md` | `wellmed-api-client-pattern.md` |
| `wellmed-working-document-v3.md` | `wellmed-working-document.md` |
| `kalpa-documentation-testing-prd.md` | `kalpa-documentation-testing-prd.md` |
| (new) `wellmed-system-architecture.md` | `wellmed-system-architecture.md` |
| (new) `repo-governance.md` | `repo-governance.md` |

6.1.3 Add the "System Context" linking section to the Gateway README (and other service READMEs as they get documented).

6.1.4 Create the `adrs/` directory and move ADR-001 from the working document into `adrs/ADR-001-api-client-circuit-breaker.md`.

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 01 Mar 2026 | Alex + Claude | Initial creation. Establishes kalpa-docs repo pattern, defines what lives where, provides migration path from current doc state. |
| 1.1 | 01 Mar 2026 | Alex + Claude | Added §4.4: branch strategy for docs-only changes in service repos (docs/* → main directly). |
| 1.2 | 02 Mar 2026 | Alex + Claude | Added §3.2 wellmed-infrastructure repo (structure, purpose, pending GitHub creation). Added §4.5 infrastructure repo governance. Updated §4.1 decision matrix with infrastructure rows. |
