# go-job-queue-dashboard

Production-inspired asynchronous job queue built with **Go** and **PostgreSQL**.  
This project demonstrates how background jobs can be processed safely using a concurrent worker pool, atomic database locking (`FOR UPDATE SKIP LOCKED`), and retry with exponential backoff.  
A minimal one-page admin dashboard (vanilla HTML + JavaScript) is included for observability and control.

---

## Overview

Modern applications often need to execute **long-running or unreliable tasks** (such as generating reports, sending emails, or processing files) without blocking user requests.

This project implements a **job queue system** where:
- API requests only **enqueue jobs**,
- background **workers process jobs asynchronously**,
- concurrency is handled safely at the database level,
- failed jobs are retried with controlled backoff,
- and job execution can be monitored via a simple dashboard.

The goal is to showcase **backend and fullstack fundamentals** without UI complexity or external message brokers.

---

## Key Features

- Asynchronous job processing using a configurable worker pool
- Atomic job claiming with PostgreSQL `FOR UPDATE SKIP LOCKED`
- Safe concurrency without double-processing
- Retry mechanism with exponential backoff (`run_at` scheduling)
- Clear job lifecycle tracking
- Job cancellation (queued jobs only)
- Minimal admin dashboard (vanilla HTML + JS)
- Dockerized local development (API + PostgreSQL)

---

## Running Migrations (golang-migrate)

This project uses SQL migration files in `backend/migrations` and runs them via a
`migrate` service in `docker-compose`.

### 1) Start PostgreSQL
```bash
docker compose up -d db
```

### 2) Apply latest migration
```bash
docker compose --profile tools run --rm migrate
```

### 3) Check migration version
```bash
docker compose --profile tools run --rm migrate \
  -path=/migrations \
  -database=postgres://jobuser:jobpass@db:5432/jobqueue?sslmode=disable \
  version
```

### 4) Roll back one migration
```bash
docker compose --profile tools run --rm migrate \
  -path=/migrations \
  -database=postgres://jobuser:jobpass@db:5432/jobqueue?sslmode=disable \
  down 1
```

---

## Job Lifecycle

### Each job follows a strict state machine:
```
queued
│
├── (claimed by worker)
▼
running
│
├── success ─────────────▶ succeeded
│
└── failure
│
├── attempts < max_attempts
│ └── retry (queued + run_at delay)
│
└── attempts >= max_attempts
└── failed
```

### Cancellation rules
- Jobs can be canceled **only while in `queued` state**
- Running jobs cannot be canceled to avoid inconsistent execution

---

##  Retry & Backoff Policy

- Default `max_attempts`: **3**
- Retry uses **exponential backoff**

Formula:
```
delay = min(max_delay, base_delay * 2^(attempts-1))
```

Default values:
- `base_delay`: 2 seconds
- `max_delay`: 60 seconds

Example:
| Failure Attempt | Retry Delay |
|----------------|-------------|
| 1st failure | 2s |
| 2nd failure | 4s |
| 3rd failure | stop → failed |

The `attempts` counter is incremented **only after a job fails**, not when it starts running.

---

## API Contract (Draft)

### Health Check
```
GET /health
```
Response:
```
{ "status": "ok" }
```
### Enqueue Job
```json
POST /v1/jobs
```
Request:
```json
{
  "type": "mock",
  "payload": { "sleep_ms": 1200 },
  "max_attempts": 3
}
```
Response:
```json
{
  "data": {
    "id": "uuid",
    "status": "queued"
  }
}
```
### Get Job by ID
```json
GET /v1/jobs/{id}
```

### List Jobs
```json
GET /v1/jobs?status=queued&limit=20
```

### Cancel Job
```json
POST /v1/jobs/{id}/cancel
```

### Error Response Format
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "payload must be valid JSON"
  }
}
```
---

## Database Schema (Draft)
| Field | Type | Description |
|-------|------|-------------|
|id	| UID|	Primary key
type	|TEXT|	Job type
payload |JSONB|   Job input data
status  |TEXT|    queued, running, succeeded, failed, canceled
attempts    |INT| Number of failed attempts
max_attempts    |INT| Retry limit
run_at  |TIMESTAMPTZ| Scheduled execution time
last_error  |TEXT|    Last failure reason
created_at  |TIMESTAMPTZ| Creation timestamp
updated_at  |TIMESTAMPTZ| Last update timestamp

Indexes
- (status, run_at, created_at)
- (created_at)
---
## Worker Model
- Workers run in the background as goroutines
- Worker concurrency is configurable via environment variable
- Each worker:
  1. Atomically claims a job from the database
  2. Marks it as running
  3. Processes the job
  4. Updates status based on the result
  5. Idle workers sleep briefly to avoid busy waiting.
---
##  Admin Dashboard
The project includes a minimal one-page admin dashboard built with vanilla HTML and JavaScript.
Dashboard capabilities:
- Enqueue new jobs
- View job list with auto-refresh
- Filter jobs by status
- Cancel queued jobs

---
## 📄 License
MIT License