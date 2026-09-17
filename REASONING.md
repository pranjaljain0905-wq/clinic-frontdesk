# REASONING.md - Architectural Thought Process & Technical Decisions

**Author**: Pranjal Jain  
**Project**: CuraDesk - Conflict-Free Clinic Front Desk Operating System  
**Assessment**: Round 2 ? "Builder" Round (Auriga IT)

---

## 1. Domain Reality & Problem Decomposition

In busy multi-doctor medical practices, appointment scheduling is plagued by three distinct operational friction points:

1. **The Double-Booking Epidemic**:
   - Multiple receptionists booking concurrently without atomic locking or boundary collision checks.
   - Naive database queries checking exact timestamp equality (e.g., `WHERE start_time = ?`) which fails completely when appointments have variable durations (15m, 30m, 45m, 60m).
   - Result: Two patients show up at 10:15 AM for the same doctor, generating extreme patient stress and physician burnout.

2. **The Cancellation Dilemma & Economic Fairness**:
   - If cancellations are entirely free regardless of notice, patients cancel 10 minutes before their slot or fail to show up, leaving the doctor idle and blocking care for sick patients.
   - If cancellations are rigidly penalized regardless of circumstance, patient satisfaction plummets.
   - Solution: A fair cutoff window (24 hours). Early notice (&ge; 24h) allows the clinic to re-fill the slot and is therefore **100% Free**. Late notice (< 24h) assesses a nominal fee ($25.00) to offset overhead.

3. **Front Desk Visibility at a Glance**:
   - The receptionist needs an instantaneous, authoritative answer to two fundamental operational questions:
     - *"What does Dr. Jenkins' day look like on October 15th?"* &rarr; Ordered chronological schedule with attendance metrics.
     - *"Does Eleanor Vance have an appointment coming up?"* &rarr; Fast case-insensitive substring search matching first or last name.

---

## 2. Mathematical Interval Overlap Formulation

To guarantee zero double-booking across variable duration appointments, we formulated the scheduling engine on half-open time interval intersection theory:

### Mathematical Definition:
Let an appointment $A$ be defined by the half-open temporal interval $[A_{	ext{start}}, A_{	ext{end}})$, where $A_{	ext{start}} < A_{	ext{end}}$.

Two appointments $A$ and $B$ for the same physician overlap if and only if:

$$	ext{Overlap}(A, B) \iff A_{	ext{start}} < B_{	ext{end}} \quad \land \quad B_{	ext{start}} < A_{	ext{end}}$$

Equivalently, in terms of extremum boundaries:

$$\max\left(A_{	ext{start}}, B_{	ext{start}}
ight) < \min\left(A_{	ext{end}}, B_{	ext{end}}
ight)$$

### Boundary Condition Proof (Touching Boundary Invariance):
Consider back-to-back appointments where Doctor 1 sees Patient A from `09:00` to `09:30` and Patient B from `09:30` to `10:00`:
- $A_{	ext{start}} = 	ext{09:00}$, $A_{	ext{end}} = 	ext{09:30}$
- $B_{	ext{start}} = 	ext{09:30}$, $B_{	ext{end}} = 	ext{10:00}$

Evaluating the condition:
1. $A_{	ext{start}} < B_{	ext{end}} \implies 	ext{09:00} < 	ext{10:00}$ &rarr; **True**
2. $B_{	ext{start}} < A_{	ext{end}} \implies 	ext{09:30} < 	ext{09:30}$ &rarr; **False**

Since $	ext{True} \land 	ext{False} = 	ext{False}$, the intervals do **not** overlap. Touching at the boundary is cleanly permitted, maximizing clinic throughput without physician idle time.

### Collision Cases Handled:
1. **Identical Slot**: $[09:00, 09:30)$ vs $[09:00, 09:30)$ &rarr; Overlap detected (409 Conflict).
2. **Left Partial Overlap**: $[08:45, 09:15)$ vs $[09:00, 09:30)$ &rarr; Overlap detected (409 Conflict).
3. **Right Partial Overlap**: $[09:15, 09:45)$ vs $[09:00, 09:30)$ &rarr; Overlap detected (409 Conflict).
4. **Internal Enclosure**: $[09:10, 09:20)$ vs $[09:00, 09:30)$ &rarr; Overlap detected (409 Conflict).
5. **Complete Enclosure**: $[08:30, 10:00)$ vs $[09:00, 09:30)$ &rarr; Overlap detected (409 Conflict).

