[Home](README.md) | [Architecture](architecture.md) | [Business logic](business-logic.md) | **Step by step**

---

# Step by step: build plan

The order to build the wallet system, one phase at a time. Read [Business logic](business-logic.md) first for the *why*, and [Architecture](architecture.md) for the *what*. We work phase by phase: you build, I review your code, then we move on.

Decisions already made: transfers by **email**, frozen wallets **block debits only**, fee **zero by default**, **slog** for logging, **testcontainers-go** for tests.

Contents: 1 Dependency map, 2 Changes to the original order, 3 Phases 0 to 14, 4 Phase 1 in detail with checklist.

---

## 1. Dependency map

```mermaid
flowchart LR
  P0[0 Design] --> P1[1 Project + Docker]
  P1 --> P2[2 Postgres + migrations]
  P2 --> P3[3 Auth + users]
  P3 --> P4[4 Wallets]
  P4 --> P5[5 Ledger core]
  P5 --> P6[6 Deposit + withdrawal + idempotency]
  P6 --> P7[7 Transfer + locking]
  P7 --> P8[8 Concurrency proof]
  P8 --> P9[9 Redis]
  P8 --> P10[10 RabbitMQ + outbox]
  P8 --> P11[11 Admin + audit + reversal]
  P9 --> P13[13 Load test]
  P10 --> P13
  P11 --> P12[12 Full test suite]
  P12 --> P13
  P13 --> P14[14 Monitoring + docs + deploy]
```

## 2. Changes to the original order

1. **Idempotency moves to Phase 6.** The first money endpoint must never ship without replay protection. Phase 8 proves it under concurrency.
2. **Row locking arrives in Phase 7**, with the first operation that needs two locks. Phase 8 is the concurrency test suite and fixes.
3. **Outbox rows are written from Phase 6**, with no broker yet. Phase 10 only adds the relay and consumers.
4. **Tests are written inside every phase.** Phase 12 is the consolidation pass.
5. **Reversal is built in Phase 11** because it needs admin auth and audit logging. `audit_logs` and `outbox_events` tables are created in Phase 2.
6. **Redis comes after correctness.** Login rate limiting may be pulled earlier.

## 3. Phases

Each phase below is a summary. At the start of each one, I expand it into tasks, exact files, migrations, endpoints, tests and Definition of Done.

### Phase 0: Design (done)

Output: the agreed design. Done when you can redraw the transfer sequence diagram from memory and explain why each lock exists.

### Phase 1: Project initialization and Docker

- Build: Go module, folder skeleton, Gin server, `/healthz`, config, slog, Dockerfile, Compose with Postgres, Redis, RabbitMQ, Makefile.
- Done when: `docker compose up` serves `/healthz` and graceful shutdown works.
- Avoid: committing `.env`; one giant `main.go`; no graceful shutdown.
- Detailed below.

### Phase 2: PostgreSQL connection and migrations

- Build: `pgxpool` with limits and ping, golang-migrate files for all tables (including triggers and seeded system accounts), `TxManager` skeleton.
- Done when: `migrate up` and `down` work on an empty DB; an integration test shows a direct `UPDATE ledger_entries` fails.
- Avoid: float money; editing an applied migration; no `down` files.

### Phase 3: Authentication and user management

- Build: register (user only), login, refresh with rotation and reuse detection, logout, `/users/me`, JWT and role middleware, Argon2id.
- Done when: expired access token gives 401; reusing an old refresh token revokes the family; each case is tested.
- Avoid: raw refresh tokens in the DB; revealing whether an email exists; long-lived access tokens.

### Phase 4: Wallet management

- Build: wallet created in the **same DB transaction** as registration, wallet number generator, `GET /wallets/me`.
- Done when: a forced failure after the user insert leaves no orphan user.
- Avoid: creating the wallet in a second request.

### Phase 5: Double-entry ledger core

- Build: `Money`, ledger domain with balanced-set validation, `PostEntries`, system accounts, derived-balance and reconciliation queries, triggers.
- Done when: unbalanced entry sets are rejected in Go and in the database; derived equals cached.
- Avoid: validating balance only in Go; any path that updates `wallets.balance` without entries.

### Phase 6: Deposit and withdrawal with idempotency

- Build: idempotency repo and helper, deposit, withdrawal (PENDING, `PAYOUT_CLEARING`), reference numbers, outbox insert, history with keyset pagination.
- Done when: replaying a deposit with the same key returns the same response and one ledger set; same key with a different body returns 422.
- Avoid: idempotency check outside the money transaction.

### Phase 7: Transfer processing

