# Employee Attendance & Analytics API

A FastAPI + MongoDB REST API for employee attendance tracking and analytics, built for the HROne Software Engineer Trainee assignment.

## Features

- **Punch-in / Punch-out** with IST date resolution and overnight-shift support (R1)
- **Late minutes, work hours, overtime, half-day** computed per business rules R2–R5
- **Manual corrections** (`PATCH /attendance/{emp_code}/{date}`) with audit trail (`history` array) and concurrency control
- **Analytics**: Monthly employee summary, department summary, late-comers leaderboard (competition ranking with ties), and daily department trend with 7-day moving average
- **MongoDB aggregation pipelines**: All analytics run server-side using `$match`, `$lookup`, `$group`, `$densify`, and `$project`, reading stored values directly
- **MongoDB indexes** created at startup for O(log n) lookups and `IXSCAN` support (no `COLLSCAN` on ~100k records)
- **Concurrency-safe**: Unique compound index `(emp_code, date)` guarantees atomic duplicate rejection (409)

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

# 2. Configure environment (optional, defaults to localhost:27017)
cp .env.example .env
# Set MONGO_URI and MONGO_DB if using a custom instance

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

## Verification & Health Check

```bash
# Health check probe (returns 200 if MongoDB is up, 503 if unreachable)
curl http://localhost:8000/health
```

## What is Missing / Known Limitations

- **Holiday Calendar**: Per specification rule R7, working days are strictly Monday–Friday without dynamic holiday integration.
- **Authentication**: No JWT/OAuth authentication or role-based authorization is implemented, as specified by the assignment requirements.
