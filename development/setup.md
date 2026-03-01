# Local Development Setup

**Version:** 0.1 (stub)
**Date:** 01 March 2026
**Status:** Stub — content to be added
**Maintained by:** Alex

---

## 1. Prerequisites

- [ ] TODO: Document required Go version
- [ ] TODO: Document required tooling (golangci-lint, protoc, docker, etc.)
- [ ] TODO: Document AWS CLI setup requirements (for Secrets Manager access)

---

## 2. Quick Start (Gateway)

For the Gateway specifically, see [wellmed-gateway-go/README.md §4](https://github.com/Kalpa-Health/wellmed-gateway-go/blob/main/README.md).

```bash
# Prerequisites: Go 1.22+, Redis on localhost:6379, Backbone on localhost:50051
cp .env.example .env
# edit .env with local values
go run ./cmd/main.go
```

---

## 3. Running Infrastructure Locally

- [ ] TODO: Document Docker Compose setup for Postgres, Redis, RabbitMQ
- [ ] TODO: Document how to run multiple services locally for integration testing
- [ ] TODO: Document how to point local services at staging infrastructure for testing

---

## 4. Environment Variables

- [ ] TODO: Document the full set of required environment variables across services
- [ ] TODO: Document which values come from AWS Secrets Manager vs. local `.env`

---

## 5. IDE Setup

- [ ] TODO: Document recommended VS Code extensions (Go, Protobuf, etc.)
- [ ] TODO: Document recommended GoLand configuration

---

## 6. Pre-Push Checks

Before pushing any code, run:

```bash
make lint    # golangci-lint
make test    # Unit tests
make build   # Confirm it compiles
```

See [`development/ci-cd-guide.md`](ci-cd-guide.md) for the full CI pipeline.

---

# Edit Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 0.1 | 01 Mar 2026 | Alex + Claude | Initial stub |
