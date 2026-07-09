# Bug Report — CoWork API

## Bug 1: Offset-aware datetimes not converted to UTC before storage

**File:** `app/timeutils.py:13`  
**Root cause:** `parse_input_datetime` called `dt.replace(tzinfo=None)` on offset-aware inputs without first converting to UTC. E.g., `2026-07-09T10:00:00+05:00` was stored as `10:00` (local time) instead of `05:00` (UTC equivalent).  
**Fix:** Added `dt = dt.astimezone(timezone.utc)` before stripping tzinfo.

---

## Bug 2: start_time grace window of 5 minutes

**File:** `app/routers/bookings.py:86`  
**Root cause:** The check `if start <= now - timedelta(seconds=300)` gave a 5-minute grace window for past start_times instead of the required strict future check.  
**Fix:** Changed to `if start <= now`.

---

## Bug 3: Missing end_time > start_time validation

**File:** `app/routers/bookings.py:89-90`  
**Root cause:** No validation that `end_time` is strictly after `start_time`. The spec requires `end_time` strictly after `start_time` (400 INVALID_BOOKING_WINDOW).  
**Fix:** Added `if end <= start` check that raises 400.

---

## Bug 4: Missing minimum duration check

**File:** `app/routers/bookings.py:93-95`  
**Root cause:** Duration check only tested `duration_hours > MAX_DURATION_HOURS` but didn't enforce the 1-hour minimum.  
**Fix:** Changed condition to `if duration_hours < MIN_DURATION_HOURS or duration_hours > MAX_DURATION_HOURS`.

---

## Bug 5: Double-booking overlap allowed back-to-back conflict

**File:** `app/routers/bookings.py:50`  
**Root cause:** The overlap check used `b.start_time <= end and start <= b.end_time`, which treated back-to-back bookings (one ending exactly when another starts) as conflicting. The spec says back-to-back is allowed. Also, `_pricing_warmup()` sleep created a TOCTOU race window.  
**Fix:** Changed to strict `<` on both sides: `b.start_time < end and start < b.end_time`. Removed the artificial sleep.

---

## Bug 6: Double-booking race condition (TOCTOU)

**File:** `app/routers/bookings.py:88-107`  
**Root cause:** The conflict check and booking insert were not atomic. Two concurrent requests could both pass the conflict check and then both insert overlapping bookings. The `_pricing_warmup()` sleep widened the race window.  
**Fix:** Added a `threading.Lock` (`_booking_lock`) that wraps the conflict check, quota check, and booking insert in a single critical section. Removed the artificial sleep.

---

## Bug 7: Quota check race condition (TOCTOU)

**File:** `app/routers/bookings.py:55-56, 69-71, 92`  
**Root cause:** Same TOCTOU pattern as double-booking: the count query and insert were not atomic. The `_quota_audit()` sleep created a race window.  
**Fix:** The quota check is now inside the `_booking_lock` critical section, serializing it with the insert. Removed the artificial sleep.

---

## Bug 8: <24h notice returns 50% instead of 0%

**File:** `app/routers/bookings.py:205-206`  
**Root cause:** The `else` branch (notice < 24h) set `refund_percent = 50` instead of `0`.  
**Fix:** Changed to `refund_percent = 0`.

---

## Bug 9: ≥48h notice boundary off by one

**File:** `app/routers/bookings.py:201-204`  
**Root cause:** The condition `if notice_hours > 48` (using `>` and truncating to hours) meant a notice of exactly 48h got 50% instead of 100%. The spec says ≥48h → 100%. Also, using `int(notice.total_seconds() // 3600)` truncated fractional hours.  
**Fix:** Changed to compare `notice.total_seconds() / 3600 >= 48` (float comparison), and the elif uses `notice_hours >= 24`.

---

## Bug 10: Half-cent rounding used banker's rounding instead of half-up

**File:** `app/routers/bookings.py:202` (`refunds.py:15-17`)  
**Root cause:** `round(price_cents * percent / 100.0)` uses Python's banker's rounding, which rounds 0.5 to the nearest even integer (e.g., round(50.5) = 50). The spec requires half-up (50.5 → 51). The `log_refund` function separately used `int(refund_dollars * 100)` which truncates instead of rounding.  
**Fix:** Both functions now use integer arithmetic `(price_cents * percent + 50) // 100` for proper half-up rounding.

---

## Bug 11: Cancellation lock ordering deadlock risk

**File:** `app/services/notifications.py:31-35`  
**Root cause:** `notify_created` acquires `_email_lock` then `_audit_lock`, but `notify_cancelled` acquired `_audit_lock` then `_email_lock` — a classic lock ordering inversion that can cause deadlock.  
**Fix:** Changed `notify_cancelled` to acquire `_email_lock` first, then `_audit_lock`, matching the order in `notify_created`.

---

## Bug 12: Concurrent cancel could double-refund

**File:** `app/routers/bookings.py:188-207`  
**Root cause:** No lock around the status check, refund log creation, and status update. Two concurrent cancels could both see `status == "confirmed"` and both log refunds. Also, `log_refund` committed separately from the status update.  
**Fix:** Wrapped status check, refund calculation, `log_refund`, and status update inside `_booking_lock`. Changed `log_refund` to use `db.flush()` instead of `db.commit()` so the refund log is in the same transaction as the status change.

---

## Bug 13: Reference code race condition + no unique constraint

