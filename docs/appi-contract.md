# SETU API Contract (v1)

**SETU – Secure E-Governance Technology Unification**
Smart India Hackathon 2026 · Problem Statement 26129

This file is the single source of truth between backend (Dev A) and frontend (Dev B), and for the seed data (Dev C).
**To change anything here, open a pull request that edits this file.** Mark breaking changes in the PR title with `[BREAKING]`.

> All data in this project is **synthetic demo data**. No real citizen data is used.

---

## 1. Conventions

| Item | Rule |
|---|---|
| Base URL | Local: `http://127.0.0.1:8000` · Deployed: set in frontend as `VITE_API_BASE_URL` |
| Format | JSON in and out (except the `/mock/*` routes, see section 5) |
| Auth | `Authorization: Bearer <access_token>` on every route except `/health`, `/auth/login` and `/mock/*` |
| Timestamps | ISO 8601 UTC, e.g. `2026-09-21T10:30:00Z` |
| Dates | ISO `YYYY-MM-DD` in SETU responses (departments use their own formats, adapters convert them) |
| IDs | Strings. Prefixes: `SETU-C###` citizen, `APP-####` application, `CON-####` consent, `NOT-####` notification, `AUD-####` audit entry |
| Language | The API returns **codes and English text only**. Marathi/Hindi labels are handled in the frontend |
| Docs | FastAPI auto docs at `/docs` must match this file |

### Roles

| Role | Can do |
|---|---|
| `citizen` | Own profile, consents, applications, notifications |
| `official` | View all applications and unified citizen profiles, decide on applications, read audit log |
| `admin` | Everything an official can, plus monitoring, data quality, and department simulation |

### Demo users (hard-coded)

| Username | Password | Role | `citizen_id` |
|---|---|---|---|
| `asha` | `demo123` | citizen | `SETU-C001` |
| `rohan` | `demo123` | citizen | `SETU-C002` |
| `officer1` | `demo123` | official | – |
| `admin1` | `demo123` | admin | – |

### Error format

Every error uses the same shape:

```json
{
  "error": {
    "code": "CONSENT_REQUIRED",
    "message": "Consent is missing for: land",
    "details": { "missing": ["land"] }
  }
}
```

| HTTP | `code` | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | Missing or invalid token |
| 403 | `FORBIDDEN` | Role not allowed |
| 403 | `CONSENT_REQUIRED` | A department the service needs has no active consent |
| 404 | `NOT_FOUND` | Unknown ID |
| 409 | `DUPLICATE_SUBMISSION` | Citizen already has an open application for this service (`details.application_id` gives the existing one) |
| 422 | `VALIDATION_ERROR` | Bad request body |
| 503 | `DEPARTMENT_UNAVAILABLE` | Only when **no** department could be reached. If some fail, see `partial` in section 3 |

---

## 2. Common data schema ("SETU Citizen Profile")

Every department has its own format and IDs. The adapters convert them into this one shape.

```json
{
  "citizen_id": "SETU-C001",
  "name": "Asha Ramesh Patil",
  "dob": "2004-03-14",
  "birth_place": "Aurangabad",
  "education": {
    "qualification": "HSC",
    "percentage": 82.5,
    "board": "Maharashtra State Board",
    "passing_year": 2022
  },
  "land": {
    "survey_no": "112/4",
    "area_hectares": 1.62,
    "district": "Aurangabad"
  },
  "income_annual_inr": 180000,
  "sources": ["civil", "education", "land"],
  "quality_flags": []
}
```

Rules:

- Fields from a department that was not fetched (no consent, or department down) are `null`, and the department is missing from `sources`.
- `quality_flags` is a list of data-quality issues found while merging:

```json
{ "field": "name", "department": "education", "issue": "FORMAT_NORMALISED", "detail": "PATIL ASHA R -> Asha Ramesh Patil" }
```

Allowed `issue` values: `FORMAT_NORMALISED`, `MISMATCH`, `MISSING_FIELD`, `INVALID_VALUE`.

### Master ID registry (master data management)

Each citizen has one SETU ID mapped to that citizen's ID in each department. Dev C provides this as `data/master_registry.json`:

```json
[
  { "citizen_id": "SETU-C001", "civil": "BR-2004-00123", "education": "STU-77821", "land": "LR-55210" },
  { "citizen_id": "SETU-C002", "civil": "BR-2002-00456", "education": "STU-64310", "land": "LR-55377" }
]
```

