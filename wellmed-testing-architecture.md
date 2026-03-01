# WellMed Testing Architecture & Strategy

**Version:** 1.0
**Date:** 01 March 2026
**Previous Version:** N/A — reformatted from `archive/wellmed-testing-architecture-v7.md`
**Maintained by:** Alex

### Key Changes

- Initial canonical version. Removed system architecture content (now in `wellmed-system-architecture.md`). Removed executed Week 1 checklist and backup strategy (both in system architecture). Retained all testing strategy, patterns, taxonomy, CI/CD configuration, and monitoring content. Reformatted to numbered heading style.

---

# 1. Testing Culture & Values

## 1.1 The Kalpa Testing Commitment

> **"The sturdiness of the Kalpa codebase will only increase over time."**

1.1.1 Testing is not optional. It is not something we do "when we have time." It is how we build software that we can trust, that our clients can trust, and that we can confidently change without fear.

1.1.2 Many development teams treat testing as an afterthought — write the code first, maybe add tests later if there's time. This is the "cowboy" approach, and it creates fear of making changes, bugs that reach production, and technical debt that compounds over time. At Kalpa, we choose a different path. We build the testing muscle now, while we're small, so it becomes automatic as we grow.

1.1.3 **Collective commitment, individual responsibility.**

| Level | Responsibility |
|-------|----------------|
| **Company** | We commit to building testable software and providing the tools and training to do it |
| **Team** | We hold each other accountable; we don't merge untested code; we help each other learn |
| **Individual** | Each developer owns the tests for their code; if you write it, you test it |

1.1.4 **What "owning your tests" means.** When you write a function, you also write the test that proves it works. The function is not "done" until: (1) the code works, (2) a test proves it works, (3) the test runs automatically in CI.

## 1.2 Core Testing Values

### 1.2.1 SAGA Everything We Can

> **Every write operation must be reversible.**

Any function that creates or modifies data must have a corresponding compensation (undo) function. This enables clean rollback on multi-step failures, data integrity across microservices, and easier debugging and recovery.

```go
type ReversibleOperation interface {
    Execute(ctx context.Context) error
    Compensate(ctx context.Context) error
}
```

### 1.2.2 The Coverage Trap

> **Coverage is a floor, not a goal.**

High coverage numbers can create false confidence. A test that executes code without verifying correct behavior provides coverage but no value.

```go
// BAD: Has coverage, no value
func TestCalculateFee(t *testing.T) {
    result := CalculateFee(100)
    _ = result  // Test passes if no panic, but verifies nothing
}

// GOOD: Meaningful assertion
func TestCalculateFee(t *testing.T) {
    result := CalculateFee(100)
    assert.Equal(t, 85000, result.Amount)
    assert.Equal(t, "IDR", result.Currency)
}
```

### 1.2.3 External Robustness from Our Side

> **Assume external APIs will fail. Build resilience into our code.**

Government APIs (SATU SEHAT, P-Care) are notoriously unreliable and poorly documented. We cannot depend on their stability. Therefore: all external calls are async via RabbitMQ, retry with exponential backoff, circuit breakers prevent cascade failures, dead letter queues capture all failures for replay, and contract verification runs daily to detect API changes.

### 1.2.4 Speed is a Feature

> **Fast tests enable fast iteration.**

| Stage | Maximum Time | Action if Exceeded |
|-------|--------------|-------------------|
| Unit tests (CI) | 8 minutes | Optimize tests or parallelize |
| Integration tests | 12 minutes | Review test scope |
| **Total build time** | **15 minutes** | Mandatory discussion before merge |

If total build time exceeds 15 minutes, the PR cannot merge until tests are optimized or the team documents a justification for raising the threshold.

### 1.2.5 Replicable Testing Framework

> **Every new service gets the same testing foundation.**

Testing patterns, utilities, and structure are templated so that adding a new service automatically includes unit test scaffolding, integration test patterns, mock utilities, and coverage configuration.