- Build: transfer exactly as in [Architecture section 6](architecture.md), ordered `FOR UPDATE`, fee calculation, retry on `40001` and `40P01`, email lookup.
- Done when: every row of the failure table has a passing test.
- Avoid: locking in request order; checking balance before locking; network calls inside a transaction.

### Phase 8: Concurrency control and idempotency hardening

- Build: concurrency tests (100 goroutines spending one balance; random pair transfers; 20 requests with one key), fixes, pool tuning notes.
- Done when: after every test, reconciliation returns zero rows and no wallet is negative; `go test -race` is clean.
- Avoid: asserting only HTTP codes; sleeping instead of synchronizing.

### Phase 9: Redis integration

- Build: rate limiter (login, transfer, lookup), login lockout counters, optional balance read cache.
- Done when: Redis down still leaves money handling correct.
- Avoid: Redis as source of truth; keys without TTL.

### Phase 10: RabbitMQ and transactional outbox

- Build: relay with `FOR UPDATE SKIP LOCKED`, publisher confirms, topic exchange, consumers (notification, payout simulator), retry with dead-letter queue, consumer dedupe by `event_id`.
- Done when: killing RabbitMQ during traffic loses no events; duplicates are harmless.
- Avoid: publishing inside the DB transaction; auto-ack; infinite retry without a DLQ.

### Phase 11: Admin, audit logs, reversal

- Build: admin endpoints, freeze/unfreeze with reasons, audit rows written in the same transaction as the action, reversal with state-machine guard.
- Done when: every admin mutation has an audit row; a reversal creates mirrored entries and cannot run twice.
- Avoid: audit logging after commit; ledger edits "for convenience".

### Phase 12: Integration and concurrency testing

- Build: testcontainers CI job, failure injection, coverage report, invariant checker script.
- Done when: unit, integration and concurrency suites are green from a clean checkout.

### Phase 13: Load testing and optimization

- Build: k6 scenarios (login, read balance, mixed transfers, hot-wallet contention), baseline numbers, one change at a time with before/after results.
- Done when: you have a result table (throughput, p95 and p99 latency, error rate, pool saturation) and can explain the bottleneck.
- Avoid: tuning without measurement; testing with tiny data.

### Phase 14: Monitoring, documentation, deployment

- Build: Prometheus metrics and scrape config, optional Grafana dashboard, Swagger, README with diagrams, decision notes, compose deployment guide.
- Done when: a stranger can clone, run, call the API and read why each decision was made.

---

## 4. Phase 1 in detail

**Goal:** `docker compose up` starts PostgreSQL, Redis, RabbitMQ and a Go API that answers `GET /healthz`, logs structured JSON, and shuts down cleanly on SIGTERM. No business logic yet.

### Environment variables

| Variable | Example | Used from |
| --- | --- | --- |
| `APP_ENV` | `local` / `production` | Phase 1 |
| `HTTP_ADDR` | `:8080` | Phase 1 |
| `LOG_LEVEL` | `info` | Phase 1 |
| `SHUTDOWN_TIMEOUT` | `10s` | Phase 1 |
| `DATABASE_URL` | `postgres://wallet:wallet@postgres:5432/wallet?sslmode=disable` | Phase 2 |
| `REDIS_ADDR` | `redis:6379` | Phase 9 |
| `RABBITMQ_URL` | `amqp://wallet:wallet@rabbitmq:5672/` | Phase 10 |
| `JWT_SECRET` | 32+ random bytes | Phase 3 |

Inside Compose, services reach each other by service name. When running the API on your host with `make run`, use `localhost`.

### Compose reference

Development-only credentials. Pin image tags; never use `latest`.

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: wallet
      POSTGRES_PASSWORD: wallet
      POSTGRES_DB: wallet
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U wallet -d wallet"]
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7
    ports: ["6379:6379"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10

  rabbitmq:
    image: rabbitmq:3-management
    environment:
      RABBITMQ_DEFAULT_USER: wallet
      RABBITMQ_DEFAULT_PASS: wallet
    ports: ["5672:5672", "15672:15672"]
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 10s
      timeout: 5s
      retries: 10

  api:
    build: .
    env_file: .env
    ports: ["8080:8080"]
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_healthy }
      rabbitmq: { condition: service_healthy }

volumes:
  pgdata:
