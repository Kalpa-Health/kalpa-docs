# Testing Patterns — Quick Reference

**Version:** 0.1 (stub)
**Date:** 01 March 2026
**Status:** Stub — quick reference outline; full examples to be added
**Maintained by:** Alex

> For the full testing strategy, values, and CI configuration, see [`wellmed-testing-architecture.md`](../wellmed-testing-architecture.md).
> For coverage targets and layer definitions, see [`wellmed-testing-architecture.md §2`](../wellmed-testing-architecture.md).

---

## 1. Which Test to Write?

| Situation | Test type | Location |
|-----------|-----------|----------|
| Testing a single function with mocked deps | Unit test (Category A) | Same directory as code |
| Testing service + database together | Integration test (Category B) | `/tests/integration/` |
| Verifying SATU SEHAT or P-Care response format | Contract test (Category C) | `/tests/contracts/` |
| Testing who can do what (RBAC) | Permission test (Category F) | `/tests/permissions/` |
| Testing a multi-step saga with compensation | SAGA test (Category G) | `/tests/sagas/` |

---

## 2. Unit Test Patterns

### 2.1 Service Layer (Table-Driven)

```go
func TestVisitService_CreateVisit(t *testing.T) {
    tests := []struct {
        name      string
        patientID string
        doctorID  string
        setup     func(*mocks.PatientRepository, *mocks.ScheduleService)
        wantErr   error
    }{
        {
            name:      "creates visit successfully",
            patientID: "01HZABCDEF...",
            doctorID:  "01HZGHIJKL...",
            setup: func(pr *mocks.PatientRepository, ss *mocks.ScheduleService) {
                pr.On("GetByID", mock.Anything, "01HZABCDEF...").Return(&Patient{Status: "active"}, nil)
                ss.On("IsDoctorAvailable", mock.Anything, "01HZGHIJKL...", mock.Anything).Return(true)
            },
        },
        {
            name:      "returns error for inactive patient",
            patientID: "01HZABCDEF...",
            setup: func(pr *mocks.PatientRepository, ss *mocks.ScheduleService) {
                pr.On("GetByID", mock.Anything, "01HZABCDEF...").Return(&Patient{Status: "inactive"}, nil)
            },
            wantErr: ErrPatientInactive,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            patientRepo := mocks.NewPatientRepository(t)
            scheduleService := mocks.NewScheduleService(t)
            tt.setup(patientRepo, scheduleService)

            svc := NewVisitService(patientRepo, scheduleService)
            _, err := svc.CreateVisit(context.Background(), tt.patientID, tt.doctorID)

            assert.ErrorIs(t, err, tt.wantErr)
        })
    }
}
```

---

## 3. Mock Generation

- [ ] TODO: Document how to generate mocks (mockery or testify/mock?)
- [ ] TODO: Document mock location convention

---

## 4. Integration Test Pattern

- [ ] TODO: Add integration test example with real Postgres using testcontainers or Docker

---

## 5. SAGA Test Pattern

See [`wellmed-testing-architecture.md §4.1.2`](../wellmed-testing-architecture.md) for the full SAGA test pattern.

---

## 6. Test Utilities (`/pkg/testutil/`)

- [ ] TODO: Document what's available in the shared test utility package

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 01 Mar 2026 | Alex + Claude | Initial stub with unit test table-driven pattern |
