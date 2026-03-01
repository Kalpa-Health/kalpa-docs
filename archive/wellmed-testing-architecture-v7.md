# WellMed Testing Architecture & Strategy
## Version 5.0 | January 2025

---

## Document Purpose

This document defines WellMed's testing architecture, strategy, and values. It serves as the authoritative reference for how we approach quality assurance across all products (Lite, Plus, Enterprise).

---

# Part 1: System Architecture

## 1.1 Complete System Topology

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL WORLD                                      │
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                         User-Facing                                       │   │
│  │  [Browser/UI]                                                            │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                    Government Healthcare APIs                             │   │
│  │  [SATU SEHAT]        [P-Care/BPJS]        [Mobile JKN]                   │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                    Third-Party Business Services                          │   │
│  │  [Jurnal]    [Talenta]    [Xendit]    [Xero]    [Zoho Desk]    [Kyoo]   │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       │ HTTPS/REST/JSON
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           EDGE LAYER                                             │
│                                                                                  │
│  ┌─────────────────────────────────┐   ┌─────────────────────────────────────┐  │
│  │    WELLMED FRONTEND (Nuxt.js)   │   │       HQ FRONTEND (Nuxt.js)         │  │
│  │                                 │   │                                     │  │
│  │  • Serves clinic SPA           │   │  • Serves HQ admin SPA             │  │
│  │  • Static assets               │   │  • Static assets                    │  │
│  │  • Client-side routing         │   │  • Client-side routing              │  │
│  │  • Interactive UI components   │   │  • Interactive UI components        │  │
│  └─────────────────────────────────┘   └─────────────────────────────────────┘  │
│                  │                                      │                        │
│                  ▼                                      ▼                        │
│  ┌─────────────────────────────────┐   ┌─────────────────────────────────────┐  │
│  │        WELLMED BFF (Nitro)      │   │          HQ BFF (Nitro)             │  │
│  │                                 │   │                                     │  │
│  │  • Security layer              │   │  • Security layer                   │  │
│  │  • Session cookies             │   │  • Session cookies                  │  │
│  │  • Partial data persistence    │   │  • Partial data persistence         │  │
│  │  • Route mapping               │   │  • Route mapping                    │  │
│  │  • Reverse proxy to Gateway    │   │  • Reverse proxy to Gateway         │  │
│  └─────────────────────────────────┘   └─────────────────────────────────────┘  │
│                  │                                      │                        │
│                  └──────────────────┬───────────────────┘                        │
│                                     ▼                                            │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                          GATEWAY SERVICE (Go)                             │   │
│  │                                                                           │   │
│  │  • Ingress/egress management     • Protocol translation (REST ↔ gRPC)   │   │
│  │  • Authentication/authorization  • Rate limiting                         │   │
│  │  • Circuit breaking              • Request/response logging              │   │
│  │  • External API orchestration                                            │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       │ gRPC (internal)
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        INTERNAL MICROSERVICES                                    │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │  LITE TIER (7 services)                                                   │ │
│  │  ┌──────────────────────────────────────────────────────────────────────┐  │ │
│  │  │  BACKBONE                                                             │  │ │
│  │  │  • Tenant management           • Environment logic/variables         │  │ │
│  │  │  • Feature version co-deps     • Feature flags                       │  │ │
│  │  │  • Behind-the-scenes config                                          │  │ │
│  │  └──────────────────────────────────────────────────────────────────────┘  │ │
│  │                                                                            │ │
│  │  ┌─────────┐ ┌─────────┐ ┌───────────┐ ┌─────────────┐                    │ │
│  │  │   EMR   │ │ Cashier │ │ Reporting │ │ Appointment │                    │ │
│  │  │ (core)  │ │         │ │           │ │             │                    │ │
│  │  └─────────┘ └─────────┘ └───────────┘ └─────────────┘                    │ │
│  │                                                                            │ │
│  │  ┌───────────────────────┐ ┌───────────────────────┐                      │ │
│  │  │ SATU SEHAT Integration│ │   BPJS Integration    │                      │ │
│  │  └───────────────────────┘ └───────────────────────┘                      │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │  PLUS TIER (+6 services = 13 total)                        Inherits LITE  │ │
│  │                                                                            │ │
│  │  ┌──────────┐ ┌─────────┐ ┌───────────┐ ┌────────────────────┐            │ │
│  │  │ Pharmacy │ │   Lab   │ │ Radiology │ │ Outpatient (Multi- │            │ │
│  │  │          │ │         │ │           │ │   Department)      │            │ │
│  │  └──────────┘ └─────────┘ └───────────┘ └────────────────────┘            │ │
│  │                                                                            │ │
│  │  ┌───────────────────────┐ ┌─────────────────────────────────────────────┐│ │
│  │  │     ED / IGD          │ │ Cashier+ (3rd party billing: AR/insurance,  ││ │
│  │  │                       │ │ corporate accounts → extends to BPJS)       ││ │
│  │  └───────────────────────┘ └─────────────────────────────────────────────┘│ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │  ENTERPRISE TIER (+5 services = 18 total)                  Inherits PLUS  │ │
│  │                                                                            │ │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌───────────┐ ┌───────────────────┐  │ │
│  │  │  LIS+   │ │  RIS+   │ │   MCU   │ │ Inpatient │ │ Warehouse/        │  │ │
│  │  │         │ │         │ │         │ │           │ │ Inventory Mgmt    │  │ │
│  │  └─────────┘ └─────────┘ └─────────┘ └───────────┘ └───────────────────┘  │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │  HQ SERVICE (Separate from core WellMed)                                   │ │
│  │                                                                            │ │
│  │  • Subscription management        • SaaS payments                         │ │
│  │  • User management                • Billing/invoicing                     │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
          │                              │
          │ gRPC (sync)                  │ RabbitMQ (async)
          ▼                              ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            DATA LAYER                                            │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                         POSTGRES DATABASE                                │    │
│  │                                                                          │    │
│  │   Multi-tenant architecture:                                             │    │
│  │   • Each tenant has its own database                                     │    │
│  │   • Inside each tenant DB, microservices are provisioned via schema      │    │
│  │   • Tables clustered by year to create headspace                         │    │
│  │                                                                          │    │
│  │   Example: tenant_1_db                                           │    │
│  │   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │    │
│  │   │ emr schema   │ │ cashier      │ │ pharmacy     │ │ backbone     │   │    │
│  │   │ • visits_2024│ │ schema       │ │ schema       │ │ schema       │   │    │
│  │   │ • visits_2025│ │              │ │              │ │              │   │    │
│  │   └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   │    │
│  │                                                                          │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  ┌───────────────────────┐    ┌───────────────────────┐                         │
│  │        REDIS          │    │    ELASTICSEARCH      │                         │
│  │                        │    │                       │                         │
│  │  • Session cache      │    │  • Reporting queries  │                         │
│  │  • Shared type cache  │    │  • Patient search     │                         │
│  │  • Visit build cache  │    │  • Analytics          │                         │
│  │  • Transaction cache  │    │                       │                         │
│  │    (bill during visit │    │                       │                         │
│  │    before settlement) │    │                       │                         │
│  │  • Broadcast on update│    │                       │                         │
│  └───────────────────────┘    └───────────────────────┘                         │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 1.2 Service Count Summary

