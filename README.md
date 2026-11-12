**Home** | [Architecture](architecture.md) | [Business logic](business-logic.md) | [Step by step](step-by-step.md)

---

# Digital Wallet and Transaction Processing System

A Go backend portfolio project: a ledger-based digital wallet (IDR) with deposits, withdrawals and user-to-user transfers, built for financial correctness rather than as a CRUD app.

Stack: Go, Gin, PostgreSQL (pgx), Redis, RabbitMQ, JWT, Docker, golang-migrate, Swagger, testify, k6, Prometheus, slog.

## Read in this order

| # | File | Answers | Read it when |
| --- | --- | --- | --- |
| 1 | [Business logic](business-logic.md) | What does the product do, and why is money handled this way? | First. Start here to understand the business |
| 2 | [Architecture](architecture.md) | How is the system structured? Components, flows, database, folders, decisions | Before writing code |
| 3 | [Step by step](step-by-step.md) | In what order do I build it? Phases 0 to 14, with the Phase 1 checklist | While building |

## Decisions already made

- Transfers identify the recipient by **email**.
- A **frozen** wallet can receive money but cannot send or withdraw.
- Fee is **zero by default**; the fee path is built and tested.
- Logging with **slog**; tests use **testcontainers-go**.
- Modular monolith, PostgreSQL as the only source of truth, outbox for events.

## How to use these files

Keep all four files in the same folder so the links between them work. They render best in GitHub, VS Code (with a Mermaid extension) or Obsidian, because they contain Mermaid diagrams.

## Working agreement

One phase at a time. You build, I review your code, then we move on. Each phase gets its own detailed breakdown when it starts.