### 1.2.6 ULID for Collision Avoidance

> **Never generate sequential IDs in application code.**

Use `github.com/oklog/ulid/v2` for all identifiers.

| Why ULID over UUID | Detail |
|-------------------|--------|
| **Sortable** | Time-based prefix keeps database indexes ordered, improving insert performance |
| **Timestamp built-in** | First 48 bits encode creation time (useful for debugging) |
| **Shorter** | 26 characters vs 36 for UUID |
| **Same collision resistance** | 128 bits total, 80 bits of randomness per millisecond |

---

# 2. Code Architecture for Testing

## 2.1 The Four Layers

2.1.1 Every WellMed service follows the same four-layer pattern. Understanding where code lives is essential for understanding where tests live. For the full directory structure, see [`wellmed-system-architecture.md §6`](wellmed-system-architecture.md).

```
/internal/domain/{module}/
├── handler/       # Layer 1: HTTP/gRPC entry point (30% coverage target)
├── service/       # Layer 2: Business logic (80% coverage target)
├── repository/    # Layer 3: Database access (integration tests only)
└── dto/           # Layer 4: Data structures (no tests)
```

## 2.2 Layer Responsibilities

| Layer | What it does | What it does NOT do | Test strategy |
|-------|--------------|---------------------|---------------|
| **Handler** | Parse requests, validate format, call service, format response | Make business decisions, touch database | Light testing (30%) — mostly plumbing |
| **Service** | ALL business logic: calculations, validations, rules, orchestration | Direct database access, HTTP parsing | Heavy testing (80%) — bugs hurt most here |
| **Repository** | Execute SQL, map rows to structs | Business logic, validation | Integration tests only — need real database |
| **Domain/Utilities** | Shared utilities (validators, types, cache helpers) | Service-specific logic | Medium testing (50%) |

2.2.1 **Handler layer example.** Handlers are intentionally "dumb" — they just pass data through. Most handler bugs are caught by integration tests.

```go
func (h *VisitHandler) CreateVisit(ctx context.Context, req *pb.CreateVisitRequest) (*pb.Visit, error) {
    if req.PatientId == "" {
        return nil, status.Error(codes.InvalidArgument, "patient_id required")
    }
    visit, err := h.service.CreateVisit(ctx, req.PatientId, req.DoctorId)
    if err != nil {
        return nil, err
    }
    return toProto(visit), nil
}
```

2.2.2 **Service layer example.** This is where all business decisions live and where we test the most.

```go
func (s *VisitService) CreateVisit(ctx context.Context, patientID, doctorID string) (*Visit, error) {
    patient, err := s.patientRepo.GetByID(ctx, patientID)
    if err != nil {
        return nil, err
    }
    if patient.Status != "active" {
        return nil, ErrPatientInactive
    }
    if !s.scheduleService.IsDoctorAvailable(ctx, doctorID, time.Now()) {
        return nil, ErrDoctorUnavailable
    }
    visit := &Visit{
        ID:          ulid.Make().String(),
        PatientID:   patientID,
        DoctorID:    doctorID,
        VisitNumber: s.generateVisitNumber(time.Now()),
        Status:      VisitStatusCreated,
        CreatedAt:   time.Now(),
    }
    return s.visitRepo.Create(ctx, visit)
}
```

2.2.3 **Repository layer.** You cannot meaningfully test SQL without a real database. Unit testing repositories with mocks just tests that you called the mock correctly — useless. We test repositories through integration tests with a real Postgres instance.

## 2.3 Coverage Targets

| Layer | Minimum Coverage | Rationale |
|-------|-----------------|-----------|
| Handler | 30% | Thin layer, mostly plumbing. Bugs caught by integration tests. |
| Service | 80% | Business logic lives here. Bugs hurt. Test thoroughly. |
| Domain/Utilities | 50% | Important but less critical than business logic. |
| Repository | Via integration tests | Cannot meaningfully unit test SQL. Use real database. |

## 2.4 Measuring Coverage