| Tier | New Services | Total Services | Inherits From |
|------|--------------|----------------|---------------|
| Lite | 7 | 7 | - |
| Plus | 6 | 13 | Lite |
| Enterprise | 5 | 28 | Plus |
| HQ | 1 (separate) | - | Standalone |

## 1.3 Communication Patterns

| Pattern | Use Case | Example |
|---------|----------|---------|
| **gRPC (sync)** | Immediate response needed | Get patient details for UI display |
| **RabbitMQ (async)** | Can wait, needs reliability | Sync visit to SATU SEHAT |
| **REST/JSON (external)** | All external API communication | P-Care claims submission |
| **Redis** | Cache invalidation broadcast (push/pull choice between pub/sub and time based refresh) | Price update notification |

## 1.4 Infrastructure Ownership & Key Decisions

### Infrastructure Ownership

| System | Owner | Notes |
|--------|-------|-------|
| AWS (VMs, CloudWatch, X-Ray, Secrets Manager) | Hamzah | All infrastructure |
| PostgreSQL | Hamzah | Per-tenant databases |
| Redis | Hamzah | Caching layer |
| RabbitMQ | Hamzah | Message queues |
| ElasticSearch | Hamzah | Search & analytics |
| GitHub Actions / CI | Alex | Pipeline configuration |

### Key Architecture Decisions (Confirmed)

| Decision | Status | Notes |
|----------|--------|-------|
| **SAGA Orchestrator pattern** | ✅ Confirmed | Centralized coordination (not choreography) |
| **Proto/gRPC** | ✅ All-in | Already built and in use |
| **AWS X-Ray for tracing** | ✅ Confirmed | Request tracing across services |
| **AWS Secrets Manager** | ✅ Confirmed | For API tokens and credentials |
| **JSON logs (uber-go/zap)** | ✅ Confirmed | Including Redis; CloudWatch aggregation |
| **Tenant isolation** | ✅ Separate DBs | Each tenant has own database, schemas per service |

### Response Time SLOs (Monitoring Thresholds)

To be configured in CloudWatch alarms:

| Threshold | Level | Action |
|-----------|-------|--------|
| < 200ms | 🟢 Green | Normal |
| 200-500ms | 🟡 Yellow | Monitor |
| 500-1000ms | 🟠 Orange | Investigate |
| > 1000ms | 🔴 Red | Alert |

## 1.5 Application Folder Hierarchy

Understanding where code lives is essential before understanding where tests live. WellMed follows standard Go project conventions with clear separation between layers.

### Why This Structure Matters

Each service follows the same internal organization. This consistency means:
- **New team members** can navigate any service immediately
- **Code reviews** are faster because reviewers know where to look
- **Tests** have predictable locations
- **Bugs** are easier to isolate to a specific layer

### The Four Layers
Every module contains four layers, each with a specific job:
```
/internal/domain/emr/             # Example: EMR module
├── /handler                      # LAYER 1: Request/Response
│   ├── visit_handler.go         # Receives gRPC/HTTP requests
│   └── visit_handler_test.go    # Handler tests (30% coverage target)
│
├── /service                      # LAYER 2: Business Logic  
│   ├── visit_service.go         # ALL medical/clinic rules live here
│   └── visit_service_test.go    # Service tests (80% coverage target)
│
├── /repository                   # LAYER 3: Database Access
│   ├── visit_repo.go            # SQL queries, data persistence
│   └── (no unit tests)          # Tested via integration tests
│
└── /DTO                        # LAYER 4: Data structures (DTOs, entities)
    └── visit_request.go
```

### Project Structure
```
/internal/
├── /config/                      # App configuration
├── /exception/                   # Custom errors
├── /helper/                      # Utility functions
├── /middleware/                  # HTTP/gRPC middleware
├── /route/                       # HTTP routing
├── /rpc/                         # gRPC service implementations
├── /schema/                      # Shared DTOs, request/response structs
├── /validation/                  # Input validation logic
└── /domain/
    ├── /emr/                     # EMR module (handler, service, repo, model)
    ├── /billing/                 # Billing module
    └── /encoding/                # Encoding module

/proto/                           # Protobuf definitions
/command/                         # CLI commands
/package/                         # Shared libraries
```



### Layer Responsibilities Explained

| Layer | What It Does | What It Does NOT Do | Test Strategy |
|-------|--------------|---------------------|---------------|
| **Handler** | Parse incoming requests, validate format, call service, format response | Make business decisions, touch database | Light testing (30%) — mostly plumbing |
| **Service** | ALL business logic: calculations, validations, rules, orchestration | Direct database access, HTTP parsing | Heavy testing (80%) — this is where bugs hurt |
| **Repository** | Execute SQL, map database rows to structs | Business logic, validation | Integration tests only — need real database |
| **Model** | Define data structures | Any logic | No tests needed — just structs |

### Full Project Structure

```
/wellmed
├── /cmd                            # Application entrypoints (main.go files)
│   ├── /gateway
│   │   └── main.go
│   ├── /emr-service
│   │   └── main.go
│   └── ...
│
├── /internal                       # Private application code
│   ├── /emr                       # EMR service
│   │   ├── /handler
│   │   ├── /service
│   │   ├── /repository
│   │   └── /model
│   ├── /cashier                   # Cashier service (Lite)
│   ├── /cashier_plus              # Cashier+ service (Plus tier)
│   ├── /pharmacy                  # Pharmacy service (Plus)
│   ├── /lab                       # Lab service (Plus)
│   ├── /backbone                  # Backbone service (core infrastructure)
│   └── /hq                        # HQ service (standalone)
│
├── /pkg                            # Shared packages (importable by all services)
│   ├── /domain                    # Support infrastructure & utilities
│   │   ├── /cache                 # Redis caching utilities
│   │   ├── /validation            # Common validators (NIK, phone, etc.)
│   │   ├── /types                 # Shared types (Money, DateTime)
│   │   └── /logging               # Structured logging utilities
│   ├── /apiclient                 # Reusable API client (see companion doc)
│   ├── /saga                      # SAGA pattern implementation
│   ├── /queue                     # RabbitMQ utilities
│   └── /testutil                  # Test utilities and mocks
│
├── /proto                          # Protocol buffer definitions
├── /config                         # Configuration files (queues.yaml, permissions.yaml)
├── /docs                           # Documentation
├── /scripts                        # Utility scripts
├── /migrations                     # Database migrations
└── /tests                          # Cross-service test suites (see Section 2.4)
```

### Key Principle: /internal vs /pkg

- **`/internal`**: Code that is private to this repository. Go enforces this — other projects cannot import it.
- **`/pkg`**: Code that can be shared. If we ever split services into separate repos, `/pkg` code can be extracted.

**Rule**: If code is used by multiple services, it belongs in `/pkg`. If it's specific to one service, it stays in `/internal`.

---

# Part 2: Testing Strategy & Values

