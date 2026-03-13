# Plan 5: Migration & Seed Testing

**Status**: In Progress
**Author**: Claude
**Created**: 2026-03-13
**Related Plans**: Plan 3 (Go Seeder), Plan 4 (Go Migration Infrastructure)

---

## Objective

Test the complete migration and seeding pipeline after Plan 4 implementation:
1. Test `migrate fresh` - drop all tables and recreate
2. Test `migrate all` - run all migrations
3. Test `install LITE` - seed with LITE dataset
4. Add convenience Makefile targets

---

## Test Cases

### Test 1: Migrate Fresh
```bash
make migrate:fresh
```
Expected: All tables dropped and recreated successfully

### Test 2: Install LITE
```bash
make install MODE=LITE
```
Expected: LITE dataset seeded successfully

### Test 3: Combined Commands (New Makefile Targets)
```bash
make migrate-seed          # migrate all + seed LITE
make migrate-fresh-seed    # migrate fresh + seed LITE
```

---

## New Makefile Targets

```makefile
## Run migrations and seed with LITE (default)
migrate-seed:
	@echo "🔧 Running migrations + seeding..."
	@go run cmd/main.go migrate all
	@go run cmd/main.go install LITE
	@echo "✅ Migration and seeding completed"

## Fresh migration and seed with LITE
migrate-fresh-seed:
	@echo "⚠️  Running fresh migration + seeding..."
	@go run cmd/main.go migrate fresh
	@go run cmd/main.go install LITE
	@echo "✅ Fresh migration and seeding completed"
```

---

## Success Criteria

- [ ] `migrate fresh` completes without errors
- [ ] `migrate all` completes without errors
- [ ] `install LITE` completes without errors
- [ ] All 22 new entity tables created
- [ ] Reference data seeded correctly
