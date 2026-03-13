# Plan 5: Migration & Seed Testing - Progress

**Plan File**: `5-migration-seed-testing-PLAN.md`
**Started**: 2026-03-13
**Status**: ✅ COMPLETE

---

## Phase 1: Add Makefile Targets ✅ COMPLETE

**Commit**: `014f118 fix: rename Makefile targets from colon to dash syntax`
**Date**: 2026-03-13

- [x] Add `migrate-seed` target
- [x] Add `migrate-fresh-seed` target
- [x] Rename colon targets to dash (migrate:fresh → migrate-fresh)
- [x] Update help section

---

## Phase 2: Test Migration ✅ COMPLETE

### Test 2.1: Migrate Fresh ✅ PASS
### Test 2.2: Migrate All ✅ PASS

---

## Phase 3: Test Seeding ✅ COMPLETE (with fixes)

### Issue Found: Unicode Constraint Missing

**Problem**: `ON CONFLICT ("flag","name")` failed because no unique constraint existed.

**Fix Applied**: Added composite unique index to `unicode.go`:
```go
Flag string `gorm:"..;uniqueIndex:idx_unicodes_flag_name,priority:1"`
Name string `gorm:"..;uniqueIndex:idx_unicodes_flag_name,priority:2"`
```

### Test 3.1: Install LITE ✅ PASS (after fix)

**Successful Seeds**:
- ✅ Country (249 records)
- ✅ Province (34 records)
- ✅ District (514 records)
- ✅ SubDistrict (7,266 records)
- ✅ Village (83,467 records)
- ✅ ICD-10 Diseases (bulk import)
- ✅ DosageForm (45 records)
- ✅ Education (9 records)
- ✅ EmployeeType (9 records)
- ✅ FamilyRole (11 records)
- ✅ FreqUnit (11 records)
- ✅ Funding (6 records)
- ✅ ItemStuffService (21 records)
- ✅ MasterVaccine (24 records)
- ✅ Occupation (10 records)
- ✅ PatientOccupation (5 parent + 48 child)
- ✅ PatientTypeService (6 records)
- ✅ PaymentMethod (6 records)
- ✅ PeopleStuff/MaritalStatus (6 records)
- ✅ Profession (6 parent + 19 child)

**Known Issues (out of scope for Plan 5)**:
- `diseases.parent_id` column doesn't exist (ICD hierarchical seeder)
- `examination_stuffs` table doesn't exist
- `services` table doesn't exist

These are schema issues to be addressed in Plan 6.

---

## Summary

| Test | Status | Notes |
|------|--------|-------|
| Add Makefile targets | ✅ Complete | Fixed colon syntax |
| migrate-fresh | ✅ Pass | Works correctly |
| migrate (all) | ✅ Pass | Works correctly |
| install LITE | ✅ Pass | After Unicode constraint fix |

---

## Commits

1. `014f118` - fix: rename Makefile targets from colon to dash syntax
2. (pending) - fix: add unique constraint to unicodes entity

---

## Commands Reference

```bash
# Fresh migration (drop all + recreate)
make migrate-fresh

# Run all migrations + indexes
make migrate

# Seed with LITE mode
make install MODE=LITE

# Combined: fresh + seed
make migrate-fresh-seed
```