```bash
# Generate coverage report
go test -coverprofile=coverage.out ./...

# View coverage by function
go tool cover -func=coverage.out

# View coverage in browser (highlighted code)
go tool cover -html=coverage.out
```

2.4.1 CI will **warn** (not fail) if coverage drops below thresholds. We trust developers to make judgment calls. If a PR significantly drops coverage, expect questions in review.

---

# 3. Testing Taxonomy

## 3.1 Test Categories

3.1.1 Tests are organized into seven categories. Understanding which to write for which situation prevents both under-testing and over-testing.

| Category | Scope | Location | When it runs |
|----------|-------|----------|-------------|
| **A — Unit** | Single function/method, all deps mocked | `*_test.go` alongside source | Every PR |
| **B — Integration (Internal)** | Multiple WellMed components together (service+DB, service+queue) | `/tests/integration/` | On merge to `develop` |
| **C — Contract (External)** | External API response format verification | `/tests/contracts/` | Mock tests in CI; real verification daily |
| **D — End-to-End** | Full user workflows through UI | Manual checklist (Playwright future) | Before major releases |
| **E — Performance** | Benchmarks, load tests, slow query detection | `/tests/performance/` | Before major releases |
| **F — Permission** | RBAC enforcement across all resources | `/tests/permissions/` | Every PR (part of unit suite) |
| **G — SAGA** | Multi-service transaction flows with compensation | `/tests/sagas/` | Part of integration tests |

## 3.2 Test Folder Structure

3.2.1 Unit tests live **in the same directory** as the code they test. This is Go convention and non-negotiable.

```
/internal
└── /domain
    └── /emr
        ├── /handler
        │   ├── visit_handler.go
        │   └── visit_handler_test.go      # Category A
        ├── /service
        │   ├── visit_service.go
        │   └── visit_service_test.go      # Category A (most important)
        └── /repository
            └── visit_repo.go              # No unit tests

/tests
├── /integration                           # Category B
│   ├── /emr
│   │   ├── visit_flow_test.go
│   │   └── encounter_sync_test.go
│   └── /cross_service
│       └── emr_cashier_test.go
├── /contracts                             # Category C
│   ├── /satu_sehat
│   │   ├── patient_contract_test.go
│   │   └── mocks/satu_sehat_mock.go
│   └── /pcare
├── /permissions                           # Category F
│   └── matrix_test.go
├── /sagas                                 # Category G
│   ├── create_visit_saga_test.go
│   └── compensation_test.go
└── /performance                           # Category E
    ├── /benchmarks
    │   └── visit_service_bench_test.go
    └── /k6
        └── visit_load_test.js
```

## 3.3 Running Tests

```bash
# Run all unit tests (fast, <8 minutes)
go test ./...

# Run only a specific service's unit tests
go test ./internal/domain/emr/...

# Run integration tests (needs Docker services running)
go test -tags=integration ./tests/integration/...

# Run contract tests
go test -tags=contract ./tests/contracts/...

# Run benchmarks
go test -bench=. ./tests/performance/benchmarks/...

# Run with race detector (recommended for CI)
go test -race ./...
```

---

# 4. Testing Patterns

## 4.1 SAGA Pattern & Testing

### 4.1.1 Pattern Structure

The SAGA orchestrator coordinates all steps from a single point. Each step implements `Execute` and `Compensate`. On failure, the orchestrator calls `Compensate` on all previously completed steps in reverse order.

```go
type Step interface {
    Name() string
    Execute(ctx context.Context) error
    Compensate(ctx context.Context) error
}

type Saga struct {
    name  string
    steps []Step
}

func (s *Saga) Execute(ctx context.Context) error {
    var completed []Step
    for _, step := range s.steps {
        if err := step.Execute(ctx); err != nil {
            for i := len(completed) - 1; i >= 0; i-- {
                if compErr := completed[i].Compensate(ctx); compErr != nil {
                    log.Error("Compensation failed",
                        "saga", s.name,
                        "step", completed[i].Name(),
                        "error", compErr)
                }
            }
            return fmt.Errorf("saga %s failed at step %s: %w", s.name, step.Name(), err)
        }
        completed = append(completed, step)
    }
    return nil
}
```

