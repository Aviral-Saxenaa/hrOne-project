# Employee Attendance & Analytics API

Python 3.11+, FastAPI, and MongoDB 6.0+ implementation of the HROne attendance assignment.

## Technology

- **Python** runs the API and business rules.
- **FastAPI** exposes and validates the REST endpoints.
- **MongoDB** stores employees and attendance logs, enforces unique indexes, and runs the analytics aggregations.
- **PyMongo** is the Python-to-MongoDB driver; **Uvicorn** runs the FastAPI application.
- This assignment does not require an AI model, so the API does not call one. If an AI feature is added later, use Grok.

JSON is the REST API's data format, not a separate technology in the application stack. The `.json` files in [`candidate_kit/sample_data`](candidate_kit/sample_data) are MongoDB Extended JSON **sample records**, not another database or programming technology. The assignment's [`DATA_MODEL.md`](candidate_kit/DATA_MODEL.md) describes the corresponding MongoDB documents. The optional `candidate_kit/sample_seed.py` script loads them into MongoDB and **deletes existing documents in both collections first**; use it only with a disposable assignment database.

## Run locally

Install Python 3.11+ and MongoDB 6.0+ (local MongoDB or MongoDB Atlas), then from the repository root:

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
$env:MONGO_URI = "mongodb://localhost:27017"
$env:MONGO_DB = "attendance_db"
python candidate_kit\sample_seed.py
uvicorn app.main:app --port 8000
```

Set `MONGO_URI` to your MongoDB connection string. Do not commit credentials, `.env` files, or other secrets. The seed command is optional; the API creates its required indexes at startup.

The API contract and business rules are in [`candidate_kit/openapi.yaml`](candidate_kit/openapi.yaml); MongoDB collections and sample data are described in [`candidate_kit/DATA_MODEL.md`](candidate_kit/DATA_MODEL.md).

## Endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/health` | MongoDB readiness check |
| POST / GET | `/employees` | Create and list employees |
| POST | `/attendance/punch-in` | Record a punch-in |
| POST | `/attendance/punch-out` | Record a punch-out and derived values |
| GET | `/attendance` | Filtered, paginated attendance records |
| PATCH | `/attendance/{emp_code}/{date}` | Correct a record with an audit entry |
| GET | `/analytics/employees/{emp_code}/monthly` | Monthly employee summary |
| GET | `/analytics/departments/summary` | Department summary |
| GET | `/analytics/leaderboard/late` | Ranked late-arrival leaderboard |
| GET | `/analytics/departments/{department}/trend` | Daily attendance trend |
| GET | `/admin/explain/{endpoint}` | MongoDB query-plan details |
