# API Key, Quota, Webhook, and Billing Architecture

## Scope

This document explains the platform behavior for API-key security, request quota governance, outbound webhook reliability, and deferred usage billing.

---

## 1) System Layers

The platform is organized into four cooperating layers:

- **Admission Layer**: validates identity and whether a request is allowed now.
- **Metering Layer**: captures usage and request activity.
- **Settlement Layer**: computes monetary cost and charges in batches.
- **Delivery Layer**: sends outbound callbacks with retry guarantees.

This separation keeps the request path fast while preserving policy and financial control.

---

## 2) Core Domain Objects

### Credential domain

- A credential record stores key value, owner, active status, expiry, and last-used timestamp.
- A settings record stores network and delivery constraints:
  - allowlisted IPs
  - callback URL
  - server-only usage policy

### Quota domain

- A quota policy defines:
  - request limit
  - period window (daily, weekly, monthly, yearly, custom)
  - enforcement mode (hard stop vs soft warning)
- A quota assignment links credential to policy.
- A quota usage record stores counters for a concrete period boundary.

### Billing domain

- Pricing rules map request volume to charge amount.
- Daily state tracks:
  - total observed requests
  - billed requests
  - billed amount
  - adaptive flush timing
  - block status after payment failure

### Webhook domain

- Outbound delivery logs store payload snapshots, attempt metadata, status transitions, retry schedule, and response/error details.

---

## 3) Request Admission Flow

For API-key requests:

1. Validate key format.
2. Resolve key owner/profile data (cache-first, DB fallback).
3. Enforce credential state rules:
  - key active
  - owner active
  - not expired
  - server-only policy
  - IP allowlist
4. Evaluate quota state. For **hard** quota enforcement, usage is incremented on the admission path so concurrent requests cannot overshoot before metering runs.
5. Evaluate billing block state.
6. Attach authenticated context to request.

For mixed-auth endpoints:

- If API key is provided, API-key admission logic is used.
- Otherwise bearer-token logic is used.

---

## 4) Caching Strategy

Two major cache surfaces improve throughput:

- **Credential cache**
  - hashed cache key
  - long TTL
  - invalidated on credential mutation
- **Quota decision cache**
  - per-credential cache key
  - short TTL
  - invalidated after usage increment

This reduces repetitive policy reads during traffic bursts.

---

## 5) Quota Engine

### Decision path

1. Load all active quota assignments for the credential.
2. Ensure current-period usage record exists per assignment.
3. Compute usage, remaining, reset time, and exceeded flag.
4. Select the most restrictive assignment outcome.
5. Cache result briefly.

### Enforcement semantics

- **Soft mode**: requests are still allowed even when exceeded, but exceeded metadata is surfaced.
- **Hard mode**: requests are denied once limit is reached.

### Counter updates

Usage counters are incremented asynchronously through metering side effects, then quota cache is invalidated.

### Safety posture

On internal quota-evaluation failure, the system defaults to conservative denial rather than accidental over-permit.

---

## 6) Deferred Billing Model

Billing is batched, not charged inline per request.

### Daily accumulator state

Per owner/day storage keeps:

- total usage count
- billed usage count
- billed amount
- rate estimate (for adaptive scheduling)
- next/last flush timestamps
- block flag and failure markers

### Adaptive flush scheduling

Flush interval is derived from request-rate estimate:

- higher traffic -> shorter interval
- lower traffic -> longer interval

A jitter factor randomizes timing to prevent synchronized flush storms.

### Flush enqueue controls

Flush jobs are enqueued only when:

- there is positive outstanding usage
- account is not currently blocked
- scheduled flush time arrived or outstanding count crossed threshold

A short-lived distributed lock suppresses duplicate enqueue attempts.

---

## 7) Billing Enforcement Gate

If settlement previously failed due to low wallet balance, the account is marked blocked.

On later API-key admission:

1. Current debt is recomputed.
2. Wallet affordability is checked.
3. If balance now covers debt, block is removed.
4. Otherwise request is rejected with payment-required response.

This acts as a hard financial circuit breaker.

**Wallet selection:** Admission-time debt checks consult the owner’s wallet. Asynchronous settlement debits the **merchant** wallet. Keep both funded (or align product rules) so payment blocks and actual charges stay consistent.

---

## 8) Settlement Worker Semantics

Background settlement worker flow:

1. Acquire distributed lock for owner/day segment.
2. Load billing state.
3. Compute differential charge amount:
  - total cost at current usage minus already billed amount
4. If differential <= 0:
  - normalize counters and exit.
5. If differential > 0:
  - run wallet debit + ledger write in a DB transaction.
6. Persist updated billed counters/amount.
7. If insufficient balance:
  - mark blocked state and failure timestamp.
8. Release lock in all outcomes.

Key guarantees:

- atomic money movement and ledgering
- differential charging avoids double charge
- repeated flushes converge to fully billed state

---

## 9) Pricing Behavior

Cost for a request volume snapshot is computed with priority:

1. exact-volume rule
2. matching range rule
3. overage-capable fallback rule

Final amount = base amount + overage component (if applicable).

Because settlement compares recomputed total against already billed amount, the system self-corrects drift over time.

---

## 10) Outbound Webhook Reliability

### Delivery lifecycle

1. Business event enqueues callback delivery job.
2. Worker validates destination and key state.
3. Delivery attempt is logged persistently.
4. HTTP callback is sent with:
  - strict timeout
  - signed payload header (`x-waysdrop-signature`)
  - stable delivery idempotency header (`X-Idempotency-Key`, format `{webhookLogId}:{attemptNumber}`)
  - log correlation header (`X-Webhook-Log-Id`)
5. Success updates terminal delivered state.
6. Failure updates retry metadata and status.

