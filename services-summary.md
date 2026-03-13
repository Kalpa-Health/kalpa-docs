# WellMed Microservices Summary

**Version:** 1.0
**Date:** 2026-03-13
**Purpose:** Overview semua microservices dalam ekosistem WellMed

---

## 1. Architecture Overview

WellMed menggunakan arsitektur microservices dengan Go sebagai bahasa utama:

```
                    ┌─────────────────┐
                    │  wellmed-gateway│ :8080 (HTTP)
                    └────────┬────────┘
                             │ gRPC
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│wellmed-backbone│  │wellmed-consult │  │wellmed-cashier │
│    :50051      │  │    :50052      │  │    :50053      │
│ Saga Orchestr. │  │ Saga Particip. │  │ Saga Particip. │
└────────────────┘  └────────────────┘  └────────────────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │ RabbitMQ
                    ┌────────▼────────┐
                    │  Saga Events    │
                    │ wellmed.saga    │
                    └─────────────────┘
```

---

## 2. Service Summary

### 2.1 wellmed-backbone (Core Service)

| Item | Value |
|------|-------|
| **Port** | `:50051` (gRPC) |
| **Role** | Saga orchestrator, canonical data owner, auth |
| **Module** | `github.com/kalpa-health/wellmed-backbone` |
| **Path** | `/var/www/projects/kalpa/wellmed-backbone` |

**Key Responsibilities:**
- Canonical domain ownership: patient, user, employee, item catalog
- Sole saga orchestrator (ADR-005)
- JWT authentication & session management
- Multi-tenant database routing
- Reference data management (unicode, autolist)

**Domain Modules:** 45+ modules
- Core: patient, visit_patient, visit_examination, user, employee
- Reference: medicine, item, itemcatalog, treatment, unicode
- Saga: patient/saga, pharmacy_sale/saga

**Entities:** 45+ entities
- Patient, People, Employee, User
- Item, Stock, Medicine
- VisitPatient, VisitRegistration, VisitExamination
- Saga (state machine)

**Seeders:** 33 Go seeder files
- Reference data: anatomy, dosage_form, profession, etc.
- Hierarchical: therapeutic_class, medic_service, patient_occupation

---

### 2.2 wellmed-consultation (Clinical EMR Service)

| Item | Value |
|------|-------|
| **Port** | `:50052` (gRPC) |
| **Role** | Saga participant, visit lifecycle owner |
| **Path** | `/var/www/projects/kalpa/wellmed-consultation` |

**Key Responsibilities:**
- Visit lifecycle management (registration → examination → sign-off)
- Clinical assessment & diagnosis
- Prescription management
- Referral handling
- Saga participant (receives triggers from Backbone)

**Domain Modules:** 15 modules
- visit_patient, visit_registration, visit_examination
- assessment, referral, treatment, frontline
- prescription, medication_request

**Entities:** 60+ entities (mostly in `/schema/entity/emr/`)
- VisitPatient, VisitRegistration, VisitExamination
- Assessment (polymorphic via Morph field)
- Referral, Prescription, MedicationRequest
- Diagnosis, ExaminationTreatment

**gRPC Services:** 8 services
- AssessmentService, FrontlineService, ReferralService
- TreatmentService, VisitExaminationService
- VisitPatientService, VisitRegistrationService

---

### 2.3 wellmed-cashier (POS/Financial Service)

| Item | Value |
|------|-------|
| **Port** | `:50053` (gRPC) |
| **Role** | Saga participant, financial transactions |
| **Path** | `/var/www/projects/kalpa/wellmed-cashier` |

**Key Responsibilities:**
- Transaction processing (create, list, cancel)
- Billing management
- Invoice generation
- Payment summary tracking
- Saga participant for POS operations

**Domain Modules:** 4 modules
- transaction, billing, invoice, payment_summary

**Entities:** 14 entities
- Transaction, TransactionItem, TransactionHasConsument
- Billing, Invoice
- PaymentSummary, PaymentDetail
- Consument, Refund, SplitPayment
- WalletTransaction, Activity, ActivityStatus

**gRPC Services:** 4 services
- TransactionService (Index, Show, Store, Delete)
- BillingService (Index, Show)
- InvoiceService (Index, Show)
- PaymentSummaryService (Store)

---

### 2.4 wellmed-infrastructure (Shared Infrastructure)

