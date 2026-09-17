# ?? CuraDesk - Clinic Front Desk & Conflict-Free Appointment Operating System

A full-stack clinic scheduling and front desk management platform with **mathematical interval overlap detection**, **fair cancellation management**, **doctor day timeline views**, **instant patient lookup**, and **three automated operational challenge twists**.

Built for Round 2 ("Builder" Round) of the Auriga IT Recruitment Drive.

---

## ?? Core System Highlights

- **Zero Overlap Guarantee**: Rigorous mathematical interval collision engine preventing double-booking any doctor (`s1 < e2 AND s2 < e1`) while explicitly permitting back-to-back touching-boundary slots.
- **Fair Cancellation Policy**: Automated 24-hour lead time cutoff. Appointments cancelled in good time (&ge; 24h prior) are **100% free**, while late cancellations (&lt; 24h notice) incur a small, fair fee ($25.00) to protect doctor productivity. Cancelled slots immediately open up for other patients.
- **High-Velocity Doctor's Day Workstation**: Dynamic timeline displaying each doctor's daily agenda chronologically with status badges and performance metrics.
- **Universal Patient Name Search**: Case-insensitive substring search matching patient first or last name across historical and upcoming appointments.
- **Official Problem Twists**:
  - **Level 1 ? T6 (lifecycle)**: Reschedule an appointment to a new time while strictly preserving patient and doctor invariant, re-verifying overlap against the doctor's schedule while excluding the appointment itself from collision detection.
  - **Level 2 ? T1 (integrate)**: Dispatches morning reminders for today's active appointments to patients via the Notification Service (`/outbox`), triggered by `POST /clock`.
  - **Level 3 ? T2 (automation)**: Automated background job that auto-marks appointments as `no_show` 30 minutes after their start time if not completed, triggered and evaluated via `POST /clock`.
- **Modern Responsive Web UI**: Tailwind CSS interface with Doctor's Day agenda, 1-click booking modal with live conflict detection, fair cancellation fee preview, and clock simulation controls.
- **Automated Test Suite**: 33 comprehensive unit, integration, twist, and API tests with **100% pass rate**.

---

## ?? Quickstart & Setup Guide

### 1. Prerequisites
- Python 3.8+ (tested on Python 3.13)
- `pip` package manager

### 2. Clone Repository
```bash
git clone https://github.com/pranjaljain0905-wq/clinic-frontdesk.git
cd clinic-frontdesk
```

### 3. Install Dependencies
```bash
pip install -e .
# Or install directly:
pip install flask python-dateutil pytest
```

### 4. Run the Web Application
```bash
python web/app.py
```
Open your browser at **[http://localhost:5000](http://localhost:5000)** (or [http://localhost:5000/desk](http://localhost:5000/desk)).

---

## ?? Running Automated Tests

Run the full pytest suite:
```bash
python -m pytest -v
```

### Running Specific Test Suites:
```bash
# Conflict-free booking & overlap edge cases
python -m pytest tests/test_booking.py -v

# Fair cancellation cutoff & fee calculation
python -m pytest tests/test_cancellation.py -v

# Doctor day agenda & patient name lookup
python -m pytest tests/test_lookups.py -v

# Level 1 Twist (Lifecycle Rescheduling)
python -m pytest tests/test_twist_level1.py -v

# Level 2 Twist (Morning Notifications via /clock & /outbox)
python -m pytest tests/test_twist_level2.py -v

# Level 3 Twist (Automated No-Show Marker 30 min after start)
python -m pytest tests/test_twist_level3.py -v

# REST API & Web view integration tests
python -m pytest tests/test_api_and_web.py -v
```

---

## ?? REST API Endpoints Specification

| Method | Endpoint | Description | Status Codes |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/appointments` | Book a new appointment (conflict-checked) | `201 Created`, `409 Conflict`, `400 Bad Request` |
| `GET` | `/api/appointments` | Query appointments by doctor, date, or status | `200 OK` |
| `GET` | `/api/appointments/<id>` | Fetch appointment details by ID | `200 OK`, `404 Not Found` |
| `POST` | `/api/appointments/<id>/reschedule` | **Level 1 (T6)**: Reschedule appointment to new time | `200 OK`, `409 Conflict`, `400 Bad Request` |
| `POST` | `/api/appointments/<id>/cancel` | Cancel appointment with fair fee calculation | `200 OK`, `400 Bad Request` |
| `POST` | `/api/appointments/<id>/complete` | Mark appointment completed by doctor | `200 OK`, `400 Bad Request` |
| `GET` | `/api/doctors` | List all clinic doctors | `200 OK` |
| `GET` | `/api/doctors/<id>/day?date=YYYY-MM-DD` | **Desk View**: Doctor's daily schedule & stats | `200 OK`, `404 Not Found` |
| `GET` | `/api/appointments/search?name=<query>` | **Desk View**: Patient search by name | `200 OK` |
| `POST` | `/clock` | **Levels 2 & 3**: Morning reminders + auto no-show | `200 OK`, `400 Bad Request` |
| `GET` | `/outbox` | **Level 2**: Notification Outbox query | `200 OK` |
| `DELETE`| `/outbox` | Clear notification outbox | `200 OK` |
| `GET` | `/api/stats` | Overall clinic performance metrics | `200 OK` |

---

## ?? Twist Implementations Detail

### Level 1 ? T6 (lifecycle): Conflict-Free Rescheduling
- **Endpoint**: `POST /api/appointments/<id>/reschedule`
- **Behavior**:
  - Doctor ID and patient identity are strictly immutable.
  - Checks interval collision against the doctor's existing active appointments.
  - **Self-Exclusion**: Explicitly ignores `<id>` during collision checking so an appointment can be extended or shifted by 15 minutes without falsely colliding with itself.
  - On conflict: Rejects with `409 Conflict` and preserves the original booking undisturbed.

### Level 2 ? T1 (integrate): Morning Reminders via Notification Outbox
- **Endpoints**: `POST /clock` &rarr; `GET /outbox`
- **Behavior**:
  - When `POST /clock` is triggered with `{"date": "YYYY-MM-DD"}` (or timestamp), queries all active (`booked`) appointments for that date.
  - Generates personalized morning appointment reminders in the `/outbox`.
  - Idempotent: Subsequent clock calls for the same date do not send duplicate reminders.
  - Cancelled appointments are excluded from reminders.

### Level 3 ? T2 (automation): Auto-Mark No-Shows 30 Min After Start
- **Endpoint**: `POST /clock` (evaluated on clock advancement)
- **Behavior**:
  - Whenever the clock advances to timestamp $T$, the job scans all appointments with `status == 'booked'`.
  - Condition: $	ext{Current Clock Time} \ge 	ext{Start Time} + 30	ext{ minutes}$.
  - Any appointment meeting this condition is automatically marked as `no_show`.
  - Completed and cancelled appointments are untouched.

---

## ?? Author
- **Pranjal Jain** ([@pranjaljain0905-wq](https://github.com/pranjaljain0905-wq))