## 2.1 Testing Culture at Kalpa

> **"The sturdiness of the Kalpa codebase will only increase over time."**

Testing is not optional. It is not something we do "when we have time." It is how we build software that we can trust, that our clients can trust, and that we can confidently change without fear.

### Why This Matters for Our Team

Many development teams treat testing as an afterthought — write the code first, maybe add tests later if there's time. This is the "cowboy" approach, and it creates:
- Fear of making changes (because you might break something)
- Bugs that reach production (because no one verified the code worked)
- Technical debt that compounds over time

**At Kalpa, we choose a different path.** We build the testing muscle now, while we're small, so it becomes automatic as we grow.

### Collective Commitment, Individual Responsibility

Testing at Kalpa works like this:

| Level | Responsibility |
|-------|----------------|
| **Company** | We commit to building testable software and providing the tools/training to do it |
| **Team** | We hold each other accountable; we don't merge untested code; we help each other learn |
| **Individual** | Each developer owns the tests for their code; if you write it, you test it |

### What "Owning Your Tests" Means

When you write a function, you also write the test that proves it works. This is not extra work — it is part of the work. The function is not "done" until:

1. The code works
2. A test proves it works
3. The test runs automatically in CI

### Building This Muscle

If testing is new to you, that's okay. Everyone starts somewhere. Here's how we build the skill:

1. **Start small**: Write one test for one function. See it pass. Feel the confidence.
2. **Pair with teammates**: Watch how others write tests. Ask questions.
3. **Use the patterns**: This document provides patterns. Copy them. Adapt them.
4. **Review tests in PRs**: Reading other people's tests teaches you faster than writing alone.

Over time, writing tests becomes as natural as writing the code itself. That's the muscle we're building.

---

## 2.2 Core Testing Principles

These values guide all testing decisions at WellMed.

### Value 1: SAGA Everything We Can

> **Every write operation must be reversible.**

Any function that creates or modifies data must have a corresponding compensation (undo) function. This enables:
- Clean rollback on multi-step failures
- Data integrity across microservices
- Easier debugging and recovery

```go
type ReversibleOperation interface {
    Execute(ctx context.Context) error
    Compensate(ctx context.Context) error
}
```


### Value 2: The Coverage Trap

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


### Value 3: External Robustness from Our Side

> **Assume external APIs will fail. Build resilience into our code.**

Government APIs (SATU SEHAT, P-Care) are notoriously buggy, unstable, and poorly documented. We cannot depend on their reliability. Therefore:
- All external calls are async via RabbitMQ
- Retry with exponential backoff
- Circuit breakers prevent cascade failures
- Dead letter queues capture all failures for retry
- Contract verification runs daily to detect API changes

### Value 4: Speed is a Feature

> **Fast tests enable fast iteration.**

| Stage | Maximum Time | Action if Exceeded |
|-------|--------------|-------------------|
| Unit tests (CI) | 8 minutes | Optimize tests or parallelize |
| Integration tests | 12 minutes | Review test scope |
| **Total build time** | **15 minutes** | Mandatory discussion before merge |

If total build time exceeds 15 minutes, the PR cannot merge until either:
1. Tests are optimized
2. Team agrees to raise the threshold (requires documented justification)

**Parallelization strategy**: Explore spinning up ephemeral GitHub runners for parallel test execution to reduce wall-clock time while keeping costs manageable.

### Value 5: Replicable Testing Framework

> **Every new service gets the same testing foundation.**

Testing patterns, utilities, and structure should be templated so that adding a new service automatically includes:
- Unit test scaffolding
- Integration test patterns
- Mock utilities
- Coverage configuration

### Value 6: ULID for Collision Avoidance
> **Never generate sequential IDs in application code.**

Use `github.com/oklog/ulid/v2` for all identifiers to prevent:
- Duplicate ID issues under concurrent load
- Race conditions in distributed systems
- Test isolation problems

**Why ULID over UUID:**
- **Sortable** — Time-based prefix keeps database indexes ordered, improving insert performance
- **Timestamp built-in** — First 48 bits encode creation time (useful for debugging)
- **Shorter** — 26 characters vs 36 for UUID
- **Same collision resistance** — 128 bits total, 80 bits of randomness per millisecond

---

## 2.3 Code Layer Definitions

Before discussing where tests live, we must understand how our code is organized. Each service has four distinct layers, and understanding these is essential for knowing what to test and how.

### The Three Layers

```
Request comes in
       │
       ▼
┌─────────────┐
│   HANDLER   │  ← Receives request, validates format, calls service
└─────────────┘
       │
       ▼
┌─────────────┐
│   SERVICE   │  ← ALL business logic lives here (the brain)
└─────────────┘
       │
       ▼
┌─────────────┐
│ REPOSITORY  │  ← Talks to database (SQL queries)
└─────────────┘
       │
       ▼
   Database
```

### Layer Responsibilities in Detail

#### Handler Layer (30% coverage target)
**What it does:**
- Receives incoming requests (gRPC, HTTP, message queue)
- Validates that the request format is correct
- Calls the appropriate service method
- Formats the response

**What it does NOT do:**
- Make business decisions
- Access the database directly
- Contain logic that could be "wrong" in a business sense

**Why low coverage target:** Handlers are "dumb" — they just pass data through. Most handler bugs are caught by integration tests anyway.

```go
// Example handler — notice it has almost no logic
func (h *VisitHandler) CreateVisit(ctx context.Context, req *pb.CreateVisitRequest) (*pb.Visit, error) {
    // Validate request format (not business rules)
    if req.PatientId == "" {
        return nil, status.Error(codes.InvalidArgument, "patient_id required")
    }
    
    // Call service — the service decides if this is a valid visit
    visit, err := h.service.CreateVisit(ctx, req.PatientId, req.DoctorId)
    if err != nil {
        return nil, err
    }
    
    // Format response
    return toProto(visit), nil
}
```

#### Service Layer (80% coverage target)
**What it does:**
- Contains ALL business logic
- Makes decisions (Is this patient eligible? Is this price correct?)
- Orchestrates operations across repositories
- Enforces business rules

**What it does NOT do:**
- Parse HTTP/gRPC requests
- Execute SQL directly

**Why high coverage target:** This is where bugs hurt. A bug in business logic can mean wrong billing, wrong medical records, compliance failures. Test thoroughly.

```go
// Example service — notice all the business logic
func (s *VisitService) CreateVisit(ctx context.Context, patientID, doctorID string) (*Visit, error) {
    // Business rule: Patient must exist and be active
    patient, err := s.patientRepo.GetByID(ctx, patientID)
    if err != nil {
        return nil, err
    }
    if patient.Status != "active" {
        return nil, ErrPatientInactive  // Business decision
    }
    
    // Business rule: Doctor must be available
    if !s.scheduleService.IsDoctorAvailable(ctx, doctorID, time.Now()) {
        return nil, ErrDoctorUnavailable  // Business decision
    }
    
    // Business rule: Generate visit number with today's date
    visitNumber := s.generateVisitNumber(time.Now())  // Business logic
    
    // Create the visit
    visit := &Visit{
        ID:          uuid.New().String(),
        PatientID:   patientID,
        DoctorID:    doctorID,
        VisitNumber: visitNumber,
        Status:      VisitStatusCreated,
        CreatedAt:   time.Now(),
    }
    
    return s.visitRepo.Create(ctx, visit)
}
```