**File:** `app/services/reference.py:17-21`, `app/models.py:55`  
**Root cause:** `next_reference_code` read the counter, slept, then incremented — allowing two concurrent calls to produce the same code. The `reference_code` column had no unique constraint.  
**Fix:** Added `threading.Lock` around the read-increment-return sequence in `reference.py`. Added `unique=True` on the column and a `UniqueConstraint` table arg in `models.py`.

---

## Bug 14: Rate limiter race condition + window boundary off-by-one

**File:** `app/services/ratelimit.py:18-26`  
**Root cause:** The read-filter-sleep-append pattern allowed concurrent requests to both pass with one slot left. The filter used `t > now - _WINDOW_SECONDS` (strict `>`), dropping entries exactly 60s old and potentially allowing 21 requests through.  
**Fix:** Added `threading.Lock` around the entire read-filter-append-check sequence. Changed to `>=` for the filter.

---

## Bug 15: Room stats counter race condition

**File:** `app/services/stats.py:15-26`  
**Root cause:** `record_create` and `record_cancel` each read the current value, slept, then wrote back — a classic read-modify-write race that loses concurrent updates.  
**Fix:** Added `threading.Lock` and moved all read-modify-write inside it.

---

## Bug 16: Access token lifetime 15 hours instead of 15 minutes (900s)

**File:** `app/auth.py:49`  
**Root cause:** `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)` evaluates to `timedelta(minutes=900)` = 15 hours. The spec says access token exp − iat = exactly 900 seconds (15 minutes).  
**Fix:** Changed to `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)`.

---

## Bug 17: Token revocation checks `sub` instead of `jti`

**File:** `app/auth.py:97`  
**Root cause:** `revoke_access_token` stores `payload["jti"]` in the revoked set, but `get_token_payload` checked `payload.get("sub") in _revoked_tokens`. This meant revoking a token had no effect — it stored the jti but checked the user id.  
**Fix:** Changed to `payload.get("jti") in _revoked_tokens`.

---

## Bug 18: Refresh tokens not single-use

**File:** `app/routers/auth.py:78-93`  
**Root cause:** The `/refresh` endpoint did not invalidate the old refresh token after issuing new ones, so old refresh tokens could be reused indefinitely.  
**Fix:** Added revocation check before processing, and `revoke_refresh_token(data)` after validation so the old token is immediately invalidated. Reuse returns 401.

---

## Bug 19: Registration returns existing user instead of 409

**File:** `app/routers/auth.py:34-40`  
**Root cause:** When a duplicate username within an org was detected, the endpoint returned the existing user's data with status 200 instead of raising 409 USERNAME_TAKEN.  
**Fix:** Replaced the return statement with `raise AppError(409, "USERNAME_TAKEN", ...)`.

---

## Bug 20: GET /bookings/{id} sets start_time to created_at

**File:** `app/routers/bookings.py:159` (formerly line 166)  
**Root cause:** After serializing the booking, the response's `start_time` was overwritten with `iso_utc(booking.created_at)` instead of using the correct `start_time` from serialization. This made the booking detail endpoint return the creation timestamp as the start time.  
**Fix:** Removed the erroneous override line.

---

## Bug 21: GET /bookings/{id} missing member ownership check

**File:** `app/routers/bookings.py:156`  
**Root cause:** The endpoint checked that the booking belonged to the user's org, but did not check that a non-admin user was the booking owner. Members could read any booking within their org.  
**Fix:** Added `if user.role != "admin" and booking.user_id != user.id: raise AppError(404, ...)`.

---

## Bug 22: GET /bookings pagination wrong sort, offset, and limit

**File:** `app/routers/bookings.py:128-132`  
**Root cause:** Three bugs: (a) `start_time.desc()` sorted descending instead of ascending; (b) `page * limit` computed wrong offset (page 1 should offset 0, not `1 * L`); (c) `.limit(10)` was hardcoded instead of using the `limit` query parameter.  
**Fix:** Changed to `start_time.asc()`, `(page - 1) * limit`, and `.limit(limit)`.

---

## Bug 23: GET /bookings for admin shows only own bookings

**File:** `app/routers/bookings.py:124-126`  
**Root cause:** The query always filtered by `Booking.user_id == user.id`, preventing admins from seeing all bookings in their org.  
**Fix:** Changed to filter by `Room.org_id == user.org_id`, then conditionally add `Booking.user_id == user.id` only for non-admin users.

---

## Bug 24: Usage report cache not invalidated on booking creation

**File:** `app/routers/bookings.py:111`  
**Root cause:** `create_booking` invalidated the availability cache but not the report cache, so the usage report would be stale after a new booking.  
**Fix:** Added `cache.invalidate_report(user.org_id)` in `create_booking`.

---

## Bug 25: Availability cache not invalidated on cancellation

**File:** `app/routers/bookings.py:211`  
**Root cause:** `cancel_booking` only invalidated the report cache, not the availability cache. So cancelled bookings would still appear as busy in availability.  
**Fix:** Added `cache.invalidate_availability(booking.room_id, booking.start_time.date().isoformat())` in `cancel_booking`.

---

## Bug 26: Cross-org data leak in CSV export

**File:** `app/services/export.py:22-29, 48-50`  
**Root cause:** `fetch_bookings_raw` queried bookings by `room_id` without joining on `Room.org_id`. When `include_all=True` with a `room_id`, a booking from another org's room with the same ID could leak.  
**Fix:** Added `join(Room)` and filter `Room.org_id == org_id` to `fetch_bookings_raw`. Updated its call site to pass `org_id`.
