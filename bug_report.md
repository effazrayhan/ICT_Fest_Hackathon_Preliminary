# CoWork API — Bug Report

> **Scope:** Full audit of `app/` — models, schemas, auth, cache, timeutils,
> all routers, and all services.  
> **Total bugs fixed: 18**

---

## Bug 1 — Access Token Has Wrong Lifetime (900 Minutes Instead of 900 Seconds)

**File:** `app/auth.py` — **Line 50**

### Explanation
`ACCESS_TOKEN_EXPIRE_MINUTES` is `15` (minutes). The code passed it to
`timedelta(minutes=...)` and then **multiplied by 60 again** inside the
argument:

```python
# BEFORE (wrong)
lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)
# = timedelta(minutes=900) = 15 hours
```

Business rule: access tokens must expire in exactly **900 seconds (15 minutes)**.
The buggy code issued tokens that lived **900 minutes (15 hours)**.

### Fix
```python
# AFTER (correct)
lifetime = timedelta(seconds=ACCESS_TOKEN_EXPIRE_MINUTES * 60)
# = timedelta(seconds=900) = 15 minutes ✓
```

---

## Bug 2 — Revoked-Token Check Compared the Wrong Claim

**File:** `app/auth.py` — **Line 97**

### Explanation
`_revoked_tokens` stores `jti` values (token identifiers), populated correctly
by `revoke_access_token`. However, `get_token_payload` checked
`payload.get("sub")` (the user's ID string) against that set:

```python
# BEFORE (wrong)
if payload.get("sub") in _revoked_tokens:
```

A `sub` value would never match a stored `jti`, so **logout had no effect** —
any previously-issued token remained valid indefinitely after logout.

### Fix
```python
# AFTER (correct)
if payload.get("jti") in _revoked_tokens:
```

---

## Bug 3 — UTC Offset Not Applied Before Stripping tzinfo

**File:** `app/timeutils.py` — **Line 13**

### Explanation
When a timezone-aware datetime (e.g. `2025-01-01T10:00:00+05:30`) was
received, the code stripped the tzinfo with `.replace(tzinfo=None)` **without
first converting to UTC**:

```python
# BEFORE (wrong)
dt = dt.replace(tzinfo=None)  # strips offset, treats as UTC — wrong!
# 10:00+05:30 stored as 10:00 UTC, should be 04:30 UTC
```

This caused all datetimes with non-UTC offsets to be stored at the wrong time,
breaking conflict detection, quota windows, and refund notice calculations.

### Fix
```python
# AFTER (correct)
dt = dt.astimezone(timezone.utc).replace(tzinfo=None)
```

---

## Bug 4 — Booking Start-Time Allowed Up to 5 Minutes in the Past

**File:** `app/routers/bookings.py` — **Line 86**

### Explanation
The future-time guard subtracted a 300-second (5-minute) grace window:

```python
# BEFORE (wrong)
if start <= now - timedelta(seconds=300):
```

Business rule: `start_time` must be **strictly in the future at request time
with no grace window**. Any `start_time ≤ now` must be rejected.

### Fix
```python
# AFTER (correct)
if start <= now:
```

---

## Bug 5 — Missing `end_time > start_time` and Minimum Duration Checks

**File:** `app/routers/bookings.py` — **Lines 89–94**

### Explanation
Two validation checks were absent:

1. **`end <= start`** was never checked explicitly. A booking with
   `end == start` (zero duration) would pass the integer-hours check and the
   `> MAX` check.
2. **Minimum duration (`< 1 hour`)** was never checked. The business rule
   requires `1 ≤ duration_hours ≤ 8`.

A zero- or sub-hour booking would produce `price_cents = 0` and be persisted.

### Fix
Added two new validation guards before the maximum-duration check:
```python
if end <= start:
    raise AppError(400, "INVALID_BOOKING_WINDOW", "end_time must be after start_time")
...
if duration_hours < MIN_DURATION_HOURS:
    raise AppError(400, "INVALID_BOOKING_WINDOW", "duration out of range")
```

---

## Bug 6 — Back-to-Back Bookings Incorrectly Flagged as Conflicts

**File:** `app/routers/bookings.py` — **Line 50**

### Explanation
The overlap check used `<=` on both sides:

```python
# BEFORE (wrong)
if b.start_time <= end and start <= b.end_time:
```

This treats back-to-back bookings (e.g. existing ends at 14:00, new starts at
14:00) as a conflict. Business rule: **back-to-back is allowed**. Overlap
occurs only when `existing.start < new.end AND new.start < existing.end`.

### Fix
```python
# AFTER (correct — strict inequalities)
if b.start_time < end and start < b.end_time:
```

---

## Bug 7 — `list_bookings` Wrong Sort Order, Wrong Offset, Hardcoded Limit, Members See Only Own

**File:** `app/routers/bookings.py` — **Lines 134–140**

### Explanation
Four distinct bugs in one function:

| # | Buggy code | Expected behaviour |
|---|---|---|
| 7a | `Booking.start_time.desc()` | Must be `.asc()` per spec |
| 7b | `.offset(page * limit)` | Must be `.offset((page - 1) * limit)` (page 1 should return rows 0..limit-1) |
| 7c | `.limit(10)` hardcoded | Must use the `limit` query parameter |
| 7d | Always filters by `Booking.user_id == user.id` | **Admins** must see all bookings in their org, not just their own |

### Fix
```python
if user.role == "admin":
    base = db.query(Booking).join(Room, Booking.room_id == Room.id).filter(Room.org_id == user.org_id)
else:
    base = db.query(Booking).filter(Booking.user_id == user.id)
total = base.count()
items = (
    base.order_by(Booking.start_time.asc(), Booking.id.asc())
    .offset((page - 1) * limit)
    .limit(limit)
    .all()
)
```

---

## Bug 8 — `get_booking` Overwrites `start_time` with `created_at`

**File:** `app/routers/bookings.py` — **Line 166**

### Explanation
After calling `serialize_booking(booking)` (which correctly sets `start_time`),
the handler immediately overwrote that field with the wrong value:

```python
# BEFORE (wrong)
response["start_time"] = iso_utc(booking.created_at)
```

Every call to `GET /bookings/{id}` returned `created_at` in the `start_time`
field, giving clients completely wrong scheduling data.

### Fix
The erroneous line was removed entirely. `serialize_booking` already sets
`start_time` correctly from `booking.start_time`.

---

## Bug 9 — Refund Percent Uses Truncated Hours and Wrong 0% Branch

**File:** `app/routers/bookings.py` — **Lines 200–206**

### Explanation
Two errors in the refund-tier logic:

1. **Truncated hours used for `> 48h` tier**: `notice_hours = int(notice.total_seconds() // 3600)` then `if notice_hours > 48`. This truncates, so a notice of exactly 48 hours 59 minutes gets `notice_hours = 48`, misses the `> 48` branch, and falls into the 50% tier instead of 100%.

2. **0% branch returned 50%**: The final `else` branch was `refund_percent = 50` instead of `refund_percent = 0`, so cancellations with less than 24 hours notice incorrectly received a 50% refund.

```python
# BEFORE (wrong)
notice_hours = int(notice.total_seconds() // 3600)
if notice_hours > 48:
    refund_percent = 100
elif notice >= timedelta(hours=24):
    refund_percent = 50
else:
    refund_percent = 50   # ← should be 0!
```

### Fix
```python
# AFTER (correct — compare timedelta directly)
if notice > timedelta(hours=48):
    refund_percent = 100
elif notice >= timedelta(hours=24):
    refund_percent = 50
else:
    refund_percent = 0
```

---

## Bug 10 — Response `refund_amount_cents` Inconsistent with Stored `RefundLog`

**File:** `app/routers/bookings.py` — **Lines 208–210**

### Explanation
The cancellation handler computed `refund_amount_cents` in the response using
Python's `round()` (banker's rounding), but passed `refund_percent` (not the
computed cents) to `log_refund`, which used `int()` (truncation) to compute
what was stored in the DB:

