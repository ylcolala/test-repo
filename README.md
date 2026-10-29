# Settlement Flow Analysis — Settlement-Reconciliation-Microservice

I've read the code. Here's the analysis, grounded in what this service actually does today.

---

# 1. The flow today — in small steps

The service does **one thing**: compare a *trade feed* against a *ledger feed* by `trade_id`, and shout when they disagree. It is a **batch, end-of-day, two-source comparison**, not a real-time settlement engine.

```
                  ┌──────────────┐
 18:00 file drop  │ Trade feed   │──┐
                  └──────────────┘  │   ┌──────────────────┐   ┌───────────────┐
                                    ├──▶│ INGEST (Spring   │──▶│ Postgres      │
                  ┌──────────────┐  │   │ Batch, 500/chunk)│   │ trade_ /      │
                  │ Ledger feed  │──┘   └──────────────────┘   │ ledger_       │
                  └──────────────┘                             │ transactions  │
                                                               └───────┬───────┘
                                                                       │
  01:00 UTC cron / POST /api/v1/reconciliation/run                     ▼
  ─────────────────────────────────────────────▶  ┌──────────────────────────┐
                                                  │ RECONCILE (in memory)    │
                                                  │ match by trade_id        │
                                                  │ tolerance per currency   │
                                                  └────┬──────────────┬──────┘
                                                       │              │
                                          ┌────────────▼───┐   ┌──────▼─────────┐
                                          │ reconciliation │   │ Kafka          │
                                          │ _results       │   │ settlement.    │
                                          │ (exceptions)   │   │ mismatches     │
                                          └────┬───────────┘   └────────────────┘
                                               │
                    06:00 cron: SLA breach ────┴──▶ email to ops
                    Ops: GET /mismatches, POST /mismatches/{id}/acknowledge
```

### Phase A — Trigger

| # | Step | Where | Notes |
|---|------|-------|-------|
| 1 | Nightly cron at **01:00 UTC** | `ReconciliationScheduler.nightlyReconciliation()` | Runs ingestion first, *then* reconciliation |
| 2 | Manual trigger `POST /api/v1/reconciliation/ingest` and `/run` | `ReconciliationController` | `/ingest` is ADMIN only |
| 3 | Window is computed as `[today-1, today]` UTC | scheduler line 34–35 | Manual callers pass their own dates; `portfolio` is accepted but **ignored** |

### Phase B — Ingest (two identical pipelines)

| # | Step | Where |
|---|------|-------|
| 4 | Resolve source resources | `IngestionSourceProvider` → `ClasspathIngestionSourceProvider` (classpath CSV today; S3/Azure is a pluggable interface) |
| 5 | Read CSV, skip header, chunk of 500 | `SettlementBatchConfiguration` — fixed columns: `trade_id, account_id, currency, amount, trade_date, source_file` |
| 6 | Map text → object | `TransactionFieldSetMapper` (parses `BigDecimal` amount, ISO `LocalDate`) |
| 7 | Convert to entity, stamp `ingested_at` | `TradeTransactionItemProcessor` / `LedgerTransactionItemProcessor` |
| 8 | `repository.saveAll(chunk)` | plain inserts — **no dedupe, no upsert, no validation, no quarantine** |

### Phase C — Reconcile (all in memory)

| # | Step | Where | Detail |
|---|------|-------|--------|
| 9 | Load both sides for the date window | `findByTradeDateBetween(start, end)` | full table scan into heap |
| 10 | Index ledger by `trade_id` | `Collectors.toMap(..., (a,b) -> a)` | duplicate ledgers are **silently dropped** (first wins) |
| 11 | For each trade, pop its ledger from the map | `ledgerByTradeId.remove(...)` | 1:1 matching only |
| 12a | No ledger found | → `MISSING_LEDGER` | |
| 12b | Currency differs (case-insensitive) | → `CURRENCY_MISMATCH` | no FX conversion attempted |
| 12c | `abs(trade.amount - ledger.amount) > tolerance[currency]` | → `AMOUNT_DELTA` | tolerance from `application.yaml` (USD 0.5, GBP 0.3, EUR 0.4), missing currency = 0 |
| 12d | Otherwise | **matched — nothing is written** | no positive confirmation record |
| 13 | Ledger rows left in the map | → `MISSING_TRADE` | |
| 14 | Persist exception + publish Kafka event | `recordMismatch(...)` | one row + one event per exception, every single run |