---

## 3. Endpoints

### Health

`GET /health` → `200 {"status": "ok"}` (no auth)

### Auth

**`POST /auth/login`**

```json
// request
{ "username": "asha", "password": "demo123" }

// 200 response
{
  "access_token": "<jwt>",
  "token_type": "bearer",
  "user": { "username": "asha", "name": "Asha Patil", "role": "citizen", "citizen_id": "SETU-C001" }
}
```

**`GET /auth/me`** → same `user` object as above.

### Services

**`GET /services`** → list of services.

```json
[
  {
    "service_id": "SVC-SCHOLARSHIP",
    "name": "Skill Training Scholarship",
    "description": "Financial support for approved skill training courses",
    "required_departments": ["civil", "education", "land"],
    "sla_days": 7
  },
  {
    "service_id": "SVC-STARTUP-SEED",
    "name": "Startup Seed Support",
    "description": "Seed grant for first-time entrepreneurs",
    "required_departments": ["civil", "land"],
    "sla_days": 10
  }
]
```

Only `SVC-SCHOLARSHIP` is needed for the first working demo.

### Consents (consent-based data sharing)

**`POST /consents`** (citizen) grants consent for a purpose.

```json
// request
{ "service_id": "SVC-SCHOLARSHIP", "departments": ["civil", "education", "land"] }

// 201 response
[
  {
    "consent_id": "CON-0001",
    "citizen_id": "SETU-C001",
    "department": "civil",
    "service_id": "SVC-SCHOLARSHIP",
    "status": "ACTIVE",
    "granted_at": "2026-09-21T10:30:00Z",
    "expires_at": "2026-10-21T10:30:00Z"
  }
]
```

**`GET /consents`** (citizen) → the citizen's own consents (any status).

**`DELETE /consents/{consent_id}`** (citizen) → revokes it. Response `200` with `status: "REVOKED"`.

`status` is one of `ACTIVE`, `REVOKED`, `EXPIRED`. Every grant and revoke is written to the audit log.

### Applications

Status flow:

```
SUBMITTED -> DATA_FETCHING -> DATA_VERIFIED -> UNDER_REVIEW -> APPROVED | REJECTED
                    \-> DATA_ERROR  (department failed, retry possible)
UNDER_REVIEW -> PENDING_INFO -> UNDER_REVIEW
```

**`POST /applications`** (citizen)

```json
// request
{
  "service_id": "SVC-SCHOLARSHIP",
  "consent_ids": ["CON-0001", "CON-0002", "CON-0003"],
  "form_data": { "course_name": "Certified Solar Technician", "training_provider": "MIDC Skill Centre" }
}
```

Behaviour: the request is processed synchronously. The backend checks consent, checks for duplicates, fetches from each department through its adapter, merges into the common schema, and stores everything before responding.

```json
// 201 response
{
  "application_id": "APP-0001",
  "service_id": "SVC-SCHOLARSHIP",
  "citizen_id": "SETU-C001",
  "status": "UNDER_REVIEW",
  "submitted_at": "2026-09-21T10:30:00Z",
  "sla_due_at": "2026-09-28T10:30:00Z",
  "sla_breached": false,
  "profile": { "...": "SETU Citizen Profile from section 2" },
  "partial": false,
  "failed_departments": []
}
```

If **some** departments fail but at least one succeeds: `201` with `status: "DATA_ERROR"`, `partial: true` and `failed_departments: ["land"]`. If none respond: `503 DEPARTMENT_UNAVAILABLE`.

Errors specific to this route: `CONSENT_REQUIRED`, `DUPLICATE_SUBMISSION`, `VALIDATION_ERROR`.

**`GET /applications`**

- Citizen: only their own.
- Official/admin: all. Optional query params: `status`, `service_id`, `sla_breached` (true/false), `limit` (default 50).

Each item is a summary:

```json
{
  "application_id": "APP-0001",
  "service_id": "SVC-SCHOLARSHIP",
  "citizen_id": "SETU-C001",
  "citizen_name": "Asha Ramesh Patil",
  "status": "UNDER_REVIEW",
  "submitted_at": "2026-09-21T10:30:00Z",
  "sla_due_at": "2026-09-28T10:30:00Z",
  "sla_breached": false
}
```

**`GET /applications/{application_id}`** → summary fields plus:

```json
{
  "form_data": { "course_name": "...", "training_provider": "..." },
  "profile": { "...": "SETU Citizen Profile" },
  "timeline": [
    { "status": "SUBMITTED", "at": "2026-09-21T10:30:00Z", "note": "Application received" },
    { "status": "DATA_VERIFIED", "at": "2026-09-21T10:30:02Z", "note": "Data fetched from 3 departments" },
    { "status": "UNDER_REVIEW", "at": "2026-09-21T10:30:02Z", "note": "Assigned to reviewing officer" }
  ],
  "trace": [
    {
      "step": 1,
      "system": "civil",
      "action": "FETCH",
      "format_in": "JSON",
      "format_out": "SETU v1",
      "duration_ms": 120,
      "status": "OK"
    }
  ]
}
```

`trace` powers the **integration flow visualizer**. One entry per step: consent check, each department fetch and conversion, merge, and store. `status` is `OK`, `FAILED` or `SKIPPED`. Citizens can only read their own; officials/admin can read any.

**`POST /applications/{application_id}/decision`** (official, admin)

```json
{ "decision": "APPROVE", "remarks": "All documents verified" }
```

`decision` is `APPROVE`, `REJECT` or `REQUEST_INFO`. Returns the updated application detail. Creates a notification for the citizen and an audit entry.

**`POST /applications/{application_id}/retry`** (citizen, official, admin) re-runs the department fetch for an application in `DATA_ERROR`. Returns the updated application detail.

### Unified profile (consolidated view)

**`GET /citizens/me/profile`** (citizen) → the SETU Citizen Profile built from the citizen's active consents.

**`GET /citizens/{citizen_id}/profile`** (official, admin) → the profile built from the most recent successful data for that citizen, with the same shape as above.

### Notifications (event-driven)

An event is created on each status change, and each event creates a notification for the citizen.

**`GET /notifications`** → the current user's notifications, newest first.

```json
[
  {
    "notification_id": "NOT-0001",
    "application_id": "APP-0001",
    "message_code": "APPLICATION_APPROVED",
    "message": "Your application APP-0001 has been approved",
    "created_at": "2026-09-22T09:00:00Z",
    "read": false
  }
]
```

`message_code` values: `APPLICATION_RECEIVED`, `DATA_VERIFIED`, `DATA_ERROR`, `INFO_REQUESTED`, `APPLICATION_APPROVED`, `APPLICATION_REJECTED`, `CONSENT_GRANTED`, `CONSENT_REVOKED`. The frontend translates by `message_code`.

**`POST /notifications/{notification_id}/read`** → `200` with the updated notification.

### Audit log

**`GET /audit`** (official, admin)

Query params (all optional): `application_id`, `actor`, `action`, `limit` (default 100).

```json
[
  {
    "audit_id": "AUD-0001",
    "at": "2026-09-21T10:30:01Z",
    "actor": "asha",
    "actor_role": "citizen",
    "action": "DATA_FETCH",
    "system": "education",
    "application_id": "APP-0001",
    "outcome": "OK",
    "detail": "Fetched via education adapter (XML -> SETU v1)"
  }
]
```

`action` values: `LOGIN`, `CONSENT_GRANT`, `CONSENT_REVOKE`, `APPLICATION_SUBMIT`, `DATA_FETCH`, `DATA_MERGE`, `DECISION`, `RETRY`, `SIMULATION_CHANGE`. `outcome` is `OK` or `FAILED`.

### Monitoring (admin only)

**`GET /admin/stats`**

```json
{
  "total_applications": 42,
  "by_status": { "UNDER_REVIEW": 12, "APPROVED": 24, "REJECTED": 3, "DATA_ERROR": 3 },
  "avg_processing_hours": 18.5,
  "sla_compliance_percent": 91.0,
  "duplicates_blocked": 5,
  "departments_connected": 3
}
```

**`GET /admin/health`** → status of each connected department.

```json
[
  { "department": "civil", "format": "JSON", "status": "UP", "latency_ms": 118, "success_rate_percent": 100.0, "last_checked": "2026-09-21T10:35:00Z" },
  { "department": "education", "format": "XML", "status": "UP", "latency_ms": 210, "success_rate_percent": 98.5, "last_checked": "2026-09-21T10:35:00Z" },
  { "department": "land", "format": "CSV", "status": "DEGRADED", "latency_ms": 3150, "success_rate_percent": 90.0, "last_checked": "2026-09-21T10:35:00Z" }
]
```

`status` is `UP`, `DEGRADED` or `DOWN`.