### 4.1.2 SAGA Test Pattern

```go
func TestCreateMCUVisitSaga_FailureAtStep3_CompensatesSteps1And2(t *testing.T) {
    radiologyService.ForceError(ErrNoAvailableSlots)

    saga := NewCreateMCUVisitSaga(testRequest)
    err := saga.Execute(ctx)

    assert.Error(t, err)
    assert.Contains(t, err.Error(), "ReserveRadiologySlots")

    // Verify orchestrator compensated previous steps
    _, err = emrService.GetVisit(ctx, expectedVisitID)
    assert.Error(t, err) // Visit deleted by compensation

    labSlots := labService.GetReservedSlots(ctx, testRequest.PatientID)
    assert.Empty(t, labSlots) // Lab slots released by compensation
}
```

### 4.1.3 Why Orchestration, Not Choreography

| Orchestration ✅ | Choreography ❌ |
|-----------------|----------------|
| Centralized control and visibility | Requires event bus infrastructure |
| Easier to understand and debug | Harder to track saga state across services |
| Simpler error handling and compensation | More complex debugging of distributed flows |
| Clear transaction boundaries | Overkill for our current scale |

## 4.2 State Machine Pattern

4.2.1 For entities that move through states (visits, orders), use a formal state machine with an audit trail.

```go
const (
    VisitCreated        = "created"
    VisitInProgress     = "in_progress"
    VisitPendingPayment = "pending_payment"
    VisitPaid           = "paid"
    VisitComplete       = "complete"
    VisitCancelled      = "cancelled"
)

var ValidTransitions = map[string][]string{
    VisitCreated:        {VisitInProgress, VisitCancelled},
    VisitInProgress:     {VisitPendingPayment, VisitPaid, VisitCancelled},
    VisitPendingPayment: {VisitPaid, VisitCancelled},
    VisitPaid:           {VisitComplete},
    VisitComplete:       {},
    VisitCancelled:      {},
}
```

4.2.2 **Audit trail.** Every state transition is recorded with who triggered it and from which service.

```go
type StateTransition struct {
    ID          ulid.ULID
    EntityType  string
    EntityID    ulid.ULID
    FromState   string
    ToState     string
    TriggeredBy ulid.ULID
    ServiceName string
    Timestamp   time.Time
    Metadata    map[string]interface{}
}
```

4.2.3 **Configurable flows.** The transition map is reconfigurable per deployment to support standard flow (service → payment → complete), MCU flow (all services → payment → complete), and postpaid flow (service → complete → deferred payment).

## 4.3 Permission Testing

4.3.1 Test RBAC enforcement with a matrix-based approach driven by `permissions.yaml`.

```yaml
# /config/permissions.yaml
resources:
  patient:
    create: [admin, receptionist]
    read: [admin, doctor, nurse, receptionist, lab_tech]
    update: [admin, doctor, nurse]
    delete: [admin]
  prescription:
    create: [admin, doctor]
    read: [admin, doctor, nurse, pharmacist]
    update: [admin, doctor]
    delete: [admin]
```

```go
func TestPermissionMatrix(t *testing.T) {
    matrix := LoadPermissionMatrix("permissions.yaml")
    for resource, actions := range matrix.Resources {
        for action, allowedRoles := range actions {
            for _, role := range AllRoles {
                t.Run(fmt.Sprintf("%s_%s_%s", role, action, resource), func(t *testing.T) {
                    user := CreateTestUserWithRole(role)
                    allowed := HasPermission(user, resource, action)
                    shouldBeAllowed := Contains(allowedRoles, role)
                    assert.Equal(t, shouldBeAllowed, allowed)
                })
            }
        }
    }
}
```

## 4.4 Contract Testing (External APIs)

