# Slotbook: Design & Reliability Report

## 1. Links

- **GitHub repo:** `<PASTE YOUR PUBLIC REPO URL HERE>`
- **Live URL:** none (not deployed; runs with `docker compose up`)

Stack: TypeScript, Fastify, plain SQL on Postgres 16 (`pg` driver), vanilla HTML/JS front end. No Redis or external lock service.

## 2. Architecture

```
 Browser (guest)             Browser (host)
 /{host}/{slug}, /manage/..  /host
        \                        /
         +-------- JSON API -----+
                     |
         Fastify (src/server.ts)   validation, auth, error mapping
                     |
         Booking service (src/service.ts)
           - slot generation (src/slots.ts, pure function)
           - capacity / buffer check
           - idempotency, outbox
                     |
         Postgres (single source of truth)
           hosts, event_types, availability_*, bookings,
           idempotency_keys, notifications (outbox)
```

**How a booking request flows (hold):**

1. `POST /api/holds` with an `Idempotency-Key`. The route validates input, then starts one DB transaction.
2. The transaction takes a row lock on the host: `SELECT … FROM hosts WHERE id=$1 FOR UPDATE`. All writes for that host now run one at a time.
3. It looks up the idempotency key. If it exists, the stored response is returned.
4. It regenerates the host's slots for that instant (rules, overrides, notice, horizon) and subtracts active bookings. Time comes from the **database clock**.
5. If the slot is valid and has capacity, it inserts a `held` booking (`hold_expires_at = now() + 5 min`), stores the idempotency response, and commits.

Confirm, reschedule and cancel use the same pattern (lock host → re-read the row → change state → write a notification row in the same transaction).

## 3. Data model

| Table | Key columns | Constraints that matter |
|---|---|---|
| `hosts` | `id`, `username`, `timezone`, `password_hash` | `username` unique and format-checked |
| `event_types` | `host_id`, `slug`, `duration_min`, `capacity`, `buffer_before/after_min`, `min_notice_min`, `horizon_days` | `unique(host_id, slug)`; CHECKs: duration 15–120, buffers 0–30, capacity ≥ 1 |
| `availability_rules` | `host_id`, `weekday` (0=Sun), `start_time`, `end_time` | `end_time > start_time`; several rows per weekday allowed |
| `availability_overrides` | `host_id`, `on_date`, `is_off`, `start_time`, `end_time` | day-off rows have no times; custom-hours rows need `end > start` |
| `bookings` | `event_type_id`, `host_id`, `start_at`, `end_at`, **`occ_start`, `occ_end`**, `status`, `hold_expires_at`, `guest_*`, `manage_token` | `status` in (held, confirmed, cancelled, expired); held ⇒ has expiry; confirmed ⇒ has guest; `manage_token` unique and random; partial index on active rows |
| `idempotency_keys` | `key` (PK), `request_hash`, `response` | PK makes a key unique; hash detects reuse with a different request |
| `notifications` | `type`, `booking_id`, `payload`, `delivered_at` | outbox; written in the same transaction as the change |

`occ_start`/`occ_end` is the buffered busy interval stored per booking. It makes cross-event-type conflict detection a plain interval-overlap check.

**Capacity rule:** an *active* booking is `confirmed`, or `held` with `hold_expires_at > now()`. Expired and cancelled bookings never count.

## 4. Key decisions

**1. Serialize writes per host with a row lock** (`SELECT … FOR UPDATE` on `hosts`).
*Alternatives:* a Postgres exclusion constraint on time ranges; `SERIALIZABLE` isolation with retries; advisory locks; optimistic version counters.
*Why:* capacity > 1 (group sessions) and buffers across different event types do not fit a plain exclusion constraint, and `SERIALIZABLE` needs retry loops everywhere. One lock per host is simple to reason about and throughput per host is not a concern for a scheduling app. Different hosts do not contend. *Trade-off:* a hot host is a serial queue.

**2. Expiry is decided by the database clock at read time; the sweeper is only housekeeping.**
*Alternatives:* a background job as the only expirer; app-server timestamps.
*Why:* a delayed or crashed sweeper, or clock skew between app servers, then cannot keep a dead hold alive or free a live one.

**3. Buffers stored on the row (`occ_*`) instead of recomputed from the event type.**
*Alternative:* join to `event_types` and compute at query time.
*Why:* editing an event type's buffers later must not retroactively change existing bookings; and overlap checks become one indexed predicate.

**4. Idempotency keys stored with a request hash and the original response, checked after taking the host lock.**
*Alternatives:* in-memory dedupe; unique constraint only.
*Why:* survives restarts, and two parallel retries serialize on the lock so the second always sees the first's result. Same key with a different payload returns 422 instead of silently doing something else. Failed attempts are not stored, so a client can retry them.

**5. Transactional outbox for notifications.**
*Alternative:* call a webhook inside the request.
*Why:* an event exists if and only if the state change committed; a slow or dead receiver cannot fail or delay a booking. A separate worker can deliver rows later (not built).

**6. Slots stepped in absolute time from window start; windows defined in host-local wall time.**
*Alternative:* step in local wall-clock time.
*Why:* DST days have the correct real length (see section 6).

**7. Plain SQL + `pg` instead of an ORM.**
*Why:* the interesting part is locking and constraints; raw SQL keeps it explicit and reviewable.

## 5. Reliability