| Item | Value |
|------|-------|
| **Type** | Infrastructure repository (NOT a service) |
| **Path** | `/var/www/projects/kalpa/wellmed-infrastructure` |

**Contents:**
- Go SDK: X-Ray tracing, CloudWatch logging, RabbitMQ tracing
- CI/CD Templates: GitHub Actions workflows (ci.yml, pr-review.yml)
- AWS IAM: Policies, roles, instance profiles
- Scripts: bootstrap-repo.sh, agent installation

---

### 2.5 kalpa-docs (Documentation Hub)

| Item | Value |
|------|-------|
| **Type** | Documentation repository |
| **Path** | `/var/www/projects/kalpa/kalpa-docs` |

**Contents:**
- ADRs: 8 architecture decision records
- Services docs: backbone.md, consultation.md, gateway.md
- Plans: Project plans with PLAN/PRD/PROGRESS structure
- Development guides: setup, conventions, testing
- Operations: backup, migrations, incident response

---

## 3. Entity Cross-Reference

### 3.1 Shared Entities (Backbone Canonical)

Entities owned by Backbone, referenced by other services:

| Entity | Backbone | Consultation | Cashier | Notes |
|--------|----------|--------------|---------|-------|
| Patient | ✅ Owner | Reference | Reference | ULID, includes People FK |
| People | ✅ Owner | Reference | — | Master person record |
| Employee | ✅ Owner | Reference | Reference | Staff/practitioner |
| User | ✅ Owner | — | — | Auth credentials |
| Item | ✅ Owner | — | Reference | Inventory items |
| Medicine | ✅ Owner | — | — | Drug catalog |
| MedicService | ✅ Owner | Reference | — | Department/specialty |

### 3.2 Domain-Specific Entities

| Domain | Service | Key Entities |
|--------|---------|--------------|
| Visit Lifecycle | Consultation | VisitPatient, VisitRegistration, VisitExamination |
| Clinical | Consultation | Assessment, Diagnosis, Referral |
| Prescription | Consultation | Prescription, MedicationRequest |
| Transactions | Cashier | Transaction, Billing, Invoice |
| Payments | Cashier | PaymentSummary, PaymentDetail |
| Saga State | Backbone | Saga (per-tenant) |

---

## 4. Communication Patterns

### 4.1 Saga Flow (ADR-005)

```
Backbone (Orchestrator)
    │
    ├── RabbitMQ Trigger ─────────► Consultation
    │   saga.create_visit.step.trigger
    │                                   │
    │                                   ▼
    │                              Process Step
    │                                   │
    ◄── gRPC Callback ────────────────┘
        SagaCallbackService.ReportStepResult
```

### 4.2 Permitted Cross-Service Calls

| From | To | Method | When |
|------|----|--------|------|
| Backbone | Consultation | RabbitMQ trigger | Saga step execution |
| Consultation | Backbone | gRPC SagaCallbackService | Step completion |
| Consultation | Backbone | gRPC CanonicalVisitService | Doctor sign-off only |
| Backbone | Cashier | RabbitMQ trigger | POS saga steps |
| Cashier | Backbone | gRPC SagaCallbackService | Step completion |

---

## 5. Database Architecture

### 5.1 Multi-Tenant Pattern

- **Database per tenant**: `db_clinic_a`, `db_clinic_b`, etc.
- **Year-based schemas**: `schema_2026`, `schema_2027`
- **JWT claims**: `tenant_db`, `tenant_schema` for routing
- **ConnectionManager**: Lazy pool per database

### 5.2 ID Generation

- **Format**: ULID (26-character, lexicographically sortable)
- **Generator**: `helper.GenerateUlid()`
- **Storage**: `char(26)` or `varchar(26)`

---

## 6. Legacy Laravel Project

Path: `/var/www/projects/kalpa/wellmed/projects/wellmed-backbone/`

**Seeders:** 67 PHP files
- Orchestrators: DatabaseSeeder, LiteDatabaseSeeder, MasterSeeder
- Data: permissions, roles, forms, ICD codes, provinces, villages
- Modes: PLUS (full), LITE (restricted features)

**Key Data Files:**
- `diseases.sql` (4.3 MB) - ICD-10 codes
- `villages.sql` (7.8 MB) - Indonesian villages
- `/data/forms/*.php` - Medical examination forms
- `/data/permissions/**/*.php` - Permission hierarchy

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-03-13 | Claude | Initial comprehensive summary |