```python
# BEFORE (wrong)
refund_amount_cents = round(booking.price_cents * (refund_percent / 100.0))
log_refund(db, booking, refund_percent)   # stored amount may differ!
```

The response could return a different value than what was in the `RefundLog`
table, violating the business rule that "the amount returned in the response
must match the amount stored in the RefundLog."

### Fix
`log_refund` now returns the saved `RefundLog` entry, and the response reads
its `amount_cents` field directly, guaranteeing consistency:

```python
refund_entry = log_refund(db, booking, refund_percent)
refund_amount_cents = refund_entry.amount_cents
```

---

## Bug 11 — Registration of Duplicate Username Returns 200 Instead of 409

**File:** `app/routers/auth.py` — **Lines 37–43**

### Explanation
When a user registered with a username already taken in the same organization,
the handler silently returned HTTP 200 with the **existing user's data** instead
of rejecting the request:

```python
# BEFORE (wrong)
if existing is not None:
    return {"user_id": existing.id, ...}   # HTTP 200 — wrong!
```

Business rule: duplicate username within same org → HTTP 409 `USERNAME_TAKEN`.

### Fix
```python
if existing is not None:
    raise AppError(409, "USERNAME_TAKEN", "Username already taken in this organization")
```

---

## Bug 12 — Refresh Tokens Are Not Single-Use (Reuse Allowed)