| Failure case | How it is handled | Test (`test/booking.test.ts`) |
|---|---|---|
| Many guests click the same slot at once | host row lock → exactly one insert succeeds, the rest get `409` | "25 simultaneous holds…exactly one wins" |
| Group session over-sold | capacity counted under the same lock | "capacity 3: 12 simultaneous holds → exactly 3" |
| Two different event types overlap (incl. buffers) | stored `occ_*` interval overlap check under the lock | "overlapping times across DIFFERENT event types" |
| Network retry / double-click on hold | `Idempotency-Key`; same key returns the same hold | "same Idempotency-Key (even concurrently)" |
| Key reused for a different request | `422 idempotency_key_reuse` | "same key with a different request" |
| Missing key | `400` | "missing Idempotency-Key" |
| Double-click / retry on confirm | confirm is state-based; second call returns the same token, one notification | "double-clicking confirm is safe" |
| Hold expires; sweeper late or dead | DB-clock check on every read and write; confirm → `410` | "expired hold frees the slot immediately" |
| Guest cancels twice / in parallel | cancel is idempotent; one event only | "cancel frees the slot, is idempotent" |
| Reschedule target taken | whole reschedule is one transaction; on `409` the original booking is untouched | "reschedule into a taken slot fails…" |
| Two reschedules race for one slot | same lock → one wins | "two guests racing to reschedule" |
| Forged / off-grid / out-of-window start time | start must equal a generated slot (rules, overrides, notice, horizon) | "rejects a start time that is not a generated slot" |
| Notification lost if request crashes | outbox row commits with the change | covered by design; asserted via notification counts in the tests above |
| DST transitions | absolute-time stepping; gap and overlap handled | `test/slots.test.ts` |

## 6. Time zones

- Hosts store an **IANA zone**. Weekly rules and overrides are **host-local wall-clock** times (e.g. 09:00–17:00 Mon–Fri).
- For each host-local date in the requested range, the windows are converted to absolute instants (UTC). Slots step by the event duration **in absolute time** from the window start, and are filtered by minimum notice and booking horizon. The API returns UTC instants; the browser formats them in the guest's chosen zone (default: browser zone, with a switcher).
- Local → UTC rule: a time that does not exist (spring-forward gap) is shifted forward; an ambiguous time (fall-back overlap) uses the earlier occurrence.

**Worked example: America/New_York, Sunday 2026-03-08 (clocks jump 02:00 → 03:00).** Host window 01:00–04:00, 60-min event.
- Window start `01:00 EST` = `06:00Z`. Window end `04:00 EDT` = `08:00Z`. Real length: **2 hours**, not 3.
- Slots: `06:00Z` (01:00 EST) and `07:00Z` (03:00 EDT). A slot at "02:00" is never offered because it does not exist.
- Same host on 2026-11-01 (clocks fall back), window 00:00–03:00: `04:00Z` to `08:00Z` = **4 real hours**, so 4 slots (04:00Z, 05:00Z, 06:00Z, 07:00Z) are offered, including the repeated 01:00 hour.
- A guest in Asia/Kolkata (UTC+5:30, no DST) sees the March example as 11:30 and 12:30 IST on that same Sunday (`06:00Z` and `07:00Z`), because only the host's side has a DST jump.

Both examples are asserted in `test/slots.test.ts`.

## 7. Optional extensions

None built. Time was spent on the core and its reliability. The outbox table is the natural hook for webhooks, reminders and `.ics` delivery; idempotency and the host lock already protect a waitlist or daily-cap check, which would run under the same lock.

## 8. Testing

- `test/slots.test.ts` (9 tests, no database): Kolkata offset, spring-forward and fall-back days, multiple windows, day-off and custom-hours overrides, window-end clipping, minimum notice and horizon. **These were executed during development and pass.**
- `test/booking.test.ts` (14 tests, real Postgres): concurrency, capacity, buffers across event types, idempotency, expiry, confirm/cancel/reschedule, validation.
- Run: `docker compose up -d db` then `DATABASE_URL=… npm test`.
- **Results to fill in after you run the suite:** `<PASTE test output here>`. The Postgres-backed tests were written but not executed in the environment where the code was drafted (no database available), so run them before submitting.
- Load test and Playwright tests: not built.

## 9. Using AI

Claude drafted the schema, slot engine, service, API, UI and tests from the brief. It helped most with enumerating failure cases and writing the concurrency tests. It misled in small ways that review caught: tests that ignored how buffers make adjacent slots conflict, and a test time that was off the other event type's grid. See `AI_LOG.md` for details and my own edits. `<EDIT: add your real experience>`

## 10. Known gaps and next steps

**Gaps**
- Postgres-backed tests and the running app still need a first run by the author; expect small fixes.
- No rate limiting or bot protection: a bot could hold every slot for 5 minutes at a time.
- No notification delivery worker; the sink is a table.
- Host auth is a seeded password and a signed token (no sign-up, no password change).
- Plain UI; overnight availability windows not supported; no browser tests.
- Host row lock serializes a single host's writes (fine at this scale).

**With another day**
1. Run and fix: first end-to-end run and test pass on real Postgres.
2. Abuse protection: per-IP/per-email hold limits and a cap on active holds.
3. Outbox worker with retries, then host webhooks (HMAC) and `.ics` attachments.
4. Playwright guest flow and a GitHub Actions workflow; a k6 script for the booking path.
5. Waitlist with time-limited offers under the same host lock.
