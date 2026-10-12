# REVIEW.md

List every defect you found in the starter `app/main.py` (helpers and endpoints). For each one:

| # | Where (function / line) | What is wrong | How you'd notice it (test, input, or symptom) | How you fixed it |
|---|---|---|---|---|
| 1 | `health()` | Always returns 200 even when MongoDB is unreachable; never pings the DB | Grader polls `/health` before testing; 503 is never returned | Added `db.command("ping")` inside a try/except; raises 503 on failure |
| 2 | `compute_late_minutes` | Uses the punch-in timestamp's own date/time for shift_start instead of the attendance date in IST; breaks overnight shifts | Overnight employee punching in at 22:55 on day N is compared against wrong baseline | Fixed by building shift_start from the attendance date string in IST |
| 3 | `compute_late_minutes` | Threshold is `> 10` minutes but docs say "strictly more than 10 minutes after shift_start"; also uses `int(.../ 60)` instead of `floor` — same for overtime | 10:40:59 should give 10 late_minutes but returned 0 | Changed to floor-division via `int(elapsed_seconds // 60)` and kept `> 10` threshold |
| 4 | `compute_overtime` | Minimum threshold is missing — any overtime is recorded, but R3 says counted only when >= 30 min | 5 min after shift end incorrectly recorded as 5 overtime_minutes | Added `return elapsed_minutes if elapsed_minutes >= 30 else 0` |
| 5 | `compute_work_hours` | Uses `round(..., 2)` which is Python banker's rounding, not half-up | 9.125 h rounds to 9.12 instead of 9.13 | Replaced with custom `_round_half_up` using `math.floor(x * 100 + 0.5) / 100` |
| 6 | `create_employee()` | Race condition: uses find-then-insert; concurrent duplicate creates could both succeed | Two simultaneous POST /employees with same emp_code | Removed manual duplicate check; rely solely on the unique index + catch `DuplicateKeyError` |
| 7 | `create_employee()` | `created_at` set with `datetime.now()` (naive, local time) instead of UTC-aware datetime | Times stored as local time, returned incorrectly as epoch ms | Changed to `datetime.now(tz=timezone.utc)` |
| 8 | `list_employees()` | `skip = page * page_size` — off-by-one; page 1 skips the first page_size records | First page returns no results; page 0 returns the real first page | Fixed to `skip = (page - 1) * page_size` |
| 9 | `list_employees()` | `total = db.employees.count_documents({})` ignores the department filter | Filtered total is wrong | Changed to `count_documents(q)` after building the filter |
| 10 | `punch_in()` | `datetime.fromtimestamp(ms / 1000)` creates a naive local-time datetime instead of UTC-aware; also no truncation to whole seconds (R1) | Punch-in stored with local timezone offset; sub-second precision kept | Replaced with `_ms_to_dt` which uses `fromtimestamp(..., tz=UTC)` and strips microseconds |
| 11 | `punch_in()` | Date computed as `ts.date().isoformat()` — uses local time, not IST; overnight shift rule (R1) not applied | Wrong attendance date for employees in non-local timezones and overnight shifts | Replaced with `_attendance_date_for_punch` which converts to IST and applies overnight rule |
| 12 | `punch_in()` | No 404 check for unknown employee before using `emp` — would crash with `NoneType` | POST punch-in for non-existent emp_code causes 500 | Added `if not emp: raise HTTPException(404, ...)` |
| 13 | `punch_in()` | Returns `doc["id"] = str(res.inserted_id)` — `id` is not in the API contract; `_id` leaks | Response has extra `id` field, `_id` not removed properly | Removed; use `_serialize_attendance` which strips `_id` and converts datetimes |
| 14 | `list_attendance()` | Loads all matching documents into Python (`list(...)`) then slices — fails at scale (R10 says must stay fast on ~100k records) | Very slow / OOM on large dataset | Pushed skip/limit/sort to MongoDB cursor |
| 15 | `list_attendance()` | `doc["id"] = str(d.pop("_id"))` — `id` is not in the contract and `_id` is exposed | Response shape non-conformant | Replaced with `_serialize_attendance` |
| 16 | `punch_in()` | Race condition: check-then-insert for duplicate attendance date | Two simultaneous punch-ins for same employee/date both succeed | Rely on unique index on `(emp_code, date)` + catch `DuplicateKeyError` |
| 17 | All endpoints | `punch_in` / `punch_out` / `created_at` returned as Python datetime objects (not epoch ms) | Response fields are datetime strings, not integers | `_serialize_attendance` and `_serialize_employee` convert all datetimes to epoch ms |
| 18 | (Startup) | No indexes created at all; grader requires IXSCAN on ~100k records | `/admin/explain` returns COLLSCAN; slow queries | Added `create_indexes()` on startup for unique `emp_code`, unique `(emp_code, date)`, sort index |

**Not a defect:** The existing Pydantic models are functional as a starting point — only extended, not restructured.