4.4.1 Run daily verification against real external APIs to detect schema changes before they break production.

| API | Endpoints verified | Frequency |
|-----|-------------------|-----------|
| SATU SEHAT | Patient, Encounter, Observation | Daily |
| P-Care | Pendaftaran, Kunjungan, Rujukan | Daily |
| Jurnal | Contacts, Invoices, Payments | Daily |
| Xendit | Payment, Invoice, Callback | Daily |
| Mobile JKN | TBD (Q3 roadmap) | — |

4.4.2 Store verification results with timestamps to track API evolution:

```json
{
  "timestamp": "2026-03-01T06:00:00Z",
  "api": "satu-sehat",
  "endpoint": "/fhir-r4/v1/Patient",
  "status": "changed",
  "changes": [
    "New field 'telecom.rank' added",
    "Response time increased 200ms → 450ms"
  ]
}
```

## 4.5 Proto-Based Service Contracts

4.5.1 Use Protocol Buffers (`.proto` files) as the single source of truth for service contracts. Benefits: compile-time type safety, auto-generated API docs, visible breaking changes, cross-service validation.

```
/proto
├── common/types.proto        # Shared types (Money, DateTime)
├── emr/v1/emr.proto          # EMR service contract
├── billing/v1/billing.proto
└── ...
```

## 4.6 Caching Strategy Testing

4.6.1 Use Redis for caching shared data. All caches use a **push-based pattern** (pub/sub) with a safety TTL that auto-resets if no update is received.

| Data type | Method | Safety TTL | Rationale |
|-----------|--------|-----------|-----------|
| Patient demographics | Push (pub/sub) | 1 month | Rarely changes |
| Service catalog | Push (pub/sub) | 1 month | Changes weekly at most |
| Prices | Push (pub/sub) | 1 month | Event-driven updates ensure freshness |
| Visit status | Push (pub/sub) | 15 minutes | Must stay fresh |
| Queue position | **Never cache** | — | Real-time critical; always query live |

4.6.2 **Cache update pattern.** Write to DB, then broadcast new value via pub/sub.

```go
func (s *PatientService) UpdatePatient(ctx context.Context, patient *Patient) error {
    if err := s.repo.Update(ctx, patient); err != nil {
        return err
    }
    s.redis.Publish(ctx, "patient:updated", patient.ID)
    s.cache.SetWithTTL("patient:"+patient.ID, patient, 30*24*time.Hour)
    return nil
}
```

## 4.7 Queue Architecture

4.7.1 Queue topology is documented in `/config/queues.yaml`. Key properties for each queue: exchange, routing key, retry policy, and DLQ.

```yaml
# /config/queues.yaml
queues:
  satu_sehat_sync:
    exchange: integrations
    routing_key: sync.satu_sehat.#
    retry:
      max_attempts: 5
      initial_delay: 2s
      max_delay: 5m
      multiplier: 2
    dlq: satu_sehat_sync_dlq

  pcare_sync:
    exchange: integrations
    routing_key: sync.pcare.#
    retry:
      max_attempts: 10
      initial_delay: 500ms
      max_delay: 10m
      multiplier: 2
    dlq: pcare_sync_dlq
```

4.7.2 DLQ log files: `/var/log/wellmed/dlq/{api-name}.log` — one file per queue for easy debugging.

---

# 5. CI/CD Configuration

## 5.1 Local Development Gate (Pre-Push)

5.1.1 Before pushing code to any branch, developers must pass these local checks:

```bash
make lint     # golangci-lint — must pass with no errors
make test     # Unit tests — must pass
make build    # Service must compile
```

5.1.2 **Standard Makefile template.**

```makefile
.PHONY: lint test build all

all: lint test build

lint:
	golangci-lint run --timeout=3m ./...

test:
	go test -v -race -timeout=3m ./...

build:
	go build -o bin/service ./cmd/...

pre-commit: lint test
```

5.1.3 Configure your IDE to run `gofmt` and `go vet` on save.

