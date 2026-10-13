# Employee Attendance & Analytics API

A FastAPI and MongoDB REST API for tracking employee attendance, punch-ins/outs and attendance analytics. Built for the HROne Software Engineer Trainee technical assignment. All code is in `app/main.py`.

## What is implemented

- `GET /health`: 200 if MongoDB answers a ping, otherwise 503.
- `POST /employees`, `GET /employees`: unique `emp_code` (enforced by a unique index, 409 on duplicates), filtered `total`, pagination.
- `POST /attendance/punch-in`, `POST /attendance/punch-out`: IST attendance date, overnight shifts, and late minutes, work hours, overtime and half day (R1-R5); 404 / 409 / 422 handling.
- `GET /attendance`: sorted by `date` descending then `emp_code`, paginated.
- `PATCH /attendance/{emp_code}/{date}`: manual correction. Recomputes derived fields, appends exactly one `history` entry with only the changed fields, and rejects a concurrent edit with 409 instead of losing history.
- Analytics as MongoDB aggregation pipelines that read the stored values: employee monthly summary, department summary, late leaderboard (competition ranking with ties), department trend (`$densify`, 7-day moving average).
- `GET /admin/explain/{endpoint}`: MongoDB `explain` with `executionStats`, run on the same pipelines the analytics endpoints use.
- Indexes are created at startup (and retried from `/health` if MongoDB was not ready).

## Tech stack

Python 3.11+, FastAPI, PyMongo, Pydantic v2, MongoDB 6.0+ (`$densify` needs 5.1+).

## Run

```bash
pip install -r requirements.txt

# Linux / macOS
export MONGO_URI="mongodb://localhost:27017"
export MONGO_DB="attendance_db"
# Windows PowerShell
# $env:MONGO_URI="mongodb://localhost:27017"; $env:MONGO_DB="attendance_db"

uvicorn app.main:app --port 8000
```

A `.env` file also works; real environment variables take priority.
Swagger UI: http://localhost:8000/docs

| Variable | Default | Description |
|---|---|---|
| `MONGO_URI` | `mongodb://localhost:27017` | MongoDB connection string |
| `MONGO_DB` | `attendance_db` | Database name |

## Testing status

I ran these checks against a local MongoDB (database `attendance_test`) with a throwaway test script that is not part of this repository:

- Business rules: overnight shift dates (R1), late minutes (09:40:00 -> 0, 09:40:01 -> 10, 10:15:59 -> 45), overtime (29 min -> 0, 30 min -> 30), half-up work hours (522 seconds -> 0.15), half day.
- Contract: epoch milliseconds everywhere, strict integers (string, float and bool epoch values -> 422), no `_id` in responses, invalid dates and statuses -> 422, 404 / 409 codes.
- Concurrency with 20 threads: employee create, punch-in and punch-out each give exactly one success and the rest 409 (never 404); two simultaneous regularizations do not lose a history entry.
- Regularize: one history entry with only the changed fields, ABSENT / LEAVE clear the punches, a request that changes nothing -> 422.
- Analytics on a small hand-checked dataset: leaderboard ties (1, 2, 2, 4), employee with no logs counted in headcount, mid-month joiner, weekend records, orphan logs.
- `/admin/explain` on about 100,000 attendance records and 300 employees: all five endpoints returned 200 with `IXSCAN` and no `COLLSCAN`. Endpoint response times were roughly 10-100 ms.
- `/health` returns 503 when MongoDB is unreachable (checked by pointing the app at a closed port), and the app still starts.

## Not done / limitations

- No holiday calendar: working days are Monday to Friday (R7).
- No authentication.
- The tests are not included in this repository.
- I did not test other MongoDB versions or datasets much larger than 100,000 records.