### Phase D — After the run

| # | Step | Where |
|---|------|-------|
| 15 | Ops lists exceptions `GET /mismatches?status=UNRESOLVED` | `ReconciliationController` |
| 16 | Ops acknowledges `POST /mismatches/{id}/acknowledge` | sets `RESOLVED`, `acknowledged_by/at`, appends notes to the reason |
| 17 | 06:00 cron escalates UNRESOLVED older than SLA (24h) | `findStaleWithStatus` → `EmailNotificationService` |
| 18 | `GET /summary` aggregates **everything in the table**, not per run | recomputes counts from `findAll()` |

---

# 2. External data the settlement must "know"

Read this as: *what the service needs from outside itself to do its job correctly.*

### 2.1 What it already consumes

| Data | Comes from | Used for | Status |
|---|---|---|---|
| Trade feed | CSV on classpath (`samples/trades.csv`) | the "expected" side | ✅ but demo-only source |
| Ledger feed | CSV on classpath (`samples/ledger.csv`) | the "actual" side | ✅ but demo-only source |
| Tolerance per currency | `application.yaml` → `ReconciliationProperties` | amount comparison threshold | ✅ |
| Escalation SLA + email | `application.yaml` / env | SLA breach email | ✅ |
| Kafka topic name | `reconciliation.kafka.mismatch-topic` | event egress | ✅ |
| Operator identity | JWT `Principal` | `acknowledged_by` | ✅ |
| JWT public key / issuer | OAuth2 resource-server config | authN/authZ | ✅ (config placeholder) |

### 2.2 Data it *needs but does not have* — this is the important list

| # | External data | Why settlement needs it | Today |
|---|---|---|---|
| 1 | **Settlement date / value date** (not just trade date) | T+2 equities settle on a *different* day than the trade. Matching on `trade_date` is why real false `MISSING_TRADE`/`MISSING_LEDGER` appear | ❌ only `trade_date` |
| 2 | **Settlement amount vs traded amount** (FX, fees, accrued interest) | A trade in USD settling into a GBP ledger is not an error — it's expected. The service flags `CURRENCY_MISMATCH` instead of converting | ❌ no FX rate, no settlement ccy |
| 3 | **Direction / side (pay vs receive)** | Sign matters. `1000` vs `-1000` is a *huge* error, but `abs()` hides it completely | ❌ `abs()` on difference |
| 4 | **Quantity, price, instrument, counterparty, account/portfolio** | Real reconciliation matches on more than an ID; and `portfolio` filter is silently ignored | ❌ absent |
| 5 | **Trade version / amendment sequence** | Amended or cancelled trades create duplicate rows → phantom mismatches | ❌ no version, no `CANCELLED` handling |
| 6 | **Business date & holiday calendar** | "Yesterday" is not a business day on weekends/holidays. The cron will reconcile empty windows | ❌ `LocalDate.now(UTC)` |
| 7 | **Currency/tolerance master data** (from a reference-data service) | Tolerances in a YAML file can't be governed, versioned, or per-counterparty | ⚠️ static config |
| 8 | **Feed metadata: file name, sequence, checksum, record count, received-at** | Proof the file is complete and not a re-delivery — the basis of idempotency | ❌ only `source_file` string; no file-level ledger |
| 9 | **Feed status / cut-off confirmation** ("ledger extract final at 23:00") | Reconciling against a partially posted ledger produces noise | ❌ none |
| 10 | **Historical breaks & prior resolution state** | Re-runs should update a break, not create a new one | ❌ every run inserts new rows |
| 11 | **Run metadata: run id, trigger (cron/manual), operator, parameters** | Audit, reproducibility, "which run produced this break?" | ❌ absent |
| 12 | **Netting / settlement instruction grouping** | Multiple trades can settle as one instruction | ❌ 1:1 only |
| 13 | **Downstream consumers' contract** (Kafka schema registry) | Events must be versioned | ⚠️ JSON, no registry |