**File:** `app/routers/auth.py` — **Lines 81–93**

### Explanation
The `/auth/refresh` endpoint decoded the refresh token and issued a new pair,
but **never invalidated the presented token**. The same refresh token could be
used repeatedly to obtain fresh access tokens indefinitely — a major security
hole that also violated the single-use requirement.

### Fix
Added a module-level `_revoked_refresh_jtis: set[str]` set. After decoding, the
`jti` is checked against this set (returning 401 if already used), then added to
it before the new pair is issued:

```python
jti = data.get("jti")
if jti in _revoked_refresh_jtis:
    raise AppError(401, "UNAUTHORIZED", "Refresh token has already been used")
_revoked_refresh_jtis.add(jti)
```

---

## Bug 13 — Refund Rounding Truncates Instead of Rounding Half-Up

**File:** `app/services/refunds.py` — **Lines 15–17**

### Explanation
The stored refund amount was computed by converting to dollars, computing the
percentage, then converting back and **truncating** with `int()`:

```python
# BEFORE (wrong — truncation)
dollars = booking.price_cents / 100.0
refund_dollars = dollars * (percent / 100.0)
amount_cents = int(refund_dollars * 100)
# e.g. 50% of 101 cents = 50.5 → stored as 50, should be 51
```

Business rule: "Refund amounts must round to the nearest cent, half-cents
rounding up." `int()` always truncates (rounds down), not half-up.

### Fix
```python
# AFTER (correct — half-up rounding via math.floor(x + 0.5))
import math
raw_cents = booking.price_cents * percent / 100.0
amount_cents = math.floor(raw_cents + 0.5)
# e.g. 50% of 101 cents = 50.5 → floor(51.0) = 51 ✓
```

---

## Bug 14 — Reference Code Generation Has a Race Condition (Duplicate Codes)

**File:** `app/services/reference.py` — **Lines 17–21**

### Explanation
The counter read and write were separated by a 120 ms sleep with **no lock**:

```python
# BEFORE (wrong)
def next_reference_code() -> str:
    current = _counter["value"]   # Thread A reads 1000
    _format_pause()               # sleep — Thread B also reads 1000
    _counter["value"] = current + 1  # both write 1001
    return f"CW-{current:06d}"   # both return CW-001000 → DUPLICATE!
```

Business rule: every booking's `reference_code` must be unique, including under
concurrent creation.

### Fix
Wrapped the entire read-modify-write in a `threading.Lock`:

```python
_counter_lock = threading.Lock()

def next_reference_code() -> str:
    with _counter_lock:
        current = _counter["value"]
        _format_pause()
        _counter["value"] = current + 1
    return f"CW-{current:06d}"
```

---

## Bug 15 — Room Stats Have a Race Condition (Lost Updates)

**File:** `app/services/stats.py` — **Lines 15–19, 22–26**

### Explanation
Both `record_create` and `record_cancel` followed a read → sleep → write pattern
without any lock. Two concurrent requests racing through the same room's stats
would both read the same stale snapshot, then both write their own increment,
effectively losing one update:

```python
# Thread A reads count=5, sleeps; Thread B reads count=5, sleeps
# Thread A writes count=6; Thread B writes count=6 → count should be 7!
```

### Fix
Introduced `_stats_lock = threading.Lock()` and wrapped each function's
read-modify-write block inside `with _stats_lock:`.

---

## Bug 16 — Rate Limiter Has a Race Condition

**File:** `app/services/ratelimit.py` — **Lines 18–26**

### Explanation
The bucket read, filter, sleep, append, and write sequence had no lock. Two
concurrent requests for the same user could both read the same bucket (e.g.,
19 entries), both pass the limit check, and both write their updated 20-entry
bucket — effectively allowing the 21st request through undetected.

Additionally, the limit check used `> _MAX_REQUESTS` (triggers only at count
21), which means the **21st request was allowed** when the limit is 20.