#### Repository Layer (tested via integration tests)
**What it does:**
- Executes SQL queries
- Maps database rows to Go structs
- Handles database transactions

**What it does NOT do:**
- Business logic
- Validation beyond "does this query work?"

**Why no unit tests:** You cannot meaningfully test SQL without a real database. Unit testing repositories with mocks just tests that you called the mock correctly — useless. Instead, we test repositories through integration tests with a real Postgres instance.

```go
// Example repository — just SQL, no decisions
func (r *VisitRepo) Create(ctx context.Context, visit *Visit) (*Visit, error) {
    query := `
        INSERT INTO visits (id, patient_id, doctor_id, visit_number, status, created_at)
        VALUES ($1, $2, $3, $4, $5, $6)
        RETURNING id, created_at
    `
    err := r.db.QueryRowContext(ctx, query,
        visit.ID, visit.PatientID, visit.DoctorID, 
        visit.VisitNumber, visit.Status, visit.CreatedAt,
    ).Scan(&visit.ID, &visit.CreatedAt)
    
    return visit, err
}
```

#### Domain Layer (50% coverage target)
**What it does:**
- Provides shared utilities used by multiple services
- Caching helpers, validation functions, common types
- Infrastructure code, not business code

**What it does NOT do:**
- Contain medical/clinic business rules (those go in Service)

**Why medium coverage target:** Important to test, but less critical than business logic. Focus tests on tricky utility functions.

```go
// Example domain utility — shared validation
func ValidateNIK(nik string) error {
    if len(nik) != 16 {
        return ErrNIKInvalidLength
    }
    for _, c := range nik {
        if c < '0' || c > '9' {
            return ErrNIKNotNumeric
        }
    }
    return nil
}
```

---

## 2.4 Testing Folder Hierarchy

Now that you understand the layers, here's where tests live.

### The Golden Rule: Unit Tests Live Next to the Code

In Go, test files live **in the same directory** as the code they test. This is not optional — it's how Go works.

```
/internal/emr/service/
├── visit_service.go           # The code
└── visit_service_test.go      # The test (same folder!)
```

**Why?** 
- Tests are easy to find
- When you change code, the test is right there
- Go tooling expects this pattern

### Full Testing Folder Structure

```
/wellmed
│
├── /internal                       # Service code + unit tests together
│   └── /emr
│       ├── /handler
│       │   ├── visit_handler.go
│       │   └── visit_handler_test.go       # Unit test
│       ├── /service
│       │   ├── visit_service.go
│       │   └── visit_service_test.go       # Unit test (most important!)
│       └── /repository
│           └── visit_repo.go               # No unit test — tested via integration
│
└── /tests                          # Cross-service and special tests
    │
    ├── /integration               # Tests that need real infrastructure
    │   ├── /emr
    │   │   ├── visit_flow_test.go         # Full visit lifecycle
    │   │   └── encounter_sync_test.go     # EMR → Queue → SATU SEHAT
    │   ├── /cashier
    │   │   └── billing_flow_test.go       # Service → Invoice → Payment
    │   └── /cross_service
    │       └── emr_cashier_test.go        # EMR + Cashier together
    │
    ├── /contracts                 # External API contract tests
    │   ├── /satu_sehat
    │   │   ├── patient_contract_test.go
    │   │   └── mocks/
    │   │       └── satu_sehat_mock.go
    │   └── /pcare
    │       └── ...
    │
    ├── /permissions               # RBAC matrix tests
    │   └── matrix_test.go
    │
    ├── /sagas                     # Multi-service transaction tests
    │   ├── create_visit_saga_test.go
    │   └── compensation_test.go
    │
    └── /performance               # Benchmarks and load tests
        ├── /benchmarks
        │   └── visit_service_bench_test.go
        └── /k6
            └── visit_load_test.js
```

### When to Use /internal vs /tests

| Location | Use When | Examples |
|----------|----------|----------|
| `/internal/[service]/*_test.go` | Testing ONE function in isolation | `TestCalculateFee`, `TestValidateNIK` |
| `/tests/integration/` | Testing multiple components together | Visit creation that hits DB and queue |
| `/tests/contracts/` | Verifying external API behavior | SATU SEHAT response format |
| `/tests/permissions/` | Testing who can do what | Doctor can view patient, receptionist cannot delete |
| `/tests/sagas/` | Testing multi-step transactions | Create visit → Reserve slot → Bill → (rollback if fail) |

### Running Tests by Category

```bash
# Run all unit tests (fast, <4 minutes)
go test ./...

# Run only EMR service unit tests
go test ./internal/emr/...

# Run integration tests (needs Docker services running)
go test -tags=integration ./tests/integration/...

# Run contract tests
go test -tags=contract ./tests/contracts/...

# Run benchmarks
go test -bench=. ./tests/performance/benchmarks/...
```

---

## 2.5 Coverage Targets & Measurement

### Our Commitments

| Layer | Minimum Coverage | Why This Number |
|-------|------------------|-----------------|
| **Handler** | 30% | Thin layer, mostly plumbing. Bugs caught by integration tests. |
| **Service** | 80% | Business logic lives here. Bugs hurt. Test thoroughly. |
| **Domain** | 50% | Utility code. Important but less critical than business logic. |
| **Repository** | Via integration | Cannot meaningfully unit test SQL. Use real database. |

### How We Measure

Coverage is checked automatically in CI:

```bash
# Generate coverage report
go test -coverprofile=coverage.out ./...

# View coverage by function
go tool cover -func=coverage.out

# View coverage in browser (with highlighted code)
go tool cover -html=coverage.out
```

### Enforcement Policy

CI will **warn** (not fail) if coverage drops below thresholds. We trust developers to make judgment calls.

However, if you submit a PR that significantly drops coverage, expect questions:
- "Why isn't this new function tested?"
- "Can you add a test for the edge case on line 47?"

The goal is not 100% coverage — it's **meaningful** coverage of code that matters.

---

# Part 3: Testing Types & Patterns

This section goes deeper into specific testing approaches. Read Part 2 first to understand the foundations.

## 3.1 Testing Taxonomy

Tests are categorized into distinct types. Understanding these helps you know which test to write for which situation.

### Category A: Unit Tests
Tests that verify a single function or method in isolation, with dependencies mocked.

- **Location**: Same directory as source (`*_test.go`)
- **Run**: Every PR, must complete in <4 minutes total
- **Dependencies**: None (all mocked)

### Category B: Integration Tests (Internal)
Tests that verify multiple **internal** WellMed components work together:
- Service-to-service gRPC calls
- Service-to-database operations
- Service-to-queue (RabbitMQ) operations
- Service-to-cache (Redis) operations