---

# 3. Data flow design

## 3.1 Design in one sentence

> Normalize both feeds into **one canonical transaction shape**, land them **idempotently**, create a **reconciliation run** that produces one **item per expected obligation** (matched *or* broken), and expose only the **breaks** to humans.

The key change from today: **a run is a first-class object**, and *matching produces a result for every key*, not only for failures.

### 3.2 Layered flow

```
 EXTERNAL                     INGEST / STAGE                 CORE                       EGRESS
 ────────                     ──────────────                 ────                       ──────
 Trade file/API ──┐
                  ├─▶ 1. feed_manifest      ─▶ 2. trade_instruction ──┐
 Ledger file/API ─┘     (file id, checksum,     ledger_posting       │
                         record count, status)                        ▼
                                                  3. reconciliation_run (window, params, status)
                                                              │
 Reference data ──▶ tolerances, calendar, FX, portfolio map ──┤
                                                              ▼
                                                  4. reconciliation_item  (MATCHED / BREAK, per key)
                                                              │
                                                     ┌────────┴────────┐
                                                     ▼                 ▼
                                          5. reconciliation_exception  6. Kafka mismatch event
                                             (open break, lifecycle)      (keyed by break id)
                                                     │
                                                     ▼
                                          7. escalation / acknowledge / audit
```

### 3.3 The four concepts that must be separated

| Concept | Question it answers | Why separate |
|---|---|---|
| **Instruction** | What *should* settle? | Trade side, versioned, may be T+2 |
| **Posting** | What *did* the ledger do? | Ledger side, immutable, may be partial or netted |
| **Run** | Which comparison produced this? | Reproducibility, re-runs, audit |
| **Exception** | What is still broken and who owns it? | Human workflow, SLA, escalation |

Today all four are collapsed into `reconciliation_results`, which is why re-runs duplicate work and the summary endpoint can't answer "how did last night go?".

## 3.4 Canonical inbound schema (both feeds)

One shape, two `record_type`s — this is what the CSV mapper should produce, and what an API or S3 connector would send.

```json
{
  "recordType": "TRADE",                 // TRADE | LEDGER
  "sourceSystem": "TRADING_PLATFORM",
  "businessDate": "2026-10-28",
  "feedId": "TRADES_20261028_001",

  "tradeId": "TRD-1001",
  "legNo": 1,                            // multi-leg support
  "version": 1,                          // amendment sequence
  "tradeStatus": "NEW",                  // NEW | AMENDED | CANCELLED

  "accountId": "ACC-1",
  "portfolioId": "PF-EQ-01",
  "counterpartyId": "CP-77",
  "instrumentId": "GB00B03MLX29",

  "side": "BUY",                         // BUY | SELL  → determines sign
  "quantity": "1000",
  "price": "1.0000",
  "tradeCurrency": "USD",
  "tradeAmount": "1000.00",

  "settlementCurrency": "USD",           // may differ from tradeCurrency
  "settlementAmount": "1000.00",         // after FX / fees / accrual
  "fxRate": "1.000000",

  "tradeDate": "2026-10-27",
  "settlementDate": "2026-10-29",        // THE matching date for settlement

  "sourceFile": "trades_20261028.csv",
  "sourceRowNumber": 42,
  "ingestedAt": "2026-10-28T18:03:11Z"
}
```

## 3.5 Feed manifest (completeness proof)

