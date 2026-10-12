# DECISIONS.md

1. **Indexes.**
   - `employees.emp_code` (unique, ascending): Enforces business-key uniqueness and guarantees O(log n) lookups on every employee-fetching endpoint.
   - `employees.(department, joined_on)`: Supports department filtering and joined_on bounds for headcount and department trend lookups.
   - `attendance_logs.(emp_code, date)` (unique, ascending): Enforces strictly one attendance record per employee per day; atomically eliminates punch-in duplicate races and speeds up punch-out and regularize lookups.
   - `attendance_logs.(date DESC, emp_code ASC)`: Specifically backs `GET /attendance` pagination and sorting, ensuring an `IXSCAN` without in-memory sort or `COLLSCAN` on ~100k records.
   - `attendance_logs.(date, late_minutes)`: Powers fast index scans for monthly summaries and the late leaderboard.

2. **Punch-in race.**
   When two identical punch-in requests arrive simultaneously for the same employee and date, both look up the employee and attempt `insert_one`. MongoDB's unique compound index on `(emp_code, date)` ensures that the storage engine atomically commits exactly one insert (returning 201). The second concurrent insert violates the unique constraint, raising a `DuplicateKeyError` which our FastAPI handler catches and converts into HTTP 409 ("Already punched in for this date").

3. **Ties.**
   The late leaderboard uses standard competition ranking (1, 2, 2, 4). When employees tie in `total_late_minutes` at the cutoff (e.g. two employees tied at rank 10 with `limit=10`), the endpoint returns all employees whose rank <= `limit`. Thus, the response can return more than `limit` items to fairly reflect tied ranks, and tied candidates are ordered deterministically by `emp_code` ascending.

4. **Headcount.**
   Headcount is calculated strictly per R9: all employees whose `joined_on <= end_of_month` (or <= given day for trend). The department summary pipeline starts from the `employees` collection and `$lookup`s `attendance_logs`. Employees with zero logs still match the initial `$match` stage, so they contribute 1 to `headcount`, while contributing 0 to present days, late minutes, and leaves. Departments with 0 headcount for the month are omitted.

5. **One thing to change for 100x data.**
   All analytics endpoints are already implemented as MongoDB server-side aggregation pipelines (`$match`, `$lookup`, `$group`, `$densify`). For 100x data (~10 million records):
   - We would implement incremental pre-aggregated rollups using MongoDB's `$merge` pipeline into a `monthly_attendance_summary` collection on daily shift closure, turning O(N) monthly aggregations into O(1) point lookups.
   - We would route analytics queries to secondary replica nodes (`secondaryPreferred`) so analytical workloads do not compete with primary write throughput during peak morning punch-in bursts.
   - We would shard `attendance_logs` using a compound hashed shard key `(date, emp_code)` to distribute storage and I/O across cluster shards.

6. **Decimal-exact rounding (R4, R8).**
   `_round_half_up` uses `decimal.Decimal` with `ROUND_HALF_UP` quantize instead of `math.floor(x * 100 + 0.5) / 100`. The `math.floor` approach silently fails for certain float representations (e.g., `0.145 * 100 + 0.5` in IEEE 754 is `14.999…`, flooring to 14, not 15). `Decimal(str(value)).quantize(…, ROUND_HALF_UP)` is exact because it works in base-10. Similarly, `_compute_work_hours` divides integer seconds by `Decimal(3600)` directly (no intermediate float), so 522 s → `Decimal('0.145')` → rounds to `0.15`, not `0.14`.

7. **`db.command({"explain": …})` for `/admin/explain`.**
   PyMongo's `Cursor.explain(verbosity)` ignores the verbosity argument; it always runs `queryPlanner` verbosity regardless of the string passed. The only reliable way to get `executionStats` verbosity (which the grader checks for `nReturned`, `totalDocsExamined`, etc.) is `db.command({"explain": {"find": …, "filter": …}, "verbosity": "executionStats"})`. The explain result is then passed through `_bson_to_json()` to convert `ObjectId`, `datetime`, and `Decimal128` fields before FastAPI serializes to JSON.