- **Location**: `/tests/integration/`
- **Run**: On merge to staging, must complete in <6 minutes total
- **Dependencies**: Real Postgres, Redis, RabbitMQ (via Docker)

### Category C: Contract Tests (External)
Tests that verify our code correctly handles **external** API contracts:
- SATU SEHAT API
- P-Care/BPJS API
- Mobile JKN API
- Third-party services (Jurnal, Xendit, etc.)

Includes:
- Mock server testing (in CI)
- Daily provider contract verification (cron job)
- Response format validation

- **Location**: `/tests/contracts/`
- **Run**: Mock tests in CI; real verification daily

### Category D: End-to-End Tests
Tests that simulate complete user workflows through the full stack. 

- **Current**: Manual QA with checklist
- **Future**: Playwright automation for critical paths
- **Run**: Before major releases

### Category E: Performance Tests
- **Benchmarks**: Repeatable performance measurements (`go test -bench`)
- **Load tests**: Behavior under concurrent users (k6)
- **Slow query detection**: Postgres monitoring

- **Target**: No operation >1000ms
- **Run**: Before major releases, after architecture changes

### Category F: Permission Tests
- **Scope**: RBAC enforcement across all resources
- **Pattern**: Matrix-based (role × resource × action)
- **Location**: `/tests/permissions/`
- **Run**: Every PR (part of unit test suite)

### Category G: SAGA Tests
- **Scope**: Multi-service transaction flows with compensation
- **Verifies**: Execute + Compensate for all reversible operations
- **Location**: `/tests/sagas/`
- **Run**: Part of integration tests

### Taxonomy Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         WELLMED TESTING TAXONOMY                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  UNIT TESTS                                                    [A]     │ │
│  │  Scope: Single function/method, dependencies mocked                    │ │
│  │  Location: Same directory as source (*_test.go)                        │ │
│  │  Run: Every PR, <4 minutes total                                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  INTEGRATION TESTS (Internal)                                  [B]     │ │
│  │  Scope: Internal WellMed service interactions                          │ │
│  │  • Service ↔ Service (gRPC)                                           │ │
│  │  • Service ↔ Database (Postgres)                                      │ │
│  │  • Service ↔ Queue (RabbitMQ)                                         │ │
│  │  • Service ↔ Cache (Redis)                                            │ │
│  │  Location: /tests/integration/                                         │ │
│  │  Run: On merge to staging, <6 minutes total                           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  CONTRACT TESTS (External)                                     [C]     │ │
│  │  Scope: External API contract verification                             │ │
│  │  • Mock server tests (in CI)                                          │ │
│  │  • Provider verification (daily cron)                                 │ │
│  │  APIs: SATU SEHAT, P-Care, Mobile JKN, Jurnal, Xendit, etc.          │ │
│  │  Location: /tests/contracts/                                           │ │
│  │  Run: Mock tests in CI; verification daily                            │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  END-TO-END TESTS                                              [D]     │ │
│  │  Scope: Full user workflows through UI                                 │ │
│  │  Current: Manual QA with checklist                                     │ │
│  │  Future: Playwright automation for critical paths                      │ │
│  │  Run: Before major releases                                            │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  PERFORMANCE TESTS                                             [E]     │ │
│  │  • Benchmarks: go test -bench                                         │ │
│  │  • Load tests: k6 (note: requires production-like data)               │ │
│  │  • Query monitoring: Postgres slow query log                          │ │
│  │  Target: No operation >1000ms                                          │ │
│  │  Run: Before major releases, after architecture changes                │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  PERMISSION TESTS                                              [F]     │ │
│  │  Scope: RBAC enforcement across all resources                          │ │
│  │  Pattern: Matrix-based (role × resource × action)                     │ │
│  │  Location: /tests/permissions/                                         │ │
│  │  Run: Every PR (part of unit test suite)                              │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  SAGA TESTS                                                    [G]     │ │
│  │  Scope: Multi-service transaction flows with compensation              │ │
│  │  Verifies: Execute + Compensate for all reversible operations         │ │
│  │  Location: /tests/sagas/                                               │ │
│  │  Run: Part of integration tests                                        │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Caching Strategy

Use Redis for caching shared data. All caches use a **push-based pattern** (pub/sub) with a safety duration that auto-resets if no update is received.

### Why Push + Safety Duration?

- **Push (pub/sub)**: When data changes, the owning service broadcasts the update. All services receive the new value immediately.
- **Safety duration**: If a push is missed (network issue, service restart), the cache auto-expires and forces a database refresh. This is a fallback, not the primary mechanism.

| Data Type | Method | Safety Duration | Reasoning |
|-----------|--------|-----------------|-----------|
| Patient demographics | Push (pub/sub) | 1 month | Rarely changes |
| Service catalog | Push (pub/sub) | 1 month | Changes weekly at most |
| Package definitions | Push (pub/sub) | 1 month | Changes monthly at most |
| Prices | Push (pub/sub) | 1 month | Event-driven updates ensure freshness |
| Visit status | Push (pub/sub) | 15 minutes | Must stay fresh; short safety window for quick recovery if push fails |
| Queue position | **Never cache** | - | Real-time critical; always query live |

### Cache Update Pattern
```go
// When data changes: write to DB, then broadcast new value
func (s *PatientService) UpdatePatient(ctx context.Context, patient *Patient) error {
    // 1. Write to database
    if err := s.repo.Update(ctx, patient); err != nil {
        return err
    }
    
    // 2. Broadcast new value to all services (push)
    s.redis.Publish(ctx, "patient:updated", patient.ID)
    
    // 3. Update local cache with safety TTL
    s.cache.SetWithTTL("patient:"+patient.ID, patient, 30*24*time.Hour) // 1 month safety
    
    return nil
}

// All services listen for updates
func (c *CacheManager) ListenForPatientUpdates(ctx context.Context) {
    pubsub := c.redis.Subscribe(ctx, "patient:updated")
    for msg := range pubsub.Channel() {
        // Invalidate local cache — next read will fetch fresh from DB
        c.cache.Delete("patient:" + msg.Payload)
    }
}

### Cache Invalidation Pattern

```go
// When price changes
func (s *CatalogService) UpdatePrice(ctx context.Context, serviceID string, newPrice Money) error {
    // Update database
    if err := s.repo.UpdatePrice(ctx, serviceID, newPrice); err != nil {
        return err
    }
    
    // Broadcast invalidation
    s.redis.Publish(ctx, "cache:invalidate:prices", serviceID)
    
    return nil
}

// All services listen for invalidation
func (c *CacheManager) ListenForInvalidation(ctx context.Context) {
    pubsub := c.redis.Subscribe(ctx, "cache:invalidate:prices")
    for msg := range pubsub.Channel() {
        c.cache.Delete("price:" + msg.Payload)
    }
}
```

---

## 3.3 State Machine Pattern

For entities that move through states (visits, orders, etc.), use a formal state machine with audit trail.

### State Definition

```go
// visit/states.go
const (
    VisitCreated         = "created"
    VisitInProgress      = "in_progress"
    VisitPendingPayment  = "pending_payment"  // MCU: services done, awaiting payment
    VisitPaid            = "paid"
    VisitComplete        = "complete"
    VisitCancelled       = "cancelled"
)

