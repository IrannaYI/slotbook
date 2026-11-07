# Slotbook

A small Calendly-style booking engine: hosts publish availability, guests hold → confirm slots, and nobody is double-booked, even with retries, expiring holds, or concurrent requests. TypeScript end to end (Fastify + plain SQL on Postgres, no ORM, no Redis).

No starter template or boilerplate was used.

## Run it

```bash
docker compose up --build        # app on http://localhost:3000, migrations + seed run on start
```

Or without Docker for the app: `docker compose up -d db && npm install && npm run dev`.

Seeded demo host: **`demo` / `demo1234`**, time zone `Asia/Kolkata`, Mon–Fri 09:00–12:00 and 13:00–17:00, with two event types:

| Slug | What |
|---|---|
| `/demo/intro-call` | 30 min, one-on-one, 10 min buffer after, 4 h notice |
| `/demo/group-workshop` | 60 min, **5 seats**, 10 min buffers, 24 h notice |

## Tests

Tests run against a **real Postgres** (they truncate the tables, so use a throwaway database).

```bash
docker compose up -d db
DATABASE_URL=postgres://slotbook:slotbook@localhost:5432/slotbook npm test
```

`test/slots.test.ts` needs no database (time zones, DST, overrides). `test/booking.test.ts` is the concurrency / idempotency / expiry / reschedule suite.

## 60-second tour

1. Open `/host`, sign in (`demo` / `demo1234`). Change the time zone, create an event type, edit weekly availability and date overrides.
2. Open `/demo/intro-call` in another tab. Switch the time-zone dropdown and watch the slots move. Pick one: it is **held for 5 minutes** (countdown shown).
3. Enter a name and email and confirm. You get a private **manage link** (`/manage/<token>`). Reschedule or cancel from there.
4. Back on `/host`, the booking appears under *Upcoming bookings*; cancel it with a reason.
5. Every confirm / reschedule / cancel writes a row to the `notifications` table: `select type, payload from notifications;`

## API

All JSON. Errors look like `{ "error": "slot_unavailable", "message": "..." }`.

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/hosts/{host}/events/{slug}/slots?from&to&tz` | Available slots. `from`/`to` are `YYYY-MM-DD` (interpreted in `tz`) or ISO instants; default next 30 days, max 62. Returns UTC ISO `start`/`end` and `remaining` seats. |
| POST | `/api/holds` | Body `{host, slug, start}`. **Requires `Idempotency-Key` header.** `201 {holdId, expiresAt}`; `409 slot_unavailable`; `422 invalid_slot`. |
| POST | `/api/holds/{id}/confirm` | Body `{name, email}` → `{manageToken, booking}`. `410 hold_expired`. Safe to retry. |
| GET | `/api/bookings/{token}` | View a booking. |
| POST | `/api/bookings/{token}/reschedule` | Body `{start}`. Atomic move; `409` if taken. |
| POST | `/api/bookings/{token}/cancel` | Body `{reason?}`. Idempotent. |
| POST | `/api/host/login` | `{username, password}` → bearer token (12 h, HMAC-signed). |
| GET/PUT | `/api/host/me` | Read / set home time zone (IANA). |
| GET/POST/PUT | `/api/host/event-types[/{id}]` | Manage event types. |
| GET/PUT | `/api/host/availability` | `{rules:[{weekday,start,end}], overrides:[{date,off,start,end}]}` (replace-all). |
| GET | `/api/host/bookings` | Upcoming confirmed bookings. |
| POST | `/api/host/bookings/{id}/cancel` | `{reason}` (required). |
| GET | `/health` | Liveness + DB check. |

## How correctness works (short version)

- **One lock per host.** Every write that can change capacity first runs `SELECT … FROM hosts WHERE id = $1 FOR UPDATE`. Within that lock the code recomputes availability and inserts. Concurrent requests for one host queue up; different hosts never block each other.
- **Postgres decides what is "active".** A hold counts only while `hold_expires_at > now()` (database clock). The sweeper that marks rows `expired` is housekeeping; correctness does not depend on it.
- **Buffers are stored on the row** (`occ_start`/`occ_end`), so conflict checks across different event types are a simple interval overlap. Bookings of the *same* event type at the *same* start share capacity (group sessions).
- **Idempotency**: holds use an `Idempotency-Key` stored with a request hash and the original response (checked *after* the host lock, so parallel retries serialize). Confirm / cancel / reschedule are idempotent by state.
- **Notifications are an outbox**: inserted in the same transaction as the state change, so an event exists if and only if the change committed.

## Assumptions

- Slots are stepped by the event duration inside each availability window (a 60-min event in 09:00–17:00 gives 09:00, 10:00, …).
- Buffers make the host busy for `[start − buffer_before, end + buffer_after]`; other bookings may not overlap that range.
- "Held" bookings are only visible to the holder; the host dashboard lists confirmed bookings.
- Guests are not authenticated; the 64-hex-char manage token is the credential.
- Overnight windows (crossing midnight) are not supported; split them across two days.
- A nonexistent local time (DST gap) is skipped; an ambiguous one (DST overlap) uses the earlier occurrence, and slots are stepped in absolute time so every real hour is offered.

## Not built (see report)

Optional extensions, auth beyond the seeded password login, rate limiting, real email, and browser (Playwright) tests.