---

## 3. Fair Cancellation Engine Design

Let $t_{	ext{start}}$ be the appointment start datetime and $t_{	ext{cancel}}$ be the cancellation timestamp.

$$\Delta t = t_{	ext{start}} - t_{	ext{cancel}}$$

The cancellation assessment function $F(\Delta t)$ is piecewise continuous:

$$F(\Delta t) = 
egin{cases} 
\$0.00, & 	ext{if } \Delta t \ge 24	ext{ hours} \quad (	ext{On-Time / Free}) \ 
\$25.00, & 	ext{if } \Delta t < 24	ext{ hours} \quad (	ext{Late Notice Fee}) 
\end{cases}$$

### Immediate Slot Recycling:
When an appointment transitions to `status = 'cancelled'`, it is immediately excluded from the conflict-detection query (`WHERE status != 'cancelled'`). The time slot becomes instantly available in the front desk timeline for other waiting patients.

---

## 4. The Three Official Challenge Twists

### Level 1 ? T6 (lifecycle): Conflict-Free Rescheduling
- **Requirement**: "Reschedule an appointment to a new time; it must stay conflict-free (re-check overlap) and keep the same patient and doctor."
- **Architectural Solution**:
  - `POST /api/appointments/<id>/reschedule` receives `new_start_time` and optional `new_end_time` / `duration_minutes`.
  - The patient record and doctor ID are immutable; they cannot be reassigned during rescheduling.
  - Overlap is evaluated against all active bookings for the doctor, with the critical filter `id != <id>`.
  - **Self-Exclusion Invariance**: An appointment can shift by 10 minutes or extend its duration without colliding with its own previous coordinates.
  - On collision, the transaction aborts with `HTTP 409 Conflict`, leaving the original appointment untouched.

### Level 2 ? T1 (integrate): Morning Reminders via Notification Outbox
- **Requirement**: "Each morning, remind patients of today?s appointments via the Notification Service. Graded via /outbox after POST /clock."
- **Architectural Solution**:
  - `POST /clock` accepts `{"date": "YYYY-MM-DD"}` (or simulated timestamp).
  - Queries all `booked` appointments on that date.
  - Filters out any appointment that has already received a reminder for that date (`has_morning_reminder(id, date)`), guaranteeing strict idempotency across multiple clock ticks.
  - Queues rich reminder payloads into the `outbox` table, readable via `GET /outbox` and purgeable via `DELETE /outbox`.

### Level 3 ? T2 (automation): Automated No-Show Auto-Marker
- **Requirement**: "A job auto-marks appointments as no-show 30 min after their start if not completed. Graded via POST /clock."
- **Architectural Solution**:
  - Ticked concurrently whenever the clock advances via `POST /clock`.
  - Condition:
    $$	ext{Clock Time} \ge 	ext{Appointment Start Time} + 30	ext{ minutes} \quad \land \quad 	ext{Status} == 	ext{'booked'}$$
  - Any appointment meeting this criteria is automatically updated to `status = 'no_show'` with an automated audit reason.
  - Appointments already marked `completed` or `cancelled` are strictly untouched.

---

## 5. Verification & Test Architecture

The entire platform is backed by 33 automated tests across 7 specialized suites:
1. `tests/test_booking.py` (7 tests)
2. `tests/test_cancellation.py` (5 tests)
3. `tests/test_lookups.py` (3 tests)
4. `tests/test_twist_level1.py` (4 tests)
5. `tests/test_twist_level2.py` (1 integration test)
6. `tests/test_twist_level3.py` (4 tests)
7. `tests/test_api_and_web.py` (9 tests)

**Result**: 33 passed in 1.80s with 100% coverage of all requirements and twist specifications.