// Valid transitions (configurable per deployment)
var ValidTransitions = map[string][]string{
    VisitCreated:        {VisitInProgress, VisitCancelled},
    VisitInProgress:     {VisitPendingPayment, VisitPaid, VisitCancelled},
    VisitPendingPayment: {VisitPaid, VisitCancelled},
    VisitPaid:           {VisitComplete},
    VisitComplete:       {}, // Terminal state
    VisitCancelled:      {}, // Terminal state
}
```

### Audit Trail

```go
type StateTransition struct {
    ID          uuid.UUID
    EntityType  string    // "visit", "order", etc.
    EntityID    uuid.UUID
    FromState   string
    ToState     string
    TriggeredBy uuid.UUID // User or service that triggered
    ServiceName string    // Which service made the change
    Timestamp   time.Time
    Metadata    map[string]interface{}
}
```

### Configurable Order

The transition map can be reconfigured per deployment to support different workflows:
- **Standard flow**: Service → Payment → Complete
- **MCU flow**: All services → Payment → Complete
- **Postpaid flow**: Service → Complete → Payment (deferred)

---

## 3.4 Proto-Based Service Contracts

Use Protocol Buffers (`.proto` files) as the single source of truth for service contracts.

### Benefits

1. **Type safety**: Compile-time verification of message formats
2. **Documentation**: Generate API docs from proto files
3. **Version management**: Breaking changes are immediately visible
4. **Cross-service validation**: Proto compiler catches incompatibilities

### Directory Structure

```
/proto
├── common/
│   └── types.proto       # Shared types (Money, DateTime, etc.)
├── emr/
│   └── v1/
│       └── emr.proto     # EMR service contract
├── billing/
│   └── v1/
│       └── billing.proto
└── ...
```

### Documentation Generation

```bash
# Generate markdown docs from proto files
protoc --doc_out=./docs/api --doc_opt=markdown,services.md ./proto/**/*.proto
```

---

## 3.5 Provider Contract Verification

Run daily verification against real external APIs to detect changes before they break production.

### Architecture

```
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│  Contract Checker   │────▶│   External APIs     │────▶│   Alert on Change   │
│  (Python/Cron)      │     │   (SATU SEHAT,etc.) │     │   (Slack/Email)     │
│                     │     │                     │     │                     │
│  Runs: Daily 6 AM   │     │  Endpoints checked: │     │  Records changes    │
│  Host: webhost      │     │  - Response format  │     │  in changelog       │
│                     │     │  - Status codes     │     │                     │
└─────────────────────┘     │  - Required fields  │     └─────────────────────┘
                            └─────────────────────┘
```

### Verification Scope

| API | Endpoints Verified | Check Frequency |
|-----|-------------------|-----------------|
| SATU SEHAT | Patient, Encounter, Observation | Daily |
| P-Care | Pendaftaran, Kunjungan, Rujukan | Daily |
| Mobile JKN | TBD (Q3 roadmap) | - |
| Jurnal | Contacts, Invoices, Payments | Daily |
| Xendit | Payment, Invoice, Callback | Daily |

### Changelog

Store verification results with timestamps to track API evolution:

```json
{
  "timestamp": "2025-01-19T06:00:00Z",
  "api": "satu-sehat",
  "endpoint": "/fhir-r4/v1/Patient",
  "status": "changed",
  "changes": [
    "New field 'telecom.rank' added",
    "Response time increased 200ms → 450ms"
  ]
}
```

---

## 3.6 Frontend Monitoring (Future Phase)

> **Status**: Next phase focus. Current priority is backend testing foundation.

This section covers tools for catching frontend errors both pre-production and in live use. Both tools have free tiers suitable for our scale.

### Playwright (Pre-Production)

Automated browser testing — simulates real user clicking through the app.

| | |
|---|---|
| **What it catches** | Broken buttons, failed page loads, UI regressions |
| **When it runs** | CI/CD before deployment |
| **Free tier** | Open source, completely free |
| **Setup estimate** | 1-2 days for critical path coverage |

**Use case**: Automate our current manual E2E checklist. Instead of QA manually clicking "create visit → add service → payment", Playwright does it automatically on every PR.

### Sentry (Production)

Error monitoring — catches JavaScript crashes when real users hit them.

| | |
|---|---|
| **What it catches** | Runtime errors, crashes, failed API calls |
| **When it runs** | Live in production, 24/7 |
| **Free tier** | 5K errors/month, 1 user (sufficient for early stage) |
| **Setup estimate** | ~2 hours |

**Use case**: User hits a bug on the payment page → Sentry captures the error with stack trace, browser info, user context → We fix it before more users are affected.

### How They Work Together
```
Development → CI/CD → Staging → Production
                ↑                    ↑
            Playwright            Sentry
          (catches before       (catches what
           deployment)          slips through)
```

### Smoke Tests (Implement Now)

While Playwright/Sentry are future phase, basic smoke tests are quick to implement:
```bash
#!/bin/bash
# smoke_test.sh - run after each staging deployment

BASE_URL="https://staging.wellmed.id"
FAILED=0

check_endpoint() {
    local name=$1
    local url=$2
    local expected=$3
    
    status=$(curl -s -o /dev/null -w "%{http_code}" "$url")
    if [ "$status" != "$expected" ]; then
        echo "FAIL: $name returned $status (expected $expected)"
        FAILED=1
    else
        echo "PASS: $name"
    fi
}

check_endpoint "Health" "$BASE_URL/health" "200"
check_endpoint "Login page" "$BASE_URL/login" "200"
check_endpoint "API status" "$BASE_URL/api/v1/status" "200"

if [ $FAILED -eq 1 ]; then
    # Alert team
    exit 1
fi

echo "All smoke tests passed"
```

### Triage Workflow (Future)

Once both tools are in place:

1. **Playwright fails in CI** → Bug caught before merge → Developer fixes
2. **Sentry alert in production** → Bug slipped through → Create ticket, prioritize based on frequency/impact
3. **Weekly review** → Look at Sentry trends → Inform what Playwright tests to add

**Future enhancement**: Auto-restart on crash via systemd with health checks.

---

## 3.7 Permission Testing

Test RBAC enforcement with a matrix-based approach.

### Permission Matrix (Example)

```yaml
# permissions.yaml
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
  
  lab_result:
    create: [admin, lab_tech]
    read: [admin, doctor, nurse, lab_tech]
    update: [admin, lab_tech]
    delete: [admin]
```

### Test Implementation

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

---

## 3.8 SAGA Pattern Implementation

### Core Concept
A SAGA is a sequence of local transactions where each step has a compensating action. If any step fails, previous steps are rolled back via their compensations.

**We use the Orchestration pattern**: A central orchestrator explicitly coordinates all steps and their compensations, rather than services reacting to events (choreography).

### Pattern Structure
```go
// saga/saga.go
type Step interface {
    Name() string
    Execute(ctx context.Context) error
    Compensate(ctx context.Context) error
}

