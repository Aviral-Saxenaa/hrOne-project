# Employee Attendance & Analytics API

A high-performance Employee Attendance & Analytics REST API built with **Python 3.10+**, **FastAPI**, and **MongoDB**. Designed and implemented for the HROne engineering assignment following strict OpenAPI 3.0.3 contracts, business rules R1 through R10, concurrency constraints, and optimized MongoDB aggregation pipelines.

---

## Architecture & Design

### Technology Stack
- **Framework**: FastAPI (asynchronous ASGI framework, automatic validation, OpenAPI compliance)
- **Database**: MongoDB 6.0+ (utilizing WiredTiger unique compound indexes and aggregation pipelines)
- **Driver**: PyMongo 4.7+
- **Validation**: Pydantic v2
- **Precision**: Python `decimal.Decimal` with `ROUND_HALF_UP` for all mathematical calculations

### Business Rules Compliance (R1 - R10)
1. **R1 Time Zone & Attendance Date**:
   - Shifts and attendance dates operate strictly in Indian Standard Time (IST, UTC+05:30).
   - Instants are truncated to whole seconds before calculation and returned as epoch milliseconds integers.
   - For overnight shifts (`shift_end <= shift_start`), punch-ins earlier than `shift_end` in IST belong to the previous calendar day's shift.
2. **R2 Late Calculation**:
   - 10-minute grace period. Punch-in is late only if strictly more than 10 minutes past scheduled `shift_start`.
   - When late, `late_minutes = floor((punch_in - shift_start) / 60)` elapsed from shift start; otherwise 0.
3. **R3 Overtime**:
   - Overtime is calculated from scheduled `shift_end` to `punch_out`. Counted only when `>= 30` minutes; otherwise 0.
4. **R4 Work Hours**:
   - `(punch_out - punch_in)` in seconds / 3600, rounded to 2 decimals half-up (`Decimal.quantize(Decimal("0.01"), ROUND_HALF_UP)`).
5. **R5 Half Day**:
   - After punch-out, `half_day = True` when rounded `work_hours < 4.50`. Counts as 0.5 present day.
6. **R6 Presence**:
   - Statuses `PRESENT`, `WFH`, and `ON_DUTY` count as present. `ABSENT` and `LEAVE` do not.
7. **R7 Working Days**:
   - Mon-Fri (weekdays), calculated from `joined_on` for mid-month joiners. `present_days` counts only Monday–Friday logs.
8. **R8 Rounding**:
   - Quantized to 2 decimal places half-up (rates: 4 decimal places half-up).
9. **R9 Headcount**:
   - Employees with `joined_on <= period_end`, including those with zero logs.
10. **R10 Pagination**:
    - 1-indexed pagination (`skip = (page - 1) * page_size`), total reflects applied filters.

---

## Indexes & Performance Guarantees

All indexes are created idempotently on server startup:

### `employees` collection
- `{"emp_code": 1}` (**unique**): Enforces customer employee code uniqueness and O(1) point lookups.
- `{"department": 1, "emp_code": 1}`: Optimizes filtered listing by department with ascending sort.
- `{"department": 1, "joined_on": 1}` & `{"joined_on": 1}`: Accelerates headcount eligibility queries.

### `attendance_logs` collection
- `{"emp_code": 1, "date": 1}` (**unique**): Guarantees at most one record per employee per attendance date; prevents punch-in race conditions.
- `{"date": -1, "emp_code": 1}`: Backs fast pagination and sorting on `GET /attendance`.
- `{"status": 1, "date": -1, "emp_code": 1}`: Optimizes status-filtered queries.
- `{"emp_code": 1, "punch_in": -1}`: Speeds up finding the latest open punch-in for punch-out.
- `{"date": 1, "late_minutes": 1}`: Accelerates late leaderboard and monthly aggregation pipelines.

---

## Setup & Running

### Prerequisites
- Python 3.10+
- MongoDB 6.0+ (local instance or MongoDB Atlas)

### Local Execution
```bash
# Clone the repository
git clone <repo-url>
cd hrOne-project

# Create and activate virtual environment
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Configure environment variables (or export them directly)
export MONGO_URI="mongodb://localhost:27017"
export MONGO_DB="attendance_db"

# Seed sample data (optional)
python candidate_kit/sample_seed.py

# Start application server
uvicorn app.main:app --port 8000 --reload
```

---

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Database readiness check (pings MongoDB; returns 200 or 503) |
| `POST` | `/employees` | Create employee with validation and unique index enforcement |
| `GET` | `/employees` | List employees filtered by department, sorted by `emp_code`, paginated |
| `POST` | `/attendance/punch-in` | Punch-in recording with R1 overnight shift logic and R2 late calculation |
| `POST` | `/attendance/punch-out` | Punch-out recording with atomic concurrency lock and R3–R5 derived metrics |
| `GET` | `/attendance` | Fast index-backed paginated list with date/status filters |
| `PATCH` | `/attendance/{emp_code}/{date}` | Regularization with validation, derived recomputation, and audit history |
| `GET` | `/analytics/employees/{emp_code}/monthly` | Monthly employee summary via MongoDB aggregation |
| `GET` | `/analytics/departments/summary` | Department summary across collections (headcount includes zero-log employees) |
| `GET` | `/analytics/leaderboard/late` | Standard competition ranking (`$rank`) with post-ranking limit |
| `GET` | `/analytics/departments/{department}/trend` | Daily series with `$densify` gap-filling and 7-day moving average |
| `GET` | `/admin/explain/{endpoint}` | Execution plan output verifying `IXSCAN` index usage |

---

## Deliverables Checklist
- [x] `app/main.py` complete with all endpoints and business rules.
- [x] `requirements.txt` with required dependencies.
- [x] `REVIEW.md` detailing starter code defects and fixes.
- [x] `DECISIONS.md` explaining index design, concurrency, tie-breaking, and scalability.
- [x] Clean repository without `.env`, secrets, or `Dockerfile`.