```sql
CREATE TABLE feed_manifest (
    id                BIGSERIAL PRIMARY KEY,
    feed_id           VARCHAR(128) NOT NULL,
    feed_type         VARCHAR(16)  NOT NULL,      -- TRADE | LEDGER
    source_system     VARCHAR(64)  NOT NULL,
    business_date     DATE         NOT NULL,
    file_name         VARCHAR(256),
    checksum_sha256   CHAR(64),
    expected_records  INTEGER,
    received_records  INTEGER,
    received_at       TIMESTAMPTZ  NOT NULL,
    status            VARCHAR(16)  NOT NULL,      -- RECEIVED | VALIDATED | REJECTED | SUPERSEDED
    UNIQUE (feed_id, feed_type)
);
```

## 3.6 Landing tables (idempotent)

```sql
CREATE TABLE trade_instruction (
    id                  BIGSERIAL PRIMARY KEY,
    trade_id            VARCHAR(64)  NOT NULL,
    leg_no              SMALLINT     NOT NULL DEFAULT 1,
    version             INTEGER      NOT NULL DEFAULT 1,
    trade_status        VARCHAR(16)  NOT NULL,     -- NEW | AMENDED | CANCELLED

    account_id          VARCHAR(64)  NOT NULL,
    portfolio_id        VARCHAR(64),
    counterparty_id     VARCHAR(64),
    instrument_id       VARCHAR(64),
    side                VARCHAR(8)   NOT NULL,     -- BUY | SELL

    trade_currency      CHAR(3)      NOT NULL,
    trade_amount        NUMERIC(19,4) NOT NULL,
    settlement_currency CHAR(3)      NOT NULL,
    settlement_amount   NUMERIC(19,4) NOT NULL,
    fx_rate             NUMERIC(19,8) NOT NULL DEFAULT 1,

    trade_date          DATE         NOT NULL,
    settlement_date     DATE         NOT NULL,     -- matched on THIS
    feed_id             VARCHAR(128) NOT NULL REFERENCES feed_manifest(feed_id),
    ingested_at         TIMESTAMPTZ  NOT NULL DEFAULT NOW(),

    -- idempotency: same file re-delivered = same row
    UNIQUE (trade_id, leg_no, version)
);
CREATE INDEX idx_trade_instruction_settle ON trade_instruction (settlement_date, trade_id);

CREATE TABLE ledger_posting (
    id                  BIGSERIAL PRIMARY KEY,
    posting_id          VARCHAR(128) NOT NULL,     -- ledger's own unique id
    trade_id            VARCHAR(64),
    account_id          VARCHAR(64)  NOT NULL,
    side                VARCHAR(8)   NOT NULL,     -- DEBIT | CREDIT
    currency            CHAR(3)      NOT NULL,
    amount              NUMERIC(19,4) NOT NULL,    -- signed, not abs()
    value_date          DATE         NOT NULL,
    posting_status      VARCHAR(16)  NOT NULL,     -- POSTED | REVERSED | PENDING
    feed_id             VARCHAR(128) NOT NULL REFERENCES feed_manifest(feed_id),
    ingested_at         TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE (posting_id)                            -- re-delivery safe
);
CREATE INDEX idx_ledger_posting_trade ON ledger_posting (trade_id);
CREATE INDEX idx_ledger_posting_value ON ledger_posting (value_date);
```

## 3.7 Run + item (the missing middle)

```sql
CREATE TABLE reconciliation_run (
    id              BIGSERIAL PRIMARY KEY,
    run_reference   VARCHAR(64)  NOT NULL UNIQUE,
    business_date   DATE         NOT NULL,
    window_start    DATE         NOT NULL,
    window_end      DATE         NOT NULL,
    portfolio_id    VARCHAR(64),                  -- actually honoured
    trigger_type    VARCHAR(16)  NOT NULL,        -- SCHEDULED | MANUAL
    triggered_by    VARCHAR(128),
    status          VARCHAR(16)  NOT NULL,        -- RUNNING | COMPLETED | FAILED
    started_at      TIMESTAMPTZ  NOT NULL,
    finished_at     TIMESTAMPTZ,
    matched_count   INTEGER      NOT NULL DEFAULT 0,
    break_count     INTEGER      NOT NULL DEFAULT 0
);

CREATE TABLE reconciliation_item (
    id                  BIGSERIAL PRIMARY KEY,
    run_id              BIGINT      NOT NULL REFERENCES reconciliation_run(id),
    match_key           VARCHAR(160) NOT NULL,    -- trade_id|leg_no
    trade_id            VARCHAR(64),
    ledger_posting_id   BIGINT      REFERENCES ledger_posting(id),
    outcome             VARCHAR(16) NOT NULL,     -- MATCHED | BREAK
    break_type          VARCHAR(32),              -- see below
    trade_amount        NUMERIC(19,4),
    ledger_amount       NUMERIC(19,4),
    difference          NUMERIC(19,4),
    tolerance_applied   NUMERIC(19,4),
    reason              TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (run_id, match_key)
);
```

