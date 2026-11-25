# ledger

A double-entry wallet/ledger service in C# / ASP.NET Core, built from scratch as a deliberate-practice project.

The goal is not a product. The goal is to build a system that has to stay correct under concurrency, retries and partial failure, then break it on purpose and learn from what breaks. Every important design decision is written down as an ADR and defended in review.

---

## 1. Why this project

I want to grow from an engineer who implements features that work into one who has judgment about production systems: concurrency, load, failure, observability, recovery and trade-offs.

A ledger is a good vehicle for that because the rules are strict and failures are visible. Money can't be created or lost. Every bug shows up as a balance that doesn't add up.

**Goals**
- Correctness under concurrent access (no double-spend, no lost updates).
- Safe retries end to end (idempotency).
- Reliable event publishing (no lost or phantom events).
- Observable: I can explain what the system is doing from its telemetry alone.
- Every non-trivial decision documented as an ADR with trade-offs.

**Non-goals**
- UI, authentication, multi-tenancy.
- Currency conversion (each account has a single currency; cross-currency transfers are rejected).
- Kubernetes or cloud deployment. Docker Compose is enough.

---

## 2. Domain

| Concept | Meaning |
|---|---|
| **Account** | Holds money in one currency. Never deleted. |
| **Transfer** | A request to move an amount from account A to account B. Carries an idempotency key. |
| **Ledger entry** | An immutable debit or credit line. A transfer produces exactly two entries. |
| **Balance** | Derived from entries: `sum(credits) - sum(debits)`. |

**Invariants**
1. The entries of every transfer sum to zero.
2. Entries are append-only. A mistake is fixed with a reversing transfer, never with an UPDATE or DELETE.
3. A balance never goes below zero, unless an account is explicitly allowed an overdraft.
4. The same idempotency key never produces a second transfer.
5. Amounts are stored as integers in minor units (`long`, cents). Never `double`.

---

## 3. API (v1)

| Method | Path | Notes |
|---|---|---|
| `POST` | `/accounts` | Create an account (currency, optional overdraft limit). |
| `GET` | `/accounts/{id}` | Account + current balance. |
| `GET` | `/accounts/{id}/entries` | Paged entry history. |
| `POST` | `/transfers` | Requires an `Idempotency-Key` header. Returns the same result when retried. |
| `GET` | `/transfers/{id}` | Transfer status. |
| `GET` | `/health/live`, `/health/ready` | Liveness / readiness (DB, broker). |

---

## 4. Architecture

```
            ┌──────────────┐
 HTTP ───►  │  Ledger.Api  │ ── single DB transaction ──►  PostgreSQL
            └──────────────┘                                 ├─ accounts
                                                             ├─ transfers
                                                             ├─ ledger_entries
                                                             └─ outbox
                                                                  │
                                         Outbox relay (poller) ◄──┘
                                                │
                                                ▼
                                            RabbitMQ ──► Ledger.Worker (consumers)
                                                             └─ dead-letter queue
            Redis: balance cache (M4)          OpenTelemetry ──► Jaeger + Prometheus/Grafana
```

**Stack:** .NET 10, ASP.NET Core, PostgreSQL, RabbitMQ, Redis, Docker Compose, xUnit + Testcontainers, OpenTelemetry, k6 for load tests. Everything free and runs locally.

**Planned layout**
```
src/
  Ledger.Api/          HTTP endpoints, validation
  Ledger.Domain/       entities, invariants, no infrastructure
  Ledger.Infrastructure/  persistence, outbox, messaging
  Ledger.Worker/       outbox relay + consumers
tests/
  Ledger.UnitTests/
  Ledger.IntegrationTests/   real Postgres/RabbitMQ via Testcontainers
docs/adr/              architecture decision records
load/                  k6 scripts
docker-compose.yml
```

---

## 5. Milestones

| # | Target | Scope | Done when |
|---|---|---|---|
| **M0** | 30 Nov | Solution skeleton, Docker Compose (Postgres), CI on GitHub Actions (build + test), health checks | `docker compose up` + green CI |
| **M1** | 10 Dec | Accounts, transfers, double-entry entries, balance from entries, migrations | Integration tests prove invariants 1, 2, 5 |
| **M2** | 31 Dec | Concurrency control + idempotency keys | A parallel test of 100 concurrent withdrawals never overdraws; retried requests return the original result. **Demo + talk to mentor** |
| **M3** | 20 Jan | Transactional outbox, RabbitMQ publishing, consumer with retries + DLQ, idempotent consumer | Killing the broker or the worker mid-flight loses no events and creates no duplicates. **Talk to a wider audience** |
| **M4** | Feb | Redis balance cache (and its invalidation), OpenTelemetry traces + metrics, dashboards | One trace follows a transfer from HTTP to consumer |
| **M5** | Mar | Load and failure experiments (see §6), findings written up | A written report with numbers and fixes |

Dates are targets for ~6–8 h/week. If a milestone slips, the scope gets cut, not the tests.

---

## 6. Break-it experiments

Each experiment: write down a prediction, run it, measure, explain, fix, add a regression test.

| Milestone | Experiment |
|---|---|
| M2 | Naive read-modify-write balance check under 100 parallel requests: how much money is created? |
| M2 | Compare the fixes: `SELECT … FOR UPDATE`, optimistic concurrency (version column), conditional atomic `UPDATE`. Measure throughput and conflicts for each. |
| M2 | Client retries after a timeout. Was the transfer applied once or twice? |
| M3 | Publish to RabbitMQ *after* commit without an outbox, then kill the process in between. Count the lost events. |
| M3 | Restart the broker during publishing; drop the connection mid-consume; send a poison message. |
| M4 | Stale cached balance after a write. When can a user see it, and does it matter? |
| M5 | k6 ramp until latency degrades; find the bottleneck from telemetry alone. |
| M5 | Postgres connection-pool exhaustion; slow DB (added latency); see what the API does. |

---

## 7. Decisions to make (ADRs)

Written in `docs/adr/NNNN-title.md`: context → options → decision → consequences.

1. EF Core vs Dapper for the write path.
2. Balance derived from entries vs stored running balance vs both.
3. Concurrency strategy (pessimistic vs optimistic vs conditional update).
4. Idempotency storage: table + unique constraint, TTL, what is stored as the response.
5. Outbox relay: polling vs Postgres `LISTEN/NOTIFY`; delivery guarantees.
6. Consumer idempotency: inbox table vs natural keys.
7. Cache strategy and invalidation.

---

## 8. How I work on this

- Work is split into small tickets; one ticket = one PR.
- **I write the code myself.** I learn from docs, source code and debugging, not from generated code.
- Every PR is reviewed like a senior review: I must be able to explain and defend each decision.
- My mentor does design reviews of ADRs and reviews the milestone PRs; checkpoint after each milestone.
- Theory only when a gap appears; notes go into the relevant ADR or experiment write-up.

---

## Running locally

_Filled in at M0._