### Integrator notes: idempotency

Each HTTP delivery attempt uses a **new** `X-Idempotency-Key` (`{webhook_log_id}:{attempt}`). The same logical event may legitimately produce multiple keys across retries; treat the key as idempotent for **that single HTTP POST only**. Use `X-Webhook-Log-Id` to correlate retries of the same log row.

### Retry policy

Retry is allowed for:

- timeout and transport failures
- throttling/timeouts at HTTP layer
- server-side response failures

Retry is generally skipped for non-retryable client errors.

### Retry scheduling

On failure, the worker persists `nextRetryAt` (exponential backoff) and enqueues a **single** Bull `retry-webhook` job with `delay` set to that timestamp. There is no separate repeat/poll retry loop.

### Terminal outcomes

- Delivered
- Failed (non-retryable)
- Exhausted (max-attempts reached)

Manual replay of failed/exhausted deliveries is supported with state checks.

---

## 11) Metering Side Effects and Consistency

After request completion, asynchronous metering performs:

- usage counter increment (skipped when quota was pre-incremented on the guard path for **hard** enforcement)
- key last-used update
- billing accumulator update

A configurable de-duplication window reduces repeated metering writes for near-identical bursts:

- `**API_KEY_USAGE_DEDUP_TTL_SEC`** (default `30`, minimum `1`): bucket width in seconds for the metering cache key.
- `**API_KEY_USAGE_DEDUP_BODY_DIGEST=1**`: optional; includes a short SHA-256 digest of the logged request body in the de-dup signature so distinct POST bodies are metered separately within the same window.

Tradeoff:

- better p99 latency and throughput
- slight lag before counters/billing fully reflect latest request
- shorter TTL or body digest reduces undercounting of intentional repeated identical routes at the cost of more writes

### Prometheus metrics

- `**waysdrop_billing_blocked_total{where}**` — `where=flush` when a settlement flush hits insufficient wallet; `where=gate` when admission rejects with HTTP 402 while billing is blocked.
- `**waysdrop_billing_flush_lock_miss_total{kind}**` — `kind=flush` or `kind=enqueue` when the Redis lock for flush/enqueue is not acquired (contention).
- `**waysdrop_webhook_retry_lag_seconds**` — histogram: seconds from webhook log creation to a retry HTTP attempt.

---

## 12) Scalability Characteristics

### Strengths

- Queue decoupling isolates expensive side effects from request path.
- Cache layers reduce repeated policy reads.
- Adaptive flush intervals reduce write amplification.
- Locking prevents duplicate settlement execution under concurrency.

### Pressure points

- Metering still performs frequent storage/DB writes at high RPS.
- Short quota-cache windows can create brief staleness.
- Signature-window de-dup may undercount intentional repeated identical hits; tune `API_KEY_USAGE_DEDUP_TTL_SEC` / enable body digest if needed.

### High-load behavior

- Admission remains strict and synchronous.
- Metering/settlement are eventually consistent but bounded.
- Financial leakage is constrained by blocked-state gate.

---

## 13) Failure Modes and Guards

- Invalid/inactive/expired credential -> auth rejection.
- Hard quota exceed -> request rejection with reset metadata.
- Settlement insufficient funds -> blocked state.
- Blocked state at admission -> payment-required rejection.
- Callback destination instability -> persistent retries then exhaustion.
- Duplicate worker triggers -> suppressed via distributed locks.

---

## 14) Operations Checklist

Monitor:

- auth rejection rates by category
- quota exceed rates and reset horizons
- blocked account count and age
- settlement queue lag and lock contention
- callback success/retry/failure/exhausted ratios
- retry backlog age
- wallet-vs-ledger consistency

Inspect storage namespaces for:

- credential cache
- quota cache
- billing state
- lock artifacts

Inspect queue health for:

- settlement jobs
- callback send jobs
- callback retry jobs

---

## 15) Billing: traffic smoothing, timing, and money

Usage billing is **rate-aware** and **ledger-safe**:

1. **Smoothed request rate** — Each request updates an estimate of how fast traffic is arriving. Very short gaps between calls imply a high instantaneous rate; the estimate blends each new sample with history—mostly history, a little fresh signal—so isolated spikes do not swing scheduling wildly while sustained load still pulls the estimate up.
2. **Adaptive flush spacing** — The wait time before the next settlement attempt is chosen from a **stepped schedule** keyed off that smoothed rate: busier keys reconcile more often, quieter keys batch longer. That bounds how far wallet state can drift from metered usage without hammering the database on every call.
3. **Randomized spacing** — A bounded random offset is applied around each scheduling interval so tenants with similar patterns do not all flush at the same wall-clock instant, which reduces lock fights and database hot spots.
4. **Decimal money** — Amounts owed, already billed, and incremental charges stay in a decimal representation end-to-end so binary floating-point cannot invent or lose small currency units. What you still owe is modeled as the non-negative gap between "priced usage so far" and "already settled."
5. **Metering fingerprints** — Voluntary de-duplication (see §11) uses short cryptographic fingerprints of route identity and, optionally, request body material so legitimate repeated hits can still be distinguished when payloads differ.

Together, smoothing and jitter keep the async billing path predictable under bursts; decimal handling keeps financial aggregates trustworthy for admission gates and settlement workers.

---

## 16) End-to-End Summary

1. Request enters admission checks.
2. Policy decides allow/deny.
3. Business operation executes.
4. Async metering records usage and billing signals.
5. Settlement worker charges accrued debt in batches.
6. Failed charge marks block, which is enforced at future admission.
7. Outbound callbacks are delivered asynchronously with durable retries.

Overall, the architecture balances performance, policy correctness, and financial safety by keeping strict decisions synchronous and heavy operations asynchronous.
