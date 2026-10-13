# DECISIONS.md

1. **Indexes.**
   - `employees.emp_code` (unique): Guarantees employee codes are unique and speeds up employee lookups to O(log n).
   - `employees.(department, joined_on)`: Supports fast filtering by department and checking employee join dates for headcount calculations.
   - `employees.joined_on`: Standalone index for department summary headcount queries across all departments when no department filter is provided, avoiding COLLSCAN on large employee datasets.
   - `attendance_logs.(emp_code, date)` (unique): Ensures exactly one attendance log exists per employee per day. This prevents duplicate punch-in race conditions at the database level and speeds up lookups for punch-out, regularize, and department trend queries.
   - `attendance_logs.(date DESC, emp_code ASC)`: Supports pagination and sorting for `GET /attendance`. This allows MongoDB to fulfill queries using an index scan (`IXSCAN`) rather than an in-memory sort or collection scan on large datasets.
   - `attendance_logs.(date, late_minutes)`: Powers index scans for date-range filters in monthly analytics and late leaderboard calculations.

2. **Punch-in race.**
   Instead of doing an application-level find-then-insert check (which is susceptible to race conditions under concurrent requests), we rely on MongoDB's unique compound index on `(emp_code, date)`. If two duplicate punch-in requests arrive simultaneously, the database atomically commits the first insert and rejects the second with a `DuplicateKeyError`. The FastAPI endpoint catches this exception and returns an HTTP 409 Conflict.

3. **Ties.**
   The late leaderboard uses standard competition ranking (1, 2, 2, 4). When employees tie with the same `total_late_minutes`, they share the same rank. If there is a tie at the rank cutoff (e.g., two people tied at rank 10 when `limit=10`), all employees with `rank <= limit` are included in the response so no tied employee is excluded. Tied employees are sorted deterministically by `emp_code` ascending.

4. **Headcount.**
   Per rule R9, headcount counts all employees whose `joined_on` date is on or before the end of the reporting period (`joined_on <= end_date`). The department summary aggregation starts from the `employees` collection and performs a `$lookup` on `attendance_logs` using `localField: "emp_code"` and `foreignField: "emp_code"` with a date-filtered sub-pipeline. This ensures employees with zero attendance records in that period are still counted toward the department's total headcount while utilizing the `emp_date_unique` index on `attendance_logs` for zero collection scans (`collectionScans: 0`).

5. **One thing to change for 100x data.**
   At 100x scale (~10 million attendance records), running multi-stage aggregations on the raw logs on each request would put heavy load on MongoDB CPU and memory.
   The primary improvement would be creating pre-aggregated summary collections (using daily scheduled jobs or MongoDB's `$merge` pipeline). That way, monthly analytics endpoints become simple point queries rather than scanning millions of raw logs. Additionally, we could route analytics read queries to secondary replica members and shard `attendance_logs` by `(date, emp_code)` to distribute write traffic across nodes.
