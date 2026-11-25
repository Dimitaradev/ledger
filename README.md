# ledger

A sportsbook wallet built on a double-entry ledger, in C# / ASP.NET Core. Players deposit, place bets on real football matches, get paid out automatically when results come in from a public sports API, and withdraw. Every cent is accounted for.

This is a deliberate-practice project, not a product. The goal is to build a system that has to stay correct under concurrency, retries, unreliable third parties and partial failure. Then I break it on purpose and learn from what breaks. Every important design decision is written down as an ADR and defended in review.

---

## 1. What it does

A player-facing wallet service for a fixed-odds sportsbook, plus the settlement pipeline behind it.

1. **Wallet:** a player deposits money, sees their balance and history, and withdraws.
2. **Betting:** the player places a single bet on a football match (home / draw / away) at fixed odds. The stake leaves their balance immediately.
3. **Settlement:** when the match ends, the system gets the result from an external sports API and settles every open bet on it: winners are paid out, losers' stakes go to the house, and postponed or cancelled matches are refunded.
4. **Events:** every money movement publishes an event, so other parts of the system can react (notifications, a deposit/loss-limit check, a transaction-history view).

There is no UI. Everything happens through the HTTP API, and the demo is driven by scripts and tests.

### Every operation is a transfer

The core of the system is a double-entry ledger. There are two kinds of accounts:
- **Player accounts:** one wallet per player.
- **System accounts:** `external-cash` (the outside world: card, bank), `bets-in-play` (stakes held until settlement) and `house` (the bookmaker's profit and loss).

Each business operation is one or more transfers between accounts:

| Player action / event | Transfers |
|---|---|
| Deposit 50 | `external-cash` → `player` 50 |
| Bet 20 on Home @ 3.00 | `player` → `bets-in-play` 20 |
| Bet wins (pays 60) | `bets-in-play` → `player` 20, `house` → `player` 40 |
| Bet loses | `bets-in-play` → `house` 20 |
| Match void / postponed | `bets-in-play` → `player` 20 (refund) |
| Withdraw 30 | `player` → `external-cash` 30 |

Every transfer produces exactly two entries (a debit and a credit) that sum to zero. So the sum of all balances in the system is always zero, and that is checked by a test. If it's ever not zero, money was created or lost.

### Life of a bet

```
 PLACE                         HOLD                    SETTLE
 POST /bets ───► check balance + odds ───► Open ───► Won | Lost | Void
 (Idempotency-Key)  (concurrency-safe)              ▲
                                                    │ MatchFinished event
                         Results poller ──► football-data.org
```

1. The player calls `POST /bets` with an `Idempotency-Key`. If the key has been seen, the original response is returned and nothing else happens.
2. In **one database transaction**: lock in the odds, check the balance (concurrency-safe), write the bet, write the transfer and its entries, and write a `BetPlaced` event to the outbox.
3. The results poller asks the sports API for finished matches. A newly finished match produces a `MatchFinished` event.
4. The settlement worker consumes it and settles each open bet on that match exactly once, even if the event arrives twice.
5. Payout transfers publish `BetSettled` events. Consumers send notifications and update read models.

---

## 2. External data: match results

**Provider:** [football-data.org](https://www.football-data.org/) (v4 API).

- Free tier: a token in the `X-Auth-Token` header, 12 competitions (Premier League, La Liga, Champions League, …), 10 requests per minute, scores slightly delayed.
- No odds in the free tier, so **odds are set inside this system**: an admin endpoint publishes a market (match + odds for home/draw/away). Odds are copied onto the bet when it's placed and never change afterwards.
- Attribution to football-data.org as its terms require.

**Design rules for the integration**
- The provider sits behind an `IMatchResultsProvider` interface (an anti-corruption layer). The rest of the system never sees football-data.org's JSON.
- There's a **fake provider** for tests and demos, so I can finish a match on command instead of waiting for a real one.
- 10 requests per minute is a real constraint: the poller batches requests, caches responses, respects `429`/`Retry-After`, and uses timeouts, retries and a circuit breaker.
- Results can be **corrected** by the provider after the fact. A correction re-settles bets with reversing transfers; nothing is ever updated in place.

Fallback if it becomes unavailable: [OpenLigaDB](https://api.openligadb.de) (free, no key, mainly German football). Swapping it in means writing one more adapter.

---

## 3. Rules and invariants

1. The entries of every transfer sum to zero; the sum of all balances is always zero.
2. Entries are append-only. A mistake is fixed with a reversing transfer, never with an UPDATE or DELETE.
3. A player's balance never goes below zero. System accounts may.
4. The same idempotency key never produces a second operation.
5. A bet is settled at most once per result. A corrected result produces a reversal plus a new settlement, never a second payout on top of the first.
6. Odds are fixed at placement.
7. Amounts are stored as integers in minor units (`long`, stotinki/cents). Never `double`.

**Non-goals:** real money or payments, UI, auth beyond an API key, live (in-play) betting, multi-leg accumulators, odds modelling, currency conversion, Kubernetes or cloud deployment.

---

## 4. API (v1)

| Method | Path | Notes |
|---|---|---|
| `POST` | `/players` | Creates a player and their wallet. |
| `GET` | `/players/{id}/wallet` | Balance + open bets. |
| `GET` | `/players/{id}/transactions` | Paged ledger history. |
| `POST` | `/players/{id}/deposits` | `Idempotency-Key` required. |
| `POST` | `/players/{id}/withdrawals` | `Idempotency-Key` required. Fails if funds are insufficient. |
| `GET` | `/matches` | Upcoming matches with open markets. |
| `POST` | `/admin/markets` | Publish odds for a match. |
| `POST` | `/bets` | `Idempotency-Key` required. Body: match, selection, stake, expected odds. |
| `GET` | `/bets/{id}` | Bet status: `Open`, `Won`, `Lost`, `Void`. |
| `POST` | `/admin/matches/{id}/result` | Fake provider only: finish a match manually. |
| `GET` | `/health/live`, `/health/ready` | Liveness / readiness (DB, broker, provider). |

---

## 5. Architecture

```
            ┌──────────────┐
 HTTP ───►  │  Ledger.Api  │ ── one DB transaction ──►  PostgreSQL
            └──────────────┘                              ├─ accounts, transfers, ledger_entries
                                                          ├─ matches, markets, bets
                                                          ├─ idempotency_keys
                                                          └─ outbox
                                                               │
                                     Outbox relay ◄────────────┘
                                          │
                                          ▼
 football-data.org ◄── Results poller ─► RabbitMQ ──► Ledger.Worker
                                                         ├─ settlement consumer
                                                         ├─ notifications / limits / read models
                                                         └─ dead-letter queue
 Redis: balance + provider-response cache    OpenTelemetry ──► Jaeger + Prometheus/Grafana
```

**Stack:** .NET 10, ASP.NET Core, PostgreSQL, RabbitMQ, Redis, Polly (resilience), Docker Compose, xUnit + Testcontainers, OpenTelemetry, k6 for load tests. Everything free and runs locally.

**Planned layout**
```
src/
  Ledger.Api/              HTTP endpoints, validation
  Ledger.Domain/           ledger, bets, settlement rules, no infrastructure
  Ledger.Infrastructure/   persistence, outbox, messaging, results providers
  Ledger.Worker/           outbox relay, results poller, consumers
tests/
  Ledger.UnitTests/
  Ledger.IntegrationTests/  real Postgres/RabbitMQ via Testcontainers
docs/adr/                  architecture decision records
load/                      k6 scripts
docker-compose.yml
```

---

## 6. Milestones

| # | Target | Scope | Done when |
|---|---|---|---|
| **M0** | 30 Nov | Solution skeleton, Docker Compose (Postgres), CI on GitHub Actions (build + test), health checks | `docker compose up` + green CI |
| **M1** | 10 Dec | Ledger core: players, system accounts, deposits, withdrawals, entries, balances, migrations | Tests prove invariants 1, 2, 3, 7, including the system-wide "sum is zero" check |
| **M2** | 31 Dec | Matches + markets (manual), bet placement, concurrency control, idempotency keys | 100 concurrent bets on one wallet never overdraw it; retried requests return the original result. **Demo + talk to mentor** |
| **M3** | 20 Jan | Outbox + RabbitMQ, settlement worker (win / lose / void) driven by the fake provider, idempotent consumers, DLQ | Killing the broker or worker mid-settlement loses no payouts and duplicates none. **Talk to a wider audience** |
| **M4** | Feb | football-data.org adapter: poller, rate-limit handling, caching, retries + circuit breaker, result corrections | Real Premier League matches settle bets end to end; provider outages don't break betting |
| **M5** | Mar | OpenTelemetry traces + metrics, dashboards, load and failure experiments (§7), write-up | One trace follows a bet from HTTP to payout; a report with numbers and fixes |

Dates are targets for ~6–8 h/week. If a milestone slips, the scope gets cut, not the tests.

---

## 7. Break-it experiments

Each experiment: write down a prediction, run it, measure, explain, fix, add a regression test.

| Milestone | Experiment |
|---|---|
| M2 | Naive read-modify-write balance check, 100 parallel bets on a 30 balance: how much money is created? |
| M2 | Compare the fixes: `SELECT … FOR UPDATE`, optimistic concurrency (version column), conditional atomic `UPDATE`. Measure throughput and conflicts for each. |
| M2 | Client times out and retries a bet. Placed once or twice? |
| M3 | Publish `BetSettled` after commit without an outbox, then kill the process in between. Count the lost events. |
| M3 | Deliver `MatchFinished` twice; restart the broker mid-settlement; send a poison message. |
| M4 | Provider returns `429`, times out, is down for 10 minutes. Does betting still work? Do bets settle afterwards? |
| M4 | Provider corrects a result (2–1 becomes 2–2) after payouts. Does the ledger end up right? |
| M5 | Stale cached balance after a bet. When can a player see it, and does it matter? |
| M5 | k6 ramp on bet placement until latency degrades; find the bottleneck from telemetry alone. |
| M5 | Postgres connection-pool exhaustion; added DB latency. What does the API do? |

---

## 8. Decisions to make (ADRs)

Written in `docs/adr/NNNN-title.md`: context → options → decision → consequences.

1. EF Core vs Dapper for the write path.
2. Balance derived from entries vs stored running balance vs both.
3. Concurrency strategy for bet placement (pessimistic vs optimistic vs conditional update).
4. Idempotency storage: table + unique constraint, TTL, what is stored as the response.
5. Outbox relay: polling vs Postgres `LISTEN/NOTIFY`; delivery guarantees.
6. Consumer idempotency: inbox table vs natural keys (bet id + result version).
7. Results integration: polling cadence under 10 req/min, caching, how a "finished" match is detected.
8. Handling result corrections: reversal + re-settlement vs manual review.
9. Cache strategy and invalidation.

---

## 9. How I work on this

- Work is split into small tickets; one ticket = one PR.
- **I write the code myself.** I learn from docs, source code and debugging, not from generated code.
- Every PR is reviewed like a senior review: I must be able to explain and defend each decision.
- My mentor does design reviews of the ADRs and reviews the milestone PRs; checkpoint after each milestone.
- Theory only when a gap appears; notes go into the relevant ADR or experiment write-up.

---

## Running locally

_Filled in at M0._

---

_Match data provided by [football-data.org](https://www.football-data.org/)._