```python
# BEFORE — check fires at > 20, i.e. at count 21:
if len(bucket) > _MAX_REQUESTS:
```

> [!NOTE]
> The check `> 20` fires at 21 entries, meaning 20 requests are accepted and
> the 21st is the first to be rejected. This is actually correct behaviour for
> a "max 20" limit (reject when the newly-appended count exceeds 20). The
> primary bug here is the **race condition** from the missing lock.

### Fix
Added `_buckets_lock = threading.Lock()` and wrapped the entire
`record_and_check` body in `with _buckets_lock:`, making the trim-append-check
sequence atomic.

---

## Bug 17 — Notifications Deadlock (Inconsistent Lock Acquisition Order)

**File:** `app/services/notifications.py` — **Lines 24–35**

### Explanation
`notify_created` acquires `_email_lock` first, then `_audit_lock`.
`notify_cancelled` acquires `_audit_lock` first, then `_email_lock`.

When a create and a cancel are processed concurrently, each holds one lock and
waits for the other → **classic deadlock**:

```
Thread A (create):   holds email_lock, waits for audit_lock
Thread B (cancel):   holds audit_lock, waits for email_lock
→ deadlock!
```

### Fix
Standardized the acquisition order to always acquire `_email_lock` before
`_audit_lock` in both functions:

```python
def notify_cancelled(booking) -> None:
    with _email_lock:          # consistent order
        with _audit_lock:
            _write_audit("cancelled", booking)
            _send_email("cancelled", booking)
```

---

## Bug 18 — CSV Export Leaks Cross-Organization Bookings

**File:** `app/services/export.py` — **Lines 48–54**

### Explanation
When `include_all=True` and a specific `room_id` was supplied, the code called
`fetch_bookings_raw(db, room_id)`, which queries bookings **without any
`org_id` filter**:

```python
# BEFORE (wrong — no org check)
if include_all:
    if room_id is not None:
        rows = fetch_bookings_raw(db, room_id)  # ← leaks any org's data!
```

An admin could supply the `room_id` of a room belonging to a different
organization and receive all of its booking data. This violates multi-tenancy
(Business Rule 9).

### Fix
Replaced the branch with `_fetch_scoped` which always enforces `org_id`:

```python
# AFTER (correct)
if include_all:
    rows = _fetch_scoped(db, org_id, None, room_id)
else:
    rows = _fetch_scoped(db, org_id, user_id, room_id)
```

---

## Summary Table

| # | File | Lines | Category | Severity |
|---|------|-------|----------|----------|
| 1 | `app/auth.py` | 50 | Token lifetime wrong (900 min vs 900 s) | High |
| 2 | `app/auth.py` | 97 | Revocation checks wrong claim (`sub` vs `jti`) | Critical |
| 3 | `app/timeutils.py` | 13 | UTC offset stripped without conversion | High |
| 4 | `app/routers/bookings.py` | 86 | 5-min grace window on start_time | Medium |
| 5 | `app/routers/bookings.py` | 89–96 | Missing `end > start` and min-duration checks | Medium |
| 6 | `app/routers/bookings.py` | 50 | Back-to-back bookings falsely conflict (`<=` vs `<`) | High |
| 7 | `app/routers/bookings.py` | 134–140 | Wrong sort/offset/limit/visibility in list_bookings | High |
| 8 | `app/routers/bookings.py` | 166 | `start_time` overwritten with `created_at` | High |
| 9 | `app/routers/bookings.py` | 200–206 | Refund percent: wrong tier logic + 0% returns 50% | High |
| 10 | `app/routers/bookings.py` | 208–210 | Response refund amount inconsistent with RefundLog | High |
| 11 | `app/routers/auth.py` | 37–43 | Duplicate username returns 200 instead of 409 | High |
| 12 | `app/routers/auth.py` | 81–93 | Refresh tokens not invalidated (reusable) | Critical |
| 13 | `app/services/refunds.py` | 15–17 | Rounding truncates instead of half-up | Medium |
| 14 | `app/services/reference.py` | 17–21 | Race condition → duplicate reference codes | Critical |
| 15 | `app/services/stats.py` | 15–26 | Race condition → lost stat updates | High |
| 16 | `app/services/ratelimit.py` | 18–26 | Race condition → rate limit bypassable | High |
| 17 | `app/services/notifications.py` | 24–35 | Inconsistent lock order → deadlock | Critical |
| 18 | `app/services/export.py` | 48–54 | Cross-org data leak in CSV export | Critical |