// Orchestrator - central coordinator for the SAGA
type Saga struct {
    name  string
    steps []Step
}

// Execute orchestrates all steps from a single point
func (s *Saga) Execute(ctx context.Context) error {
    var completed []Step
    
    // Orchestrator explicitly calls each step in sequence
    for _, step := range s.steps {
        if err := step.Execute(ctx); err != nil {
            // Orchestrator manages compensation in reverse order
            for i := len(completed) - 1; i >= 0; i-- {
                if compErr := completed[i].Compensate(ctx); compErr != nil {
                    // Log but continue compensation
                    log.Error("Compensation failed", 
                        "saga", s.name,
                        "step", completed[i].Name(),
                        "error", compErr)
                }
            }
            return fmt.Errorf("saga %s failed at step %s: %w", 
                s.name, step.Name(), err)
        }
        completed = append(completed, step)
    }
    
    return nil
}
```

### Orchestration vs Choreography
**Why Orchestration:**
- ✅ Centralized control and visibility
- ✅ Easier to understand and debug
- ✅ Simpler error handling and compensation logic
- ✅ Clear transaction boundaries

**Not using Choreography because:**
- ❌ Would require event bus infrastructure
- ❌ Harder to track saga state across services
- ❌ More complex debugging of distributed flows
- ❌ Overkill for our current scale

### Example: Create MCU Visit
```go
// Orchestrator coordinates the entire MCU visit creation flow
func NewCreateMCUVisitSaga(req CreateMCUVisitRequest) *Saga {
    return &Saga{
        name: "create-mcu-visit",
        steps: []Step{
            &CreateVisitStep{PatientID: req.PatientID},
            &ReserveLabSlotsStep{Services: req.LabServices},
            &ReserveRadiologySlotsStep{Services: req.RadiologyServices},
            &CreateInvoiceStep{},
            &NotifyDepartmentsStep{},
        },
    }
}

// Each step implements Execute and Compensate
type CreateVisitStep struct {
    PatientID string
    visitID   string // Set during Execute, used in Compensate
}

func (s *CreateVisitStep) Execute(ctx context.Context) error {
    visit, err := emrService.CreateVisit(ctx, s.PatientID)
    if err != nil {
        return err
    }
    s.visitID = visit.ID
    return nil
}

func (s *CreateVisitStep) Compensate(ctx context.Context) error {
    return emrService.DeleteVisit(ctx, s.visitID)
}
```

### SAGA Testing
```go
func TestCreateMCUVisitSaga_FailureAtStep3_CompensatesSteps1And2(t *testing.T) {
    // Setup: Make radiology reservation fail
    radiologyService.ForceError(ErrNoAvailableSlots)
    
    saga := NewCreateMCUVisitSaga(testRequest)
    err := saga.Execute(ctx)
    
    // Verify saga failed
    assert.Error(t, err)
    assert.Contains(t, err.Error(), "ReserveRadiologySlots")
    
    // Verify orchestrator compensated previous steps
    visit, err := emrService.GetVisit(ctx, expectedVisitID)
    assert.Error(t, err) // Visit should be deleted
    
    labSlots := labService.GetReservedSlots(ctx, testRequest.PatientID)
    assert.Empty(t, labSlots) // Lab slots should be released
}
```

---

## 3.9 Queue Architecture

### Queue Registry

```yaml
# /config/queues.yaml
exchanges:
  integrations:
    type: topic
    durable: true
  internal:
    type: topic
    durable: true

queues:
  satu_sehat_sync:
    exchange: integrations
    routing_key: sync.satu_sehat.#
    priority: normal
    retry:
      max_attempts: 5
      initial_delay: 2s
      max_delay: 5m
      multiplier: 2
    dlq: satu_sehat_sync_dlq
    
  pcare_sync:
    exchange: integrations
    routing_key: sync.pcare.#
    priority: high
    retry:
      max_attempts: 10
      initial_delay: 500ms
      max_delay: 10m
      multiplier: 2
    dlq: pcare_sync_dlq
    
  visit_state_change:
    exchange: internal
    routing_key: visit.state.#
    priority: high
    retry:
      max_attempts: 3
      initial_delay: 100ms
      max_delay: 1s
    # No DLQ - must succeed or alert

dead_letter_queues:
  satu_sehat_sync_dlq:
    log_file: /var/log/wellmed/dlq/satu_sehat.log
  pcare_sync_dlq:
    log_file: /var/log/wellmed/dlq/pcare.log
```

### DLQ Logging

Each dead letter queue writes to a separate log file for easy debugging:

```
/var/log/wellmed/dlq/
├── satu_sehat.log
├── pcare.log
└── xendit.log
```

---

## 3.10 Version Management

### Service Versioning

All services use semantic versioning: `MAJOR.MINOR.PATCH`

- **MAJOR**: Breaking changes to API
- **MINOR**: New features, backward compatible
- **PATCH**: Bug fixes, backward compatible

### Compatibility Matrix

```yaml
# /config/compatibility.yaml
services:
  emr-service:
    version: 1.5.0
    requires:
      patient-service: ">=1.2.0"
      billing-service: ">=1.0.0"
      
  patient-service:
    version: 1.3.0
    requires: {}
    
  billing-service:
    version: 1.1.0
    requires:
      emr-service: ">=1.4.0"
```

### CI Compatibility Check

Before deployment, CI validates that the new version is compatible with running services:

```go
func TestServiceCompatibility(t *testing.T) {
    newVersion := GetVersionToDeploy()
    runningVersions := GetRunningServiceVersions()
    
    for service, required := range newVersion.Requires {
        running := runningVersions[service]
        assert.True(t, SatisfiesVersion(running, required),
            "Incompatible: %s %s requires %s %s",
            newVersion.Name, newVersion.Version, service, required)
    }
}
```


---

## 3.11 AWS Backup Strategy

| Environment | Asset | Frequency | Retention |
|-------------|-------|-----------|-----------|
| **Production** | DB WAL (transaction log) | Continuous | 7 days |
| **Production** | DB full snapshot | Daily | 14 days |
| **Production** | Full snapshot of app and frontend | Weekly or as triggered immediately priort to upgrade (via github action/IAM rights) | last 4 copies |
| **Staging** | DB snapshot | Weekly | 2 weeks |
| **Staging** | Config/structure | Git (IaC) | Forever |
| **Dev** | None | - | - |

### Disaster Recovery Documentation

Location: `/docs/runbooks/disaster-recovery.md`

Contents:
- RTO target: 4 hours
- RPO target: 1 hour
- Step-by-step recovery procedures
- Contact information
- Quarterly test schedule

---

# Part 4: CI/CD Configuration

## 4.1 Local Development Requirements (Pre-Push Gate)

Before pushing code to any branch, developers must pass these local checks. This is the first quality gate and prevents broken code from entering the CI pipeline.

### Required Checks

```bash
# Makefile targets every developer must run before pushing
make lint       # golangci-lint - must pass with no errors
make test       # Unit tests - must pass
make build      # Service must compile
```

### Recommended Workflow

```bash
# Before pushing any code:
$ make lint && make test && make build