**`GET /admin/data-quality`** → list of `quality_flags` (section 2) across all profiles, each with `citizen_id`, `application_id` and `at`.

**`POST /admin/departments/{department}/simulate`** switches a mock department's behaviour, so the demo can show exception handling.

```json
{ "mode": "DOWN" }
```

`mode` is one of:

| Mode | Mock department behaves as |
|---|---|
| `NORMAL` | Works normally (default) |
| `SLOW` | Waits about 3 seconds before answering |
| `DOWN` | Returns HTTP 503 |
| `BAD_DATA` | Returns a record with a missing or invalid field, producing a `quality_flag` |

Returns `200` with `{ "department": "land", "mode": "DOWN" }`.

---

## 4. Frontend integration notes (Dev B)

- Read the base URL from `VITE_API_BASE_URL`. Do not hard-code it.
- Store the token after login and send it as a Bearer token. On any `401`, send the user back to login.
- Show the API's `error.code` values as user-friendly messages. The frontend owns all translations.
- Build against the `/docs` page and the example responses here until the backend is deployed.
- Status colours suggestion: `APPROVED` green, `REJECTED` red, `UNDER_REVIEW` blue, `DATA_ERROR` and `PENDING_INFO` amber.

---

## 5. Mock department routes (used only by the SETU adapters)

These stand in for existing department systems. **The frontend never calls them.** Each has a different format, field names, ID and date format on purpose. No auth, since they simulate separate systems.

### Civil Registration: `GET /mock/civil/{registration_no}` (JSON)

```json
{
  "registration_no": "BR-2004-00123",
  "child_name": "Asha Patil",
  "father_name": "Ramesh Patil",
  "dob": "14-03-2004",
  "place_of_birth": "Aurangabad"
}
```

Date format is `DD-MM-YYYY`.

### Education: `GET /mock/education/{student_id}` (XML, `application/xml`)

```xml
<Student>
  <StudentID>STU-77821</StudentID>
  <Name>PATIL ASHA R</Name>
  <DateOfBirth>2004/03/14</DateOfBirth>
  <Qualification>HSC</Qualification>
  <Percentage>82.5</Percentage>
  <Board>Maharashtra State Board</Board>
  <PassingYear>2022</PassingYear>
</Student>
```

Name is upper-case with initials. Date format is `YYYY/MM/DD`.

### Land Records: `GET /mock/land/{khata_no}` (CSV, `text/csv`)

```csv
khata_no,owner_name,survey_no,area_acres,annual_income_inr,district
LR-55210,Asha Ramesh Patil,112/4,4.0,180000,Aurangabad
```

Area is in **acres**. The adapter converts to hectares (`hectares = acres × 0.4047`).

### Adapter mapping summary

| SETU field | Civil (JSON) | Education (XML) | Land (CSV) |
|---|---|---|---|
| `name` | `child_name` (merged with father's name) | `Name` (normalised to title case) | `owner_name` (used as the reference) |
| `dob` | `dob` (`DD-MM-YYYY` → ISO) | `DateOfBirth` (`YYYY/MM/DD` → ISO) | – |
| `birth_place` | `place_of_birth` | – | – |
| `education.*` | – | `Qualification`, `Percentage`, `Board`, `PassingYear` | – |
| `land.survey_no` | – | – | `survey_no` |
| `land.area_hectares` | – | – | `area_acres` × 0.4047 |
| `land.district` | – | – | `district` |
| `income_annual_inr` | – | – | `annual_income_inr` |

If `dob` differs between civil and education, keep the civil value and add a `MISMATCH` quality flag.

---

## 6. Seed data (Dev C)

Put these files in `/data`:

| File | Content |
|---|---|
| `master_registry.json` | 10 citizens, SETU ID mapped to the three department IDs |
| `civil.json` | 10 records in the civil format above |
| `education.xml` | 10 `<Student>` records inside one root element |
| `land.csv` | 10 rows in the land format above |
| `applications_seed.json` | 15–20 past applications with mixed statuses and dates, some past their SLA, so the dashboard has data |

Keep the first two citizens (`SETU-C001` Asha Patil, `SETU-C002` Rohan Deshmukh) matching the demo users. Make one or two records deliberately imperfect (for example a slightly different date of birth) so the `MISMATCH` flag has something to show.

---

## 7. Change log

| Version | Date | Change |
|---|---|---|
| v1 | 2026-09-21 | First draft |