# Employee Attendance & Analytics API

A FastAPI and MongoDB REST API for tracking employee attendance, punch-ins/outs, and generating attendance analytics. Built as part of the HROne Software Engineer Trainee technical assignment.

## Features

- **Punch-in & Punch-out**: Automatically resolves IST calendar dates, supports overnight shifts, and computes late minutes, work hours, overtime, and half-day status per business rules (R1–R5).
- **Manual Corrections**: `PATCH /attendance/{emp_code}/{date}` with optimistic concurrency checks and an audit history trail (R6).
- **Analytics & Leaderboards**: Monthly summaries for employees and departments, late-comers leaderboard with competition ranking, and department attendance trends with 7-day moving averages (R7–R9).
- **Server-Side Aggregations**: All analytics queries run directly inside MongoDB aggregation pipelines (`$match`, `$lookup`, `$group`, `$densify`).
- **Indexes & Query Performance**: Database indexes created on application startup to ensure fast lookups (`IXSCAN`) and eliminate collection scans on large datasets (R10).
- **Concurrency Safety**: Unique compound index `(emp_code, date)` guarantees atomic duplicate rejection (409 Conflict) under race conditions.

## Tech Stack

- Python 3.11+
- FastAPI
- PyMongo
- Pydantic v2
- MongoDB 6.0+

## Local Setup & Run

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Configure environment variables (optional, defaults to localhost:27017 and attendance_db)
# On Linux / macOS:
export MONGO_URI="mongodb://localhost:27017"
export MONGO_DB="attendance_db"

# On Windows PowerShell:
$env:MONGO_URI="mongodb://localhost:27017"
$env:MONGO_DB="attendance_db"

# 3. Start the application
uvicorn app.main:app --port 8000
```

The API will be available at `http://localhost:8000`.  
Interactive Swagger documentation: `http://localhost:8000/docs`

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `MONGO_URI` | `mongodb://localhost:27017` | MongoDB connection string |
| `MONGO_DB` | `attendance_db` | Database name |

## Health Check

```bash
curl http://localhost:8000/health
```
Returns `{"status": "ok"}` when the application and MongoDB connection are healthy, or `503 Service Unavailable` if MongoDB is unreachable.

## Assumptions & Scope Notes

- **Working Days**: Per specification rule R7, working days are considered Monday through Friday.
- **Authentication**: Kept open without authentication/authorization per assignment specifications.