## 5.2 Branch & Merge Rules

For the authoritative branch governance table, see [`wellmed-system-architecture.md §7.2`](wellmed-system-architecture.md).

| Level | Scope | Who can merge |
|-------|-------|---------------|
| Green | Minor changes, bug fixes | All developers (CI passes) |
| Yellow | Database migrations, schema changes | Senior devs (Fajri, Hamzah) |
| Red | New modules, major features, prod deploy | CTO only |

**Flow:** `feature branch` → `develop` → `staging` → `production`

## 5.3 GitHub Actions Workflows

### 5.3.1 Full CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [develop]

jobs:
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - name: Run linters
        uses: golangci/golangci-lint-action@v4
        with:
          args: --timeout=3m

  unit-test:
    runs-on: ubuntu-latest
    timeout-minutes: 8
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - name: Run unit tests
        run: |
          go test -v -race -coverprofile=coverage.out -timeout=3m ./...
      - name: Check coverage
        run: |
          go tool cover -func=coverage.out | tee coverage.txt
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.out

  integration-test:
    runs-on: ubuntu-latest
    timeout-minutes: 12
    needs: unit-test
    if: github.event_name == 'push' && github.ref == 'refs/heads/develop'
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: wellmed_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        ports:
          - 6379:6379
      rabbitmq:
        image: rabbitmq:3-management
        ports:
          - 5672:5672
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - name: Run integration tests
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/wellmed_test
          REDIS_URL: redis://localhost:6379
          RABBITMQ_URL: amqp://localhost:5672
        run: |
          go test -v -tags=integration -timeout=5m ./tests/integration/...
```

### 5.3.2 Claude PR Review Workflow

```yaml
# .github/workflows/pr-review.yml
name: PR Review

on:
  pull_request:
    branches: [main, develop]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Generate diff
        run: git diff origin/${{ github.base_ref }}..HEAD > diff.txt
      - name: Generate review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          DIFF_CONTENT=$(cat diff.txt | head -c 50000)
          curl -s https://api.anthropic.com/v1/messages \
            -H "Content-Type: application/json" \
            -H "x-api-key: $ANTHROPIC_API_KEY" \
            -H "anthropic-version: 2023-06-01" \
            -d @- <<EOF | jq -r '.content[0].text' > review.md
          {
            "model": "claude-sonnet-4-6",
            "max_tokens": 2048,
            "messages": [{
              "role": "user",
              "content": "Review this code diff for WellMed healthcare clinic management system. Provide:\n\n1. **Ringkasan Perubahan** (2-3 kalimat dalam Bahasa Indonesia)\n2. **Risk Level**: Low/Medium/High dengan penjelasan\n3. **Potential Issues**: Bug atau masalah yang mungkin\n4. **Recommendations**: Saran untuk reviewer\n5. **ADR Needed?**: Apakah perubahan ini memerlukan Architecture Decision Record?\n\nDiff:\n\n$DIFF_CONTENT"
            }]
          }
          EOF
      - name: Post review comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const review = fs.readFileSync('review.md', 'utf8');
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## 🤖 AI Code Review\n\n${review}\n\n---\n*Generated by Claude. Human review still required.*`
            });
```

## 5.4 Service Version Management

5.4.1 All services use semantic versioning: `MAJOR.MINOR.PATCH` — Major for breaking API changes, Minor for backward-compatible new features, Patch for bug fixes.

5.4.2 **Service compatibility matrix** (`/config/compatibility.yaml`):

```yaml
services:
  emr-service:
    version: 1.5.0
    requires:
      patient-service: ">=1.2.0"
      billing-service: ">=1.0.0"
  patient-service:
    version: 1.3.0
    requires: {}