# If all pass, safe to push
$ git push origin feature/my-branch
```

### Makefile Template

```makefile
# Makefile (in each service directory)
.PHONY: lint test build all

# Run all checks (use before pushing)
all: lint test build

lint:
	golangci-lint run --timeout=3m ./...

test:
	go test -v -race -timeout=3m ./...

build:
	go build -o bin/service ./cmd/...

# Optional: run before committing
pre-commit: lint test
```

### IDE Integration (Recommended)

Configure your IDE to run `go fmt` and `go vet` on save. This catches issues before they become commits.

---

## 4.2 Branch & Merge Rules

| Level | Scope | Who Can Merge | Requirements |
|-------|-------|---------------|--------------|
| 🟢 **Green** | Minor changes, bug fixes | All developers | CI passes (lint + unit + integration tests compile) |
| 🟡 **Yellow** | Database migrations, schema changes | Senior devs (Fajri, Hamzah) | Green + migration review |
| 🔴 **Red** | New modules, major features, prod deploy | CTO only | Yellow + architecture review |

**Flow**: `feature branch` → `dev` → `staging` → `production`

- Merging to `dev`: Green level (after CI passes)
- Merging `staging` → `production`: Always Red (CTO approval required)

---

## 4.3 GitHub Actions Workflow

### PR Review with Claude

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
      
      - name: Generate Diff
        run: git diff origin/${{ github.base_ref }}..HEAD > diff.txt
      
      - name: Generate Review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          DIFF_CONTENT=$(cat diff.txt | head -c 50000)  # Limit size
          
          curl -s https://api.anthropic.com/v1/messages \
            -H "Content-Type: application/json" \
            -H "x-api-key: $ANTHROPIC_API_KEY" \
            -H "anthropic-version: 2023-06-01" \
            -d @- <<EOF | jq -r '.content[0].text' > review.md
          {
            "model": "claude-sonnet-4-20250514",
            "max_tokens": 2048,
            "messages": [{
              "role": "user",
              "content": "Review this code diff for a healthcare clinic management system (WellMed). Provide:\n\n1. **Ringkasan Perubahan** (2-3 kalimat dalam Bahasa Indonesia)\n2. **Risk Level**: Low/Medium/High dengan penjelasan\n3. **Potential Issues**: Bug atau masalah yang mungkin\n4. **Recommendations**: Saran untuk reviewer\n5. **ADR Needed?**: Apakah perubahan ini memerlukan Architecture Decision Record?\n\nDiff:\n\n$DIFF_CONTENT"
            }]
          }
          EOF
      
      - name: Post Review Comment
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

### Full CI Pipeline (GitHub Actions)

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
    timeout-minutes: 8  # Enforce 8-minute limit
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
          # Extract total coverage and warn if below thresholds
          # (advisory, not blocking)
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.out

  integration-test:
    runs-on: ubuntu-latest
    timeout-minutes: 12  # Enforce 12-minute limit
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

  compatibility-check:
    runs-on: ubuntu-latest
    needs: unit-test
    steps:
      - uses: actions/checkout@v4
      - name: Check version compatibility
        run: |
          # Script to verify compatibility matrix
          ./scripts/check-compatibility.sh
```

---

# Part 5: Monitoring Dashboard
// this is going to sit in the HQ-PUSAT (Wellmed) - and needs to be visible to public web after same sign in flow as HQ
## 5.1 Dashboard Goals

Single view showing:
- VM/instance health (CPU, memory, disk)
- Service status (up/down, response times)
- Deployment history (what's deployed where)
- System alerts (errors, DLQ depth)
- Backup status (last successful backup)

## 5.2 Data Sources

| Source | Data | Collection Method |
|--------|------|-------------------|
| AWS CloudWatch | CPU, memory, disk, network, logs | CloudWatch API |
| AWS X-Ray | Request traces across services | X-Ray API |
| AWS Secrets Manager | Credential rotation status | Secrets Manager API |
| Service health endpoints | `/health` response | Poll every 60s |
| Postgres | Connections, slow queries | `pg_stat_*` |
| RabbitMQ | Queue depths, message rates | Management API |
| ElasticSearch | Cluster health | Cluster health API |
| GitHub Actions | Build status, deployments | GitHub API |
| DLQ tables | Failed message count | Direct query |

## 5.3 System Event Log

Display format:
```
2025-01-19 04:00 [BACKUP]  prod-db daily snapshot completed (snap-abc123)
2025-01-19 03:45 [DEPLOY]  emr-service v1.5.2 → prod (ip-10-0-1-50)
2025-01-18 22:30 [DEPLOY]  emr-service v1.5.2 → staging
2025-01-18 16:00 [ALERT]   pcare-sync-worker: 5 failed syncs in last hour
2025-01-18 15:30 [BACKUP]  prod volumes weekly snapshot completed
2025-01-18 14:00 [CONTRACT] satu-sehat: New field detected in Patient response
```

## 5.4 Evolution Path

1. **Phase 1 (Now)**: Simple `/status` page showing service health
2. **Phase 2**: Deploy dashboard with read-only access for team
3. **Phase 3**: Integrate with Claude for automatic bug logging (Ralph loops)

---

# Part 6: Week 1 Execution Checklist

## Day 1

- [X] Team sync: Review this document, assign owners
- [ ] Set up GitHub Actions basic workflow (lint + unit tests) //alex
- [ ] Create first unit test examples for team reference //alex
- [ ] Enable Postgres slow query logging  //hamzah
- [ ] Draft RabbitMQ queue registry document // alex attempt via claude agent to create/update 

## Day 2

- [ ] Start API client package implementation (see companion doc)
- [ ] Set up dead letter queue in RabbitMQ // hamzah
- [ ] Create ADR template and first 2 ADRs // again will try to create via claude agent
- [ ] Set up Claude PR review workflow // alex

## Day 3

- [ ] Complete API client package with tests
- [ ] Implement SATU SEHAT client using API client package
- [ ] Create in-process mock for SATU SEHAT


---

# Appendix A: Library Reference

## Go Libraries

| Library | Purpose | Link |
|---------|---------|------|
| `stretchr/testify` | Assertions, mocks | github.com/stretchr/testify |
| `avast/retry-go` | Retry with backoff | github.com/avast/retry-go |
| `google/uuid` | UUID generation | github.com/google/uuid |
| `streadway/amqp` | RabbitMQ client | github.com/streadway/amqp |
| `go-redis/redis` | Redis client | github.com/go-redis/redis |
| `go-playground/validator` | Struct validation | github.com/go-playground/validator |
| `uber-go/zap` | Structured JSON logging | go.uber.org/zap |

## Development Tools

| Tool | Purpose |
|------|---------|
| `golangci-lint` | Comprehensive linter |
| `go vet` | Static analysis |
| `gofmt` | Code formatting |
| `gosec` | Security scanning |
| `k6` | Load testing |
| `protoc` | Protocol buffer compiler |

---

*Document Owner: Alex (CMO)*  
*Version: 5.0*  
*Last Updated: January 2025*  
*Status: Team decisions incorporated - ready for implementation*
