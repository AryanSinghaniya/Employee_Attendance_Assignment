# Employee Attendance & Analytics API

A FastAPI and MongoDB REST API for employee attendance tracking and attendance analytics built with PyMongo.

## Implemented Features

- **Employees**: `POST /employees` (unique `emp_code`, validation), `GET /employees` (department filter, pagination per R10).
- **Attendance**:
  - `POST /attendance/punch-in`: IST date handling and overnight shift logic (R1), grace period and late calculation (R2), presence statuses (R6), unique `(emp_code, date)` constraint (409 Conflict on duplicate).
  - `POST /attendance/punch-out`: Work hours calculation with half-up rounding (R4, R8), overtime calculation (R3), half-day flag (R5), 409 Conflict if already punched out.
  - `GET /attendance`: Filter by `emp_code`, `date_from`, `date_to`, `status`, sorted by `date DESC, emp_code ASC` with pagination (R10).
  - `PATCH /attendance/{emp_code}/{date}`: Regularization updating derived fields, appending immutable audit history with changed fields only, optimistic concurrency control.
- **Analytics**:
  - `GET /analytics/employees/{emp_code}/monthly`: Working days (Mon-Fri) taking join date into account (R7), present days (R5, R6), leave days, late count and minutes, overtime, attendance percentage (R8).
  - `GET /analytics/departments/summary`: Department headcount including zero-log employees (R9), present days, average work hours (average of present records), late count/minutes, leave/on-duty counts.
  - `GET /analytics/leaderboard/late`: Top late employees with standard competition ranking (1, 2, 2, 4) with limit applied after ranking and ties broken by `emp_code` ASC.
  - `GET /analytics/departments/{department}/trend`: Daily attendance trend with MongoDB `$densify` gap filling, daily headcount (R9), attendance rate, and 7-day moving average (R8).
- **Admin Explain**:
  - `GET /admin/explain/{endpoint}`: MongoDB query execution plan with `executionStats` verbosity using `db.command()`, explaining the same aggregation pipelines used by the analytics endpoints.
- **System**:
  - `GET /health`: MongoDB ping probe (returns 200 on success, 503 on failure) and idempotent startup index creation.

## How to Run

1. Install dependencies:
```bash
pip install -r requirements.txt
```

2. Set environment variables and run:

On Linux / macOS:
```bash
export MONGO_URI="mongodb://localhost:27017"
export MONGO_DB="attendance_test"
uvicorn app.main:app --port 8000
```

On Windows PowerShell:
```powershell
$env:MONGO_URI="mongodb://localhost:27017"
$env:MONGO_DB="attendance_test"
uvicorn app.main:app --port 8000
```

Interactive API documentation is accessible at `http://localhost:8000/docs`.

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `MONGO_URI` | `mongodb://localhost:27017` | MongoDB connection string |
| `MONGO_DB` | `attendance_db` | Database name |

## Testing Status

I tested the implementation against local MongoDB on `attendance_test`:

1. **Functional Tests (88/88 passed)**:
   - Health check probe (200 on DB ping, 503 on failure).
   - Business rules: R1 overnight shift dates and punch-out, R2 late minutes (09:40:00 -> 0, 09:40:01 -> 10, 10:15:59 -> 45), R3 overtime (29 min -> 0, 30 min -> 30), R4 work hours decimal half-up rounding (522 sec -> 0.15 h), R5 half day (< 4.50 h).
   - Schema contracts: Strict integer epoch ms validation (strings, floats, bools, seconds -> 422), no `_id` or `id` exposed, real calendar date validation (joined_on, date_from/date_to, trend from/to, patch date -> 422 for invalid calendar dates).
   - Concurrency tests: 20 concurrent employee creates (exactly one 201, remainder 409), 20 concurrent punch-ins (exactly one 201, remainder 409), 20 concurrent punch-outs (exactly one 200, remainder 409, zero 404s), concurrent regularizations without lost history.
   - Regularization: ABSENT/LEAVE clearing punches, history entry tracking with only changed fields, no-change -> 422.
   - Analytics correctness: Mid-month joiners (R7), zero-log employees in headcount (R9), weekend records, tie ranking (1, 2, 2, 4) on leaderboard, orphan log handling.
   - Explain validation: All five endpoints return HTTP 200 with `executionStats`, `IXSCAN` present, and no `COLLSCAN`.

2. **Scale & Query Performance (~100,200 records)**:
   - Seeded 300 employees and 100,200 attendance records spanning 2026.
   - Verified `/admin/explain` on all five endpoints: all 5 return HTTP 200, `IXSCAN: True`, and `COLLSCAN: False` (including `$lookup` stages with 0 collection scans).
   - Real endpoint response times on 100,200 records:
     - `GET /attendance`: ~13 ms
     - `GET /analytics/employees/{emp_code}/monthly`: ~10 ms
     - `GET /analytics/departments/summary`: ~59 ms
     - `GET /analytics/leaderboard/late`: ~103 ms
     - `GET /analytics/departments/{department}/trend`: ~18 ms

## Not Done / Limitations

- **Authentication & Authorization**: Kept open without tokens or user roles per assignment specification.
- **Dynamic Holiday Calendar**: Per business rule R7, working days are fixed to Monday through Friday without public holiday calendar lookups.
- **Dynamic Shift Assignment**: Employees have fixed shift start and end times configured on the employee document.