## 3.8 Exception (what ops actually work on)

```sql
CREATE TABLE reconciliation_exception (
    id                  BIGSERIAL PRIMARY KEY,
    first_seen_run_id   BIGINT      NOT NULL REFERENCES reconciliation_run(id),
    last_seen_run_id    BIGINT      REFERENCES reconciliation_run(id),

    trade_id            VARCHAR(64),
    account_id          VARCHAR(64),
    portfolio_id        VARCHAR(64),
    currency            CHAR(3),
    trade_amount        NUMERIC(19,4),
    ledger_amount       NUMERIC(19,4),

    break_type          VARCHAR(32) NOT NULL,
    -- AMOUNT_DELTA | MISSING_LEDGER | MISSING_TRADE | CURRENCY_MISMATCH
    -- + DATE_MISMATCH | SIDE_MISMATCH | QUANTITY_MISMATCH | DUPLICATE_POSTING

    status              VARCHAR(16) NOT NULL DEFAULT 'OPEN',
    -- OPEN | INVESTIGATING | RESOLVED | WRITTEN_OFF

    owner               VARCHAR(128),
    severity            VARCHAR(8),
    reason              TEXT,

    first_seen_at       TIMESTAMPTZ NOT NULL,
    last_seen_at        TIMESTAMPTZ NOT NULL,
    sla_due_at          TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,
    resolved_by         VARCHAR(128),
    resolution_notes    TEXT,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- one OPEN break per trade, ever
CREATE UNIQUE INDEX ux_exception_open
    ON reconciliation_exception (trade_id, break_type)
    WHERE status <> 'RESOLVED';

CREATE TABLE reconciliation_exception_event (   -- audit trail
    id              BIGSERIAL PRIMARY KEY,
    exception_id    BIGINT      NOT NULL REFERENCES reconciliation_exception(id),
    action          VARCHAR(32) NOT NULL,        -- OPENED | REOPENED | ACKNOWLEDGED | ESCALATED
    actor           VARCHAR(128),
    notes           TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 3.9 Kafka event (versioned)

```json
{
  "eventId": "9f1c...",
  "eventType": "reconciliation.break.opened",
  "eventVersion": 2,
  "occurredAt": "2026-10-29T01:04:12Z",
  "runReference": "RECON-2026-10-29-001",
  "exceptionId": 4412,
  "tradeId": "TRD-1003",
  "matchKey": "TRD-1003|1",
  "breakType": "MISSING_LEDGER",
  "status": "OPEN",
  "tradeAmount": { "amount": "1500.0000", "currency": "EUR" },
  "ledgerAmount": null,
  "severity": "HIGH",
  "slaDueAt": "2026-10-30T01:04:12Z",
  "reason": "No ledger posting for settlement date 2026-10-29"
}
```

## 3.10 Mapping: today's schema → proposed

| Today | Becomes |
|---|---|
| `trade_transactions` | `trade_instruction` (adds version, side, settlement date/amount, feed link) |
| `ledger_transactions` | `ledger_posting` (adds `posting_id`, signed amount, value date, status) |
| `reconciliation_results` | split into `reconciliation_item` (one per key per run) + `reconciliation_exception` (open break) + `reconciliation_exception_event` (audit) |
| (none) | `feed_manifest`, `reconciliation_run` |
| `MismatchEvent` | versioned event keyed by `exceptionId`, not `tradeId` |

---

# 4. Plain-English walkthrough (with the sample data)

Use the analogy of **checking your bank statement against your receipts**.

Your **receipts** = trades (what you think you spent). Your **statement** = ledger (what the bank actually booked). Reconciliation = line them up by reference and complain about the differences.

Running the actual sample files for `2024-06-01`:

| Trade | Ledger | Outcome | Why |
|---|---|---|---|
| TRD-1001 USD 1000.00 | TRD-1001 USD 1000.25 | ✅ matched | diff 0.25 ≤ USD tolerance 0.50 — *not stored anywhere* |
| TRD-1002 GBP 2000.00 | TRD-1002 GBP 2000.00 | ✅ matched | exact |
| TRD-1003 EUR 1500.00 | *(none on 06-01)* | 🔴 `MISSING_LEDGER` | trade exists, no posting |
| *(none)* | TRD-9999 USD 900.00 | 🔴 `MISSING_TRADE` | posting exists, no trade |

So 6 records read → 2 clean, 2 breaks, and `totalProcessed = 6` (trades + ledger rows, not "instructions reconciled").

Then:

1. Two rows in `reconciliation_results`, both `UNRESOLVED`.
2. Two Kafka messages to `settlement.mismatches`.
3. Ops opens `/mismatches?status=UNRESOLVED`, works TRD-1003, calls `/mismatches/{id}/acknowledge` → `RESOLVED`, their JWT name recorded, notes appended to `mismatch_reason`.
4. If nobody acts within 24h, the 06:00 cron emails `ops-team@example.com` with the trade IDs.
5. **Tonight's re-run inserts two brand-new rows again** — the old acknowledged ones stay, and `/summary` double-counts.

**In one line:** *the service proves the ledger agrees with the trades; it does not move money, and it does not yet know what the ledger is supposed to agree with.*

---

# 5. Gaps worth fixing (ranked)

| Priority | Gap | Consequence |
|---|---|---|
| 🔴 1 | No run identity → every run duplicates breaks | Cannot re-run, cannot audit, summary inflates |
| 🔴 2 | Matches on `trade_date` not `settlement_date` | Systematic false breaks on T+2 and FX |
| 🔴 3 | `abs()` on amount, no side/sign | A reversed posting looks perfect |
| 🔴 4 | Matching is 1:1 by `trade_id` with "first wins" | Duplicate postings silently disappear; netting impossible |
| 🔴 5 | `portfolio` parameter silently ignored | Users believe they filtered; they didn't |
| 🟠 6 | No per-feed completeness check | Partial ledger ingest → mass false `MISSING_LEDGER` |
| 🟠 7 | Kafka published *inside* `@Transactional` | Ghost events if the transaction rolls back |
| 🟠 8 | No `CANCELLED`/amended trade handling | Phantom instructions |
| 🟠 9 | Currency mismatch instead of FX conversion | Expected cross-currency settlements flagged as breaks |
| 🟡 10 | Tolerance in YAML, no currency default | Unlisted currency silently gets zero tolerance (any 1-cent diff breaks) |
| 🟡 11 | Matched records not persisted | No evidence of successful reconciliation |

---

# 6. Suggested next step

Three implementation slices, in order:

1. **Make runs real** — add `reconciliation_run` + `reconciliation_item`, persist matched *and* broken items, and make re-runs idempotent.
2. **Fix the matching key** — introduce `settlement_date`, `side`, `settlement_currency/amount`, `version`, and match on `(trade_id, leg_no)` with a business-date window; honour `portfolio`.
3. **Make breaks a workflow** — separate `reconciliation_exception` with `OPEN → INVESTIGATING → RESOLVED`, dedupe by partial unique index, emit versioned events *after commit*.

Want me to turn any of this into a concrete plan — e.g. a Flyway migration + entity refactor for slice 1, or a written design doc? If so, tell me which slice and I'll write it into the plan directory.