```

### Why each task exists

| Task | Why |
| --- | --- |
| Typed config that fails fast | Config is a dependency; missing values should stop startup, not break at runtime |
| `gin.New()` not `gin.Default()` | You choose the middleware and logging |
| Request ID middleware | Ties a log line to a request across the system |
| Server timeouts | Slow clients cannot exhaust the server |
| Graceful shutdown | Docker sends SIGTERM; in-flight money requests must finish |
| Multi-stage Dockerfile, non-root | Small image, small attack surface |
| Healthchecks with `service_healthy` | The API starts only after its dependencies are ready |

### Phase 1 checklist

Tick items as you finish them. Order matters.

**A. Setup**

- [ ] Install Go (latest stable), Docker with Compose v2, make, Git; confirm with `go version`, `docker compose version`
- [ ] Create the GitHub repo and clone it
- [ ] `go mod init github.com/<you>/wallet-system`
- [ ] Add `.gitignore` (binaries, `.env`, coverage files)
- [ ] Add `.env.example` with every variable above
- [ ] Add a README stub
- [ ] Commit: `chore: init module`

**B. Skeleton and config**

- [ ] Create folders: `cmd/api`, `internal/config`, `internal/handler/http/middleware`, `internal/infrastructure/logger`, `migrations`, `docs`, `scripts`, `test`
- [ ] Add `.gitkeep` to empty folders
- [ ] Write `config.Config` and `Load()` that returns an error on missing or invalid values
- [ ] Write table-driven `config_test.go` (valid, missing required, bad duration, defaults)
- [ ] Commit: `feat: typed config`

**C. Logging and HTTP**

- [ ] Write the slog logger (JSON outside local, text locally, level from config)
- [ ] Request-ID middleware (reuse incoming header or generate a UUID; echo `X-Request-ID`)
- [ ] Recovery middleware (log panic with request id, return the standard error envelope)
- [ ] Access-log middleware (method, path, status, latency, request id; no bodies, no `Authorization`)
- [ ] Router with `gin.New()` and the middleware
- [ ] `GET /healthz` and `GET /readyz`
- [ ] `httptest` tests for health and for a panicking route
- [ ] Commit: `feat: http server and health`

**D. Server lifecycle**

- [ ] `cmd/api/main.go`: load config, build logger and router
- [ ] `http.Server` with `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, `IdleTimeout`
- [ ] Start in a goroutine; wait on `signal.NotifyContext` for SIGINT and SIGTERM
- [ ] `Server.Shutdown` with the `SHUTDOWN_TIMEOUT` deadline; log start and stop
- [ ] Run `go run ./cmd/api`; confirm Ctrl+C shuts down cleanly
- [ ] Commit: `feat: graceful shutdown`

**E. Containers**

- [ ] Multi-stage Dockerfile (module download layer, `CGO_ENABLED=0`, minimal runtime image, non-root user)
- [ ] `docker-compose.yml` with healthchecks and `depends_on: service_healthy`
- [ ] Create local `.env` from `.env.example` (service names as hosts for Compose)
- [ ] `docker compose up --build`; all four containers healthy
- [ ] `curl -i localhost:8080/healthz` returns 200 with `X-Request-ID`
- [ ] `docker compose stop api`; confirm the shutdown log line
- [ ] Open RabbitMQ management at `localhost:15672` and log in
- [ ] Commit: `build: docker and compose`

**F. Developer commands**

- [ ] Makefile targets: `run`, `build`, `test`, `up`, `down`, `logs`
- [ ] Run `make test` and `make up` from a fresh clone

**G. Wrap-up**

- [ ] Every item in the Definition of Done below is true
- [ ] Write 3 sentences in `docs/decisions.md`: why `gin.New()`, why server timeouts, why healthchecks gate startup
- [ ] Push to GitHub
- [ ] Send the review package below

### Definition of Done

- [ ] `make test` passes
- [ ] `docker compose up --build` brings up all four containers healthy from a clean clone
- [ ] `/healthz` and `/readyz` respond; request id header present and logged
- [ ] SIGTERM triggers graceful shutdown (visible in logs)
- [ ] Container runs as non-root and holds only the binary
- [ ] `.env` is git-ignored and `.env.example` documents every variable
- [ ] You can explain: why `gin.New()`, why server timeouts, why multi-stage builds, why healthchecks gate `depends_on`

### Common mistakes

1. Using `gin.Default()` and inheriting logging you do not control.
2. `http.ListenAndServe` with no timeouts.
3. Ignoring the context from `signal.NotifyContext`, so Ctrl+C kills requests.
4. Committing `.env` or real passwords.
5. Using `localhost` for service addresses inside containers.
6. Using the `latest` image tag.
7. Putting everything in `main.go`.

### Review package to send me

The repository tree (`tree -I .git`), `go.mod`, `config.go` and its test, `main.go`, `router.go` and the three middleware files, `Dockerfile`, `docker-compose.yml`, `Makefile`, the output of `docker compose up --build`, and `curl -i localhost:8080/healthz`. Tell me what confused you.

---

[Home](README.md) | [Architecture](architecture.md) | [Business logic](business-logic.md) | **Step by step**