```

5.4.3 Before deployment, CI validates that the new version is compatible with all running services.

---

# 6. Frontend Monitoring

## 6.1 Current State

6.1.1 Frontend quality assurance is currently manual QA with a checklist. The two tools below are planned for the next phase.

## 6.2 Playwright (Pre-Production)

6.2.1 Automated browser testing — simulates real users clicking through the app. Catches broken buttons, failed page loads, and UI regressions. Runs in CI/CD before deployment.

6.2.2 **Use case:** Automate the current manual E2E checklist. Instead of QA manually clicking create visit → add service → payment, Playwright does it automatically on every PR.

## 6.3 Sentry (Production)

6.3.1 Error monitoring — catches JavaScript crashes when real users hit them. Free tier: 5K errors/month (sufficient for early stage). Setup estimate: ~2 hours.

6.3.2 **Triage workflow (when both tools are in place):**

```
Development → CI/CD → Staging → Production
                ↑                    ↑
            Playwright            Sentry
          (catches before       (catches what
           deployment)          slips through)
```

## 6.4 Smoke Tests

6.4.1 Basic smoke tests to run after each staging deployment:

```bash
#!/bin/bash
# scripts/smoke_test.sh

BASE_URL="https://staging.wellmed.id"
FAILED=0

check_endpoint() {
    local name=$1 url=$2 expected=$3
    status=$(curl -s -o /dev/null -w "%{http_code}" "$url")
    if [ "$status" != "$expected" ]; then
        echo "FAIL: $name returned $status (expected $expected)"
        FAILED=1
    else
        echo "PASS: $name"
    fi
}

check_endpoint "Health"      "$BASE_URL/health"      "200"
check_endpoint "Login page"  "$BASE_URL/login"       "200"
check_endpoint "API status"  "$BASE_URL/api/v1/status" "200"

[ $FAILED -eq 1 ] && exit 1
echo "All smoke tests passed"
```

---

# 7. Monitoring Dashboard

## 7.1 Goals

7.1.1 The monitoring dashboard (hosted in WellMed HQ, protected by the same sign-in flow) provides a single view showing: VM and instance health, service status and response times, deployment history, system alerts (errors, DLQ depth), and backup status.

## 7.2 Data Sources

| Source | Data | Collection method |
|--------|------|-------------------|
| AWS CloudWatch | CPU, memory, disk, network, logs | CloudWatch API |
| AWS X-Ray | Request traces across services | X-Ray API |
| Service `/health` endpoints | Up/down, response times | Poll every 60s |
| PostgreSQL | Connections, slow queries | `pg_stat_*` |
| RabbitMQ | Queue depths, message rates | Management API |
| ElasticSearch | Cluster health | Cluster health API |
| GitHub Actions | Build status, deployments | GitHub API |
| DLQ tables | Failed message count | Direct query |

## 7.3 Evolution Path

| Phase | Scope |
|-------|-------|
| Phase 1 (now) | Simple `/status` page showing service health |
| Phase 2 | Full dashboard with read-only access for team |
| Phase 3 | Integrate with Claude for automatic bug logging (Ralph loops) |

---

# 8. Library Reference

## 8.1 Go Libraries

| Library | Purpose |
|---------|---------|
| `stretchr/testify` | Assertions, mocks |
| `oklog/ulid/v2` | ULID generation |
| `avast/retry-go` | Retry with backoff |
| `streadway/amqp` | RabbitMQ client |
| `go-redis/redis` | Redis client |
| `go-playground/validator` | Struct validation |
| `uber-go/zap` | Structured JSON logging |

## 8.2 Development Tools

| Tool | Purpose |
|------|---------|
| `golangci-lint` | Comprehensive linter |
| `go vet` | Static analysis |
| `gofmt` | Code formatting |
| `gosec` | Security scanning |
| `k6` | Load testing |
| `protoc` | Protocol buffer compiler |

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 01 Mar 2026 | Alex + Claude | Reformatted from `archive/wellmed-testing-architecture-v7.md`. Removed system architecture content (now in `wellmed-system-architecture.md`). Removed backup strategy, SLOs, executed Week 1 checklist. Retained all testing strategy, patterns, taxonomy, CI/CD, monitoring content. Reformatted to numbered heading style. |
