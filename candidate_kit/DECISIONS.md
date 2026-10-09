# DECISIONS.md

1. **Indexes.**
   - `employees: { emp_code: 1 }` (unique): guarantees business key uniqueness and O(1) point lookups.
   - `employees: { department: 1, emp_code: 1 }`: accelerates department filtering and alphabetical sorting.
   - `employees: { joined_on: 1 }`: speeds up headcount eligibility queries (Rule R9).
   - `attendance_logs: { emp_code: 1, date: 1 }` (unique): enforces the one-record-per-day constraint and eliminates punch-in race conditions.
   - `attendance_logs: { date: -1, emp_code: 1 }`: backs fast pagination and sorting on `GET /attendance`.
   - `attendance_logs: { emp_code: 1, punch_in: -1 }`: enables instant lookup for punch-out.
   - *Rejected:* Single-field index on `{ status: 1 }`. With only five distinct enum values, low cardinality yields poor selectivity and wastes RAM.

2. **Punch-in race.**
   Both requests pass application validation and attempt `insert_one`. WiredTiger’s unique index on `(emp_code, date)` serializes the write: the first transaction commits (201 Created); the second triggers `DuplicateKeyError`, cleanly translated into HTTP 409 Conflict without data corruption.

3. **Ties.**
   We use standard competition ranking (`$rank` in `$setWindowFields`). If two employees tie at rank 10, both receive rank 10 and are returned since `rank <= limit`. The subsequent rank (11) is skipped. Tied rows are ordered by `emp_code` ascending.

4. **Headcount.**
   The aggregation roots in `employees`, filtering `joined_on <= period_end`, then `$lookup`s attendance logs. Employees with zero logs are preserved (`preserveNullAndEmptyArrays: true`) and counted toward headcount.

5. **One thing you would change.**
   Partition/shard `attendance_logs` by month and maintain pre-aggregated monthly summary rollups instead of running on-demand aggregation pipelines over 10M+ raw logs.
