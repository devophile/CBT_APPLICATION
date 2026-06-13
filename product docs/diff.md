# 🔍 Validation Report — Phase Plans vs Product & Technical Specs

> **Reference Documents:**
> - `product_feature.md` v1.2 — Business rules & workflows (Source of Truth)
> - `technical_prd.md` v2.0 — Architecture, schema, API specs (Source of Truth)
> - `phase1_plan.md` — MVP implementation plan
> - `phase2_plan.md` — Growth features plan
> - `phase3_plan.md` — Scale & infrastructure plan
>
> **Validation Date:** 2026-06-13
> **Validator:** Antigravity AI — Deep cross-document analysis

---

## 📊 Summary Scorecard

| Phase | Critical Errors | Key Gaps | Warnings | Risk Level |
|---|---|---|---|---|
| Phase 1 | 5 | 8 | 6 | 🔴 HIGH |
| Phase 2 | 3 | 5 | 4 | 🟠 MEDIUM |
| Phase 3 | 2 | 4 | 3 | 🟡 LOW-MEDIUM |

---

## 🔴 PHASE 1 — Critical Errors & Key Gaps

### ❌ CRITICAL ERROR 1 — `image_public_id` Column Missing from Question Model

**Location:** `phase1_plan.md` → Task 3, `add_question()` function vs `phase2_plan.md` Task 1 § 1.2 Important note

**Issue:**
Phase 2 (Task 1 §1.2) explicitly requires:
> *"Store the Cloudinary `public_id` alongside `image_url` in the `Question` model (add `image_public_id: Mapped[Optional[str]]` column)"*

However, **Phase 1 creates the Question model without `image_public_id`**. Phase 1 only includes `image_url`. Since Phase 1 must create ALL tables and columns upfront (to avoid breaking migrations later), this column must be added to the Phase 1 schema.

**Impact:** Phase 2 will **require a new Alembic migration** to add this column — directly contradicting the documented guarantee:
> *"Zero database migrations needed [in Phase 2]. All tables and columns for Phase 2 features were created in Phase 1."*

**Fix:** Add `image_public_id: Mapped[Optional[str]] = mapped_column(String(500))` to the `Question` model in Phase 1's `app/models/exam.py`. Also update the ER diagram in `technical_prd.md § 4.0`.

---

### ❌ CRITICAL ERROR 2 — `STORAGE_PROVIDER` & `SENTRY_DSN` Env Vars Missing from Phase 1 Config

**Location:** `phase1_plan.md` § 1.2 `config.py` vs `phase2_plan.md` Task 1 `storage_service.py` vs `phase3_plan.md` Task 2

**Issue:**
- Phase 2's `storage_service.py` calls `getattr(settings, "STORAGE_PROVIDER", "cloudinary")` — but `STORAGE_PROVIDER` is never added to `Settings` in any Phase 1 or Phase 2 config definition.
- Phase 3's `sentry.py` references `settings.SENTRY_DSN` — never defined in `Settings`.

The Phase 1 `config.py` shown in the plan only has: `ENV, PORT, DATABASE_URL, DATABASE_URL_SYNC, JWT_ACCESS_SECRET, JWT_REFRESH_SECRET, ADMIN_EMAIL, ADMIN_PASSWORD, SMTP_*, FRONTEND_URL`. Cloudinary vars exist in `technical_prd.md § 11` but are absent from Phase 1's plan code.

**Impact:** Runtime `AttributeError` on first Cloudinary upload call in Phase 2. Sentry silently fails in Phase 3 if DSN not defined.

**Fix:** In Phase 1's `config.py`, include all future-phase env vars with safe defaults:
```python
CLOUDINARY_CLOUD_NAME: str = ""
CLOUDINARY_API_KEY: str = ""
CLOUDINARY_API_SECRET: str = ""
STORAGE_PROVIDER: str = "cloudinary"
REDIS_URL: str = ""
SENTRY_DSN: str = ""
```
This is already done in `technical_prd.md § 11` but **missing from Phase 1's concrete implementation code**.

---

### ❌ CRITICAL ERROR 3 — `password` Field for `change_password` vs OTP Purpose Mismatch

**Location:** `phase1_plan.md` Task 1 `auth_service.py` vs `product_feature.md § 4.1, 5.1`

**Issue:**
`product_feature.md` states teachers and students can **"Change Password → OTP email verification required"** (§ 5.1). The `technical_prd.md § 5.1` API table lists `POST /auth/change-password` with body `{ current_password, new_password }` — **no OTP**.

However, `product_feature.md` says:
> *"Profile → Change Password → OTP email verification required."*

The `Otp` model in the code supports `purpose: "change_password"`, but the API table in `technical_prd.md § 5.1` shows `change-password` as a single endpoint requiring only `current_password + new_password` (no OTP step).

**Impact:** Change password flow is ambiguous — developers will implement it differently depending on which document they follow. One implementation path will be wrong by design.

**Fix:** Clarify in Phase 1 whether change-password requires OTP or just current password. If OTP is required, the flow needs:
1. `POST /auth/change-password/initiate` → sends OTP
2. `POST /auth/change-password/confirm` → verifies OTP + sets new password

---

### ❌ CRITICAL ERROR 4 — Race Condition in `ExamCostConfig` Seeding vs Multi-worker Production

**Location:** `phase1_plan.md` Task 1 DB setup, `technical_prd.md § 4.2` seed script vs `phase3_plan.md` Task 5 Production Docker (4 workers)

**Issue:**
The seed script uses:
```python
SEED_CONFIG_ID = uuid.UUID("00000000-0000-0000-0000-000000000001")
existing = await session.get(ExamCostConfig, SEED_CONFIG_ID)
if not existing:
    # create config
```

Phase 3 runs with `--workers 4` (Uvicorn). The start command in Render/Docker is:
```
alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port $PORT
```

The seed is run **separately** via `python -m scripts.seed`, not in the start command. If the seed is never run, `ExamCostConfig` table is empty. The `start_exam` function does:
```python
config = await db.execute(select(ExamCostConfig))
config = config.scalar_one()
```
This will throw a `NoResultFound` exception if the config table is empty, **crashing every exam start**.

**Fix:** Either:
1. Include `python -m scripts.seed` in the Docker start command: `alembic upgrade head && python -m scripts.seed && uvicorn ...`
2. Or, make `get_exam_cost_config()` fail gracefully with a human-readable 500 if config is missing, and document seeding as mandatory in deployment checklist.

---

### ❌ CRITICAL ERROR 5 — `conftest.py` in Phase 1 vs Phase 3 Conflicts (Test Database Isolation)

**Location:** `phase1_plan.md` Task 5 §5.3 conftest vs `phase3_plan.md` Task 2 §2.2 conftest

**Issue:**
Phase 1's conftest:
```python
async def db_session():
    async with async_session_factory() as session:
        yield session  # Uses REAL production session factory
```

Phase 3's conftest (correctly):
```python
test_engine = create_async_engine(settings.DATABASE_URL, echo=False)
TestSessionLocal = async_sessionmaker(test_engine, ...)

async def db_session():
    async with TestSessionLocal() as session:
        yield session
        await session.rollback()  # Rolls back after each test
```

Phase 1's conftest **does not roll back** and **does not use a separate test engine**. Any test that writes data will pollute subsequent tests. This is a testing hygiene error.

**Impact:** Tests pass in isolation but fail when run as a full suite (`pytest tests/`). Makes CI/CD unreliable from the start.

**Fix:** Use Phase 3's conftest pattern from Day 1. The `db_session` fixture must rollback after each test.

---

### 🔑 KEY GAP 1 — Admin Password Reset Endpoint Missing from Phase 1 Tasks

**Location:** `product_feature.md § Appendix` vs `phase1_plan.md` Task 2, `technical_prd.md § 5.2`

**Issue:**
`product_feature.md` Appendix specifies:
> *"Admin can reset a teacher's password from the admin panel, which overrides the teacher's current password."*

`technical_prd.md § 5.2` lists: `PATCH /admin/teachers/{id}/reset-password { new_password }`.

Phase 1's Task 2 implementation guide only lists `create_teacher`, `credit_wallet` etc. in `admin_service.py`. The `reset-password` endpoint is **never mentioned in any Phase 1 task** as something to implement.

**Impact:** Teacher password reset from admin panel will be missing at Phase 1 launch.

---

### 🔑 KEY GAP 2 — `is_submitted=False` Attempt Cleanup Strategy Undefined

**Location:** `product_feature.md § 5.5` vs `phase1_plan.md` Task 4

**Issue:**
`product_feature.md § 5.5` states:
> *"If the student closes the browser tab or the device shuts down, all progress is lost and the attempt is recorded as incomplete (no score, but coins are **already deducted**)."*

Phase 1 creates an `ExamAttempt` with `is_submitted=False` when the student starts. If the student never submits, this record stays in the DB as an unsubmitted attempt with `is_submitted=False` and `score=NULL`.

**Problems identified:**
1. The **next time the student opens the exam**, `GET /student/exams/{id}` checks for an existing attempt. The existing `is_submitted=False` attempt will make the API think the student has already attempted the exam and show "View Solution" — **even though they never submitted**.
2. The `UniqueConstraint(exam_id, student_id)` means the student **can never start** the exam again (the constraint blocks a new attempt record).
3. No cleanup/expiry logic for abandoned attempts is defined anywhere.

**Impact:** Students who lose connection mid-exam and never submit are permanently locked out of that exam with a phantom attempt and no results. This is a critical UX and business logic failure.

**Fix:** Define a policy for `is_submitted=False` attempts — options:
- Add a TTL-based background job to purge old unsubmitted attempts
- On `GET /student/exams/{id}`, if attempt is older than `duration_minutes + buffer` and not submitted, treat as "failed/incomplete" but allow viewing the result page with a "timed out" message
- Allow re-attempt if coins were already deducted and attempt was never submitted (requires a product decision)

---

### 🔑 KEY GAP 3 — Teacher Soft-Delete → Exam Visibility Logic Not Implemented

**Location:** `product_feature.md § Appendix` (last row) vs `phase1_plan.md` Task 5 §5.1

**Issue:**
`product_feature.md` Appendix states:
> *"When a teacher is soft-deleted, all their exams become inaccessible to students. Private exams are hidden. Global exams are removed from the public listing."*

Phase 1 Task 5 §5.1 mentions:
> *"Teacher soft-deleted → exams hidden: Create shared `get_active_exams_query()` in `exam_service.py` that auto-applies `is_deleted == False` + **teacher soft-delete join filter**."*

However, **no implementation of this join filter is shown or scaffolded** in Phase 1's concrete code. The `start_exam` flow only checks `Exam.is_deleted == False` — it does **not** join with the `users` table to check if `exam.teacher.is_deleted == False`.

**Impact:** If a teacher is soft-deleted, their global exams remain visible to students and can be attempted. Students can still spend coins on exams from deleted teachers.

---

### 🔑 KEY GAP 4 — Exam Active Window Enforcement Not Implemented

**Location:** `product_feature.md § 5.3` vs `phase1_plan.md` Task 4, `technical_prd.md § 5.5`

**Issue:**
`product_feature.md § 5.3` specifies:
> *"Active exams are sorted by date (upcoming first). Inactive exams appear at the bottom."*

The Exam model has `scheduled_date`, `start_time`, and `end_time` fields. However, nowhere in Phase 1 is there logic that:
1. **Blocks** a student from starting an exam **before** `start_time` on `scheduled_date`
2. **Blocks** a student from starting an exam **after** `end_time` on `scheduled_date`

The `start_exam` service only checks `exam.is_active` — a boolean controlled manually by the teacher. The scheduled time window is never programmatically enforced.

**Impact:** Students can attempt exams at 2 AM when they are scheduled for 10 AM. Exam timing enforcement is entirely dependent on teachers manually toggling `is_active` — which is unreliable.

**Fix:** In `start_exam`, add time-window validation:
```python
from datetime import datetime, date
now = datetime.now()
exam_date = exam.scheduled_date
start_dt = datetime.combine(exam_date, datetime.strptime(exam.start_time, "%H:%M").time())
end_dt = datetime.combine(exam_date, datetime.strptime(exam.end_time, "%H:%M").time())
if now < start_dt:
    raise HTTPException(400, "Exam has not started yet")
if now > end_dt:
    raise HTTPException(400, "Exam window has passed")
```

---

### 🔑 KEY GAP 5 — `created_by` Field in `Transaction` Not Populated for Non-Admin Actions

**Location:** `technical_prd.md` Transaction model vs `phase1_plan.md` Task 4 `start_exam`

**Issue:**
The `Transaction` model has `created_by: Mapped[Optional[uuid.UUID]]` documented as "Audit: admin who issued credit." In `start_exam`, when a coin deduction happens, the `Transaction` is created without setting `created_by` — which is correct (it's a system deduction). But the `credit_wallet` function in `admin_service.py` passes `created_by=uuid.UUID(admin_id)` — which requires the admin JWT token to carry a valid UUID.

**Problem:** The Admin is authenticated via env-var credentials and the JWT payload uses `"admin"` as the `sub` field (a string literal, not a UUID):
```python
access_token = create_access_token("admin", email, "super_admin", session_id)
```
But `created_by` is typed as `UUID(as_uuid=True)`. Calling `uuid.UUID("admin")` will throw a `ValueError`.

**Impact:** Every admin coin credit will fail with a `ValueError` at runtime.

**Fix:** Either:
- Use `None` for admin `created_by` since admin has no UUID, or
- Store admin identifier as a string in a separate `created_by_email` column, or
- Use a fixed sentinel UUID for the admin in the JWT: `create_access_token("00000000-0000-0000-0000-000000000000", ...)`

---

### 🔑 KEY GAP 6 — Batch Assignment Enforcement for Private Exams Missing from Student Exam Listing

**Location:** `product_feature.md § 7.3` vs `phase1_plan.md` Task 4, `technical_prd.md § 5.4`

**Issue:**
`product_feature.md § 7.3` states:
> *"A student who exists in the teacher's student list but is not in any assigned batch for that exam will not see or access the exam."*

`technical_prd.md § 5.4` has `GET /student/exams/my?category`. Phase 1 Task 4 never shows the implementation of this endpoint. The access control logic for "which private exams can a student see" is critical — the filter must join `exam_batches → batch_students → student_id`.

There is no implementation or even a reference to the query for `GET /student/exams/my` in Phase 1's tasks. Only `start_exam` checks batch access — but by then the exam is already visible to the student, which contradicts the spec.

**Fix:** `GET /student/exams/my` must filter private exams through: `exam_batches JOIN batch_students WHERE batch_students.student_id = current_student`.

---

### 🔑 KEY GAP 7 — Teacher `focused_exam` Categories Are Open String (No Enum Validation)

**Location:** `product_feature.md § 3.2 (Focused Zone)` vs `technical_prd.md` TeacherProfile model

**Issue:**
`product_feature.md` lists "Focused Zone" as multi-select: `NEET / JEE / SSC (multiple allowed)`. `technical_prd.md` uses `focused_zones: ARRAY(String)`. The same applies to student's `focused_exam: String(50)`.

Neither the Pydantic schemas nor the model enforce that valid values are limited to `["NEET", "JEE", "SSC"]` (or similar). A teacher could have `focused_zones = ["XYZ", "ABC"]`, and the student's home page carousel would show unknown categories, breaking the category filter.

**Fix:** Use `Enum` or `Literal` validation in the Pydantic schemas:
```python
from typing import Literal, List
class CreateTeacherRequest(BaseModel):
    focused_zones: List[Literal["NEET", "JEE", "SSC", "UPSC", "Banking"]]
```

---

### 🔑 KEY GAP 8 — No Unique Constraint on `(teacher_id, student_id)` in `teacher_students`

**Location:** `technical_prd.md` batch model vs expected behavior

**Issue:**
`teacher_students` has `UniqueConstraint("teacher_id", "invited_email")` — which correctly prevents a teacher from adding the same email twice. But there is no `UniqueConstraint("teacher_id", "student_id")`.

**Scenario:** A teacher adds `student@email.com` (pending). The student registers. The link flips to `active` with `student_id = X`. The teacher then re-adds `student@email.com` — this would fail the `invited_email` unique constraint. However, if a student changes their email between registration and linking (edge case), or if there is a data inconsistency bug, duplicate `(teacher_id, student_id)` rows could exist.

More importantly: when the student registers and multiple teachers have `pending` rows with the same email, all of them should be activated. The current code iterates `pending_links` and sets `student_id` on all of them — but does not handle the case where a student is linked to the same teacher twice (once pending, once a second invite).

**Fix:** Consider adding `UniqueConstraint("teacher_id", "student_id")` on the non-null rows at DB level, or handle it at the service level.

---

### ⚠️ WARNING 1 — `datetime.utcnow()` is Deprecated in Python 3.12

**Location:** Multiple models across all phase plans

**Issue:**
`datetime.utcnow()` is deprecated as of Python 3.12 (the stated stack). Multiple models use:
```python
created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```
This should be:
```python
from datetime import datetime, timezone
created_at: Mapped[datetime] = mapped_column(default=lambda: datetime.now(timezone.utc))
```
Or use SQLAlchemy's server-side `func.now()` instead.

---

### ⚠️ WARNING 2 — Single-Device Admin Session Not Enforced

**Location:** `phase1_plan.md` Task 1 `admin_login` vs `product_feature.md`

**Issue:**
Teacher and student logins enforce single-device sessions via `active_session_id`. But `admin_login` creates a new `session_id` every time and issues a token — without storing or checking `active_session_id` anywhere for admin. There is no `User` record for admin in the DB.

**Impact:** An admin can be simultaneously logged in from multiple devices/browsers. No security concern noted in the docs, but it is a gap vs. the stated session management model.

---

### ⚠️ WARNING 3 — `with_for_update()` on Exam in `start_exam` Blocks Concurrent Reads

**Location:** `technical_prd.md § 5.5` `start_exam`

**Issue:**
```python
select(Exam).options(selectinload(Exam.questions)).where(...).with_for_update()
```
This acquires a **row-level write lock** on the entire Exam row, including loading all questions with `selectinload`. For a popular global exam with 100+ concurrent students starting simultaneously, this serializes ALL exam start requests into a queue, creating a bottleneck.

**Fix for Phase 3:** Use `with_for_update(of=Wallet)` on only the wallet row (not the exam). The exam only needs a lock to set `is_locked = True` on first attempt — this can be done with a conditional update: `UPDATE exams SET is_locked=True WHERE id=? AND is_locked=False`.

---

### ⚠️ WARNING 4 — Frontend Token in SSE URL Query Param (Security Risk)

**Location:** `phase3_plan.md` Task 4 §4.3 `useNotificationStream.js`

**Issue:**
```javascript
const url = `${VITE_API_BASE_URL}/notifications/stream?token=${accessToken}`;
```
Passing JWT as a URL query parameter exposes it in:
- Server access logs
- Browser history
- Network proxies
- Referrer headers

**Fix:** Use `EventSource` via a custom fetch with `Authorization: Bearer` header (requires a polyfill like `eventsource-parser`), or implement token exchange: issue a short-lived SSE-specific token via `POST /notifications/sse-token` and pass that in the query param.

---

### ⚠️ WARNING 5 — No Exam Cost Config Validation (Admin Can Set to 0)

**Location:** `technical_prd.md § 5.2`, `phase1_plan.md` Task 2

**Issue:**
`PATCH /admin/config/exam-cost` has no documented Pydantic validation. Admin can set `cost_per_question = 0`, making all exams free. Admin can set `global_exam_publishing_fee = 0`, removing the publishing gate.

No validation like `cost_per_question >= 0.01` or `global_exam_publishing_fee > 0` is specified anywhere.

**Fix:** Add Pydantic validation in `UpdateExamCostConfigRequest`:
```python
class UpdateExamCostConfigRequest(BaseModel):
    cost_per_question: Decimal = Field(ge=Decimal("0.01"))
    global_exam_publishing_fee: Decimal = Field(ge=Decimal("1"))
```

---

### ⚠️ WARNING 6 — No Rate Limiting in Phase 1 (Stated as Phase 3 Feature)

**Location:** `phase1_plan.md` vs `technical_prd.md § 10 Security Architecture`

**Issue:**
`technical_prd.md § 10` lists "Rate limiting: SlowAPI" as a security measure. Phase 1 intentionally defers this to Phase 3. However, Phase 1 will be deployed to production (`technical_prd.md § 13`: "Phase 1 = Market launch").

Deploying to production with:
- OTP endpoint (no rate limit) → brute-forceable
- Login endpoints (no rate limit) → credential stuffing attacks
- Exam start (no rate limit) → potential coin drain

**Fix:** Implement basic rate limiting in Phase 1 at minimum for auth endpoints. SlowAPI with in-memory storage works without Redis.

---

## 🟠 PHASE 2 — Critical Errors & Key Gaps

### ❌ CRITICAL ERROR 1 — Cloudinary `upload_image` is Synchronous in Async Service

**Location:** `phase2_plan.md` Task 1 `storage_service.py`

**Issue:**
```python
async def upload_image(self, file_bytes: bytes, folder: str = "questions") -> Tuple[str, str]:
    result = cloudinary.uploader.upload(...)  # ← BLOCKING SYNC CALL!
    return result["secure_url"], result["public_id"]
```

`cloudinary.uploader.upload()` is a **synchronous blocking function**. Calling it inside an `async` function without `asyncio.to_thread()` or `run_in_executor()` will **block the entire event loop** for the duration of the upload.

**Impact:** During a Cloudinary upload, ALL other requests to the FastAPI server will be frozen. With large images and slow connections, this could freeze the API for several seconds.

**Fix:**
```python
import asyncio
async def upload_image(self, file_bytes: bytes, folder: str = "questions"):
    result = await asyncio.to_thread(
        cloudinary.uploader.upload,
        file_bytes,
        folder=f"cbt_platform/{folder}",
        ...
    )
    return result["secure_url"], result["public_id"]
```

Similarly, `cloudinary.uploader.destroy()` in `delete_image()` must also use `asyncio.to_thread()`.

---

### ❌ CRITICAL ERROR 2 — `analytics_service.get_exam_attempts()` Referenced but Never Defined

**Location:** `phase2_plan.md` Task 5 §5.2 CSV Export endpoint

**Issue:**
```python
# In export_exam_csv endpoint:
attempts = await analytics_service.get_exam_attempts(exam_id, current_user.id, db)
```

`analytics_service` in Phase 2 only defines:
- `get_exam_analytics()`
- `get_student_performance()`
- `get_revenue_analytics()`

`get_exam_attempts()` is called in the CSV export but **never defined**. The closest is `get_exam_analytics()` which returns analytics aggregates, not a list of attempts.

**Impact:** CSV export endpoint will throw `AttributeError: module 'analytics_service' has no attribute 'get_exam_attempts'` at runtime.

**Fix:** Either:
1. Define `get_exam_attempts()` in `analytics_service.py` that returns the raw attempt list
2. Or reuse the `get_exam_analytics()` return value's `"attempts"` key

---

### ❌ CRITICAL ERROR 3 — Bilingual Exam Language Validation is Incomplete

**Location:** `phase2_plan.md` Task 2, `technical_prd.md § 5.3` Warning

**Issue:**
`technical_prd.md § 5.3` warns:
> *"When `PATCH /teacher/exams/{id}` changes the `language` field from `single` to `bilingual`, the service layer must validate that ALL existing questions have non-null Bengali fields."*

Phase 2's Task 2 only adds validation to `add_question()`:
```python
if exam.language == ExamLanguage.bilingual:
    if not all([data.qn_bn, data.op_bn_1, ...]):
        raise ApiError(400, "Bengali fields required for bilingual exams")
```

But **the exam language update validation** (checking existing questions when switching `single → bilingual`) is **not implemented anywhere** in Phase 2's task list. The `PATCH /teacher/exams/{id}` update handler has no such validation.

**Impact:** A teacher with a 100-question English-only exam can switch it to `bilingual`, and students will see blank/null Bengali content during the exam.

---

### 🔑 KEY GAP 1 — Analytics Endpoints Not Registered in the Router

**Location:** `phase2_plan.md` Task 3 vs Phase 1's router structure

**Issue:**
Phase 2 creates `app/services/analytics_service.py` and `app/api/analytics.py`, but the analytics endpoints are listed under:
- `GET /teacher/exams/{id}/analytics` — should this be in `app/api/teacher.py` or `app/api/analytics.py`?
- `GET /admin/analytics/revenue` — under admin or analytics router?

Phase 2 never shows the router registration in `app/api/router.py`. The `router.py` from Phase 1 only has: `/auth`, `/admin`, `/teacher`, `/student`, `/exam`, `/wallet`. Adding analytics requires updating `router.py` — not mentioned.

**Fix:** Phase 2 Task 3 must include: Add `from app.api import analytics; api_router.include_router(analytics.router)` in `app/api/router.py`.

---

### 🔑 KEY GAP 2 — Notification System Lacks "Admin Credits Coins" & "Teacher Gifts Coins" Notifications

**Location:** `phase2_plan.md` Task 4 Notification triggers table vs `product_feature.md § 6.4`

**Issue:**
Phase 2's notification triggers table says:
| Event | Notify | Email? |
|---|---|---|
| Admin credits coins | Recipient | ❌ |
| Teacher gifts coins to student | Student | ❌ |

These in-app notifications are listed as required, but **Phase 2 Task 4 never implements the code to trigger these notifications** in `admin_service.py` or `teacher_service.py`. Only the low-balance notification (which exists since Phase 1's `start_exam`) gets Phase 2 UI treatment.

The `student_service.py MODIFY` in the task list only mentions "trigger notification on low balance" — which is already done in Phase 1.

---

### 🔑 KEY GAP 3 — Student Notification Endpoints Missing

**Location:** `technical_prd.md § 5.4` vs `phase2_plan.md` Task 4

**Issue:**
`technical_prd.md § 5.4` shows student notification endpoints:
```
GET  /student/notifications         (paginated)
PATCH /student/notifications/{id}/read
```

Phase 2 Task 4 only defines:
```
GET  /notifications                 (no role prefix)
GET  /notifications/unread-count
PATCH /notifications/{id}/read
PATCH /notifications/read-all
```

There is a **prefix mismatch** — Phase 2 has `/notifications` while the PRD spec has `/student/notifications`. This means either:
- The notification router needs its own prefix-less mount AND student-specific mount, OR
- The student routes in `technical_prd.md` are wrong and notifications should be under a shared `/notifications` path accessible to any role

This inconsistency will cause confusion during implementation.

---

### 🔑 KEY GAP 4 — SEO Landing Page Has No Content Plan

**Location:** `phase2_plan.md` Task 5 §5.1

**Issue:**
Phase 2 says: *"Create `src/pages/Landing.jsx` with hero section, features grid, teacher/student sections, how-it-works, and footer."*

No specifics are provided for:
- What the domain/brand copy should say
- Which features to highlight
- What the hero image/illustration should be
- What the CTA buttons link to (teacher sign-up requires admin — so students only?)

This is a content/copy gap. Without this, the AI agent will generate placeholder text that will never reflect the actual product value proposition.

---

### 🔑 KEY GAP 5 — No Poll Interval Deduplication in `notificationStore.js`

**Location:** `phase2_plan.md` Task 4 `notificationStore.js`

**Issue:**
Phase 2's notification store uses **polling every 60 seconds** (`setInterval`). Phase 3 replaces this with SSE. However, in Phase 2, if the user navigates between pages that both mount the notification bell, multiple `setInterval` timers can accumulate, causing **N×60-second polls** where N is the number of component mounts without corresponding unmounts.

**Fix:** Ensure the `notificationStore.js` uses a global singleton interval with proper cleanup.

---

### ⚠️ WARNING 1 — `image_url` Validation in Schema Accepts Any String

**Location:** `phase2_plan.md` Task 1 `schemas/exam.py`

**Issue:**
The `QuestionRequest` schema has `image_url: Optional[str] = None`. After Phase 2 introduces Cloudinary, a teacher can technically submit any URL string (e.g., an external non-Cloudinary URL) directly to the question API. The upload endpoint generates a Cloudinary URL, but nothing prevents the teacher from bypassing it and submitting a raw `image_url` pointing to a hotlinked external image.

**Fix:** Add URL validation: either validate the URL starts with the Cloudinary base URL pattern, or set `image_url` as write-once (can only be set via the upload endpoint, not directly in question body).

---

### ⚠️ WARNING 2 — Recharts Bundle Size Impact Not Addressed

**Location:** `phase2_plan.md` Task 3 `npm install recharts`

**Issue:**
Recharts is ~500KB minified. No code-splitting or lazy loading is specified for analytics pages. Since these are teacher/admin-only pages, they should be lazy-loaded:

```jsx
const ExamAnalytics = lazy(() => import('./pages/teacher/ExamAnalytics'));
const RevenueAnalytics = lazy(() => import('./pages/admin/RevenueAnalytics'));
```

Without lazy loading, Recharts will be bundled into the main chunk, increasing initial load time for all users (including students who never see analytics).

---

### ⚠️ WARNING 3 — Bengali Noto Sans Font Loading May Fail

**Location:** `phase2_plan.md` Task 2 §2.2

**Issue:**
```css
@import url('https://fonts.googleapis.com/css2?family=Noto+Sans+Bengali...');
```

Google Fonts `@import` in CSS blocks rendering. For a bilingual exam attempt, late-loading Bengali font would cause a flash of unstyled Bengali text during the exam. Should use:
```html
<!-- in index.html, with preconnect -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="stylesheet" href="...">
```

---

### ⚠️ WARNING 4 — CSV Export Has No Auth Check for Exam Ownership

**Location:** `phase2_plan.md` Task 5 §5.2

**Issue:**
```python
@router.get("/exams/{exam_id}/export/csv")
async def export_exam_csv(
    exam_id: uuid.UUID,
    current_user=Depends(require_role("teacher")),
    ...
```

The endpoint only checks `require_role("teacher")`. It does **not verify** that the authenticated teacher owns the exam (`exam.teacher_id == current_user.id`). Any teacher can export results from any other teacher's exam by guessing the exam UUID.

**Fix:** Add ownership check: `exam = await db.get(Exam, exam_id); if exam.teacher_id != current_user.id: raise 403`.

---

## 🟡 PHASE 3 — Critical Errors & Key Gaps

### ❌ CRITICAL ERROR 1 — SSE In-Memory `active_connections` Breaks with Multiple Workers

**Location:** `phase3_plan.md` Task 4 §4.3

**Issue:**
The Phase 3 plan itself acknowledges this in a Note:
> *"For multi-worker deployments, replace the in-memory `active_connections` dict with Redis Pub/Sub."*

However, Phase 3 Task 5 §5.1 deploys with `--workers 4`. The SSE implementation uses an in-memory dict. With 4 workers, notifications pushed from Worker A won't reach connections held by Workers B, C, or D.

**Problem:** The plan provides the Redis Pub/Sub solution only as a note without actually implementing it. The deliverable (`✅ Real-time notifications via SSE`) claims completion but the implementation is broken for multi-worker.

**Fix:** The Phase 3 SSE task must include the Redis Pub/Sub implementation as part of the deliverable, not as an optional note.

---

### ❌ CRITICAL ERROR 2 — `pytest-cov` Included in `requirements.txt` (Not Dev Only)

**Location:** `phase3_plan.md` Task 2 §2.2

**Issue:**
```bash
pip install pytest pytest-asyncio httpx ruff pytest-cov
```

These are all dev/test dependencies but there's no separation from production `requirements.txt`. Phase 3's production Docker image (`Dockerfile.production`) runs:
```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

This installs `pytest`, `ruff`, `locust` etc. into the production Docker image — inflating image size by hundreds of MB and introducing security surface area.

**Fix:** Use separate `requirements-dev.txt` and `requirements-prod.txt`, or use `pyproject.toml` with optional dependency groups.

---

### 🔑 KEY GAP 1 — No Alembic Migration for Phase 3 Performance Indexes

**Location:** `phase3_plan.md` Task 1 §1.2

**Issue:**
Phase 3 states: *"The indexes already defined in the SQLAlchemy models (Phase 1) cover the critical paths. Additional composite indexes to add..."* and shows modifying `__table_args__` in models.

But if models are modified, a new Alembic migration must be generated and applied. Phase 3 mentions:
```bash
alembic revision --autogenerate -m "add_performance_indexes"
```

This is correct, but the migration will only work if `alembic --autogenerate` detects index changes. **Alembic does not always auto-detect index additions** (especially for named indexes or composite ones). Manual migration creation may be needed.

**Risk:** Developer runs `alembic revision --autogenerate` and sees an empty migration file (no changes detected). Assumes indexes were already applied. Performance gains never materialize.

---

### 🔑 KEY GAP 2 — SlowAPI Requires `request: Request` Parameter in Every Route

**Location:** `phase3_plan.md` Task 3 §3.1

**Issue:**
SlowAPI's `@limiter.limit()` decorator requires the route function to have `request: Request` as a parameter. Phase 3 shows updated auth routes with `request: Request` added — but the plan does NOT show updates for ALL routes across:
- All teacher routes in `app/api/teacher.py`
- All admin routes in `app/api/admin.py`
- All student routes in `app/api/student.py`

The default limiter applies `"200/15minutes"` to all routes, but the decorated routes (login, register, exam start, upload) need `request: Request` explicitly. If this parameter is not added to existing route functions when applying rate limit decorators, they will fail at startup.

**Fix:** Phase 3 Task 3 must include an explicit list of ALL route functions that need the `request: Request` parameter added (not just auth routes).

---

### 🔑 KEY GAP 3 — CI/CD Pipeline Does Not Include Frontend Deployment Step

**Location:** `phase3_plan.md` Task 2 §2.1 `frontend-ci.yml`

**Issue:**
The frontend CI workflow comment says:
```yaml
# Vercel auto-deploys from main — no manual step needed
```

But this assumes Vercel is connected to the GitHub repository. If the project uses a different Git workflow (e.g., a mono-repo with selective deployment), Vercel auto-deploy may not work. More critically, there is **no step to update environment variables** in Vercel when backend URL changes.

Additionally, the frontend workflow only runs `npm run build` for validation — it does not run `npm test` or any frontend-specific linting beyond ESLint. Frontend test coverage is zero across all three phases.

---

### 🔑 KEY GAP 4 — No Database Migration Strategy for Production Zero-Downtime

**Location:** `phase3_plan.md` Task 5 §5.6, Phase 1 deployment

**Issue:**
All deployment configs use:
```
alembic upgrade head && uvicorn app.main:app ...
```

This runs migrations **inside the same container startup command**. With multiple workers (`--workers 4`), all 4 workers attempt to start simultaneously. Each worker runs `alembic upgrade head` — creating a **race condition** where multiple workers try to apply the same migration simultaneously.

Alembic uses an `alembic_version` table with locking, but under fast concurrent starts, this can cause transient failures.

**Fix:** Run migrations as a **separate Docker init container** or Kubernetes init job before the main app starts. In `docker-compose.prod.yml`, add a migration service:
```yaml
migrate:
  build: ./backend
  command: alembic upgrade head
  depends_on: [postgres]
backend:
  depends_on:
    migrate:
      condition: service_completed_successfully
```

---

### ⚠️ WARNING 1 — InMemoryCacheService Has No TTL Enforcement

**Location:** `phase3_plan.md` Task 1 §1.1 `InMemoryCacheService`

**Issue:**
```python
class InMemoryCacheService(ICacheService):
    async def set(self, key: str, value: Any, ttl_seconds: int = 300) -> None:
        self._store[key] = value  # TTL ignored!
```

The `ttl_seconds` parameter is accepted but **completely ignored**. Cached values will live forever in memory. For a dev environment, this means:
- Old config values are never evicted
- Memory grows unbounded on long-running dev servers
- Test results may be stale if tests rely on TTL expiry behavior

**Fix:** Implement TTL using `asyncio.create_task` + `asyncio.sleep` or store values with expiry timestamps and check on `get()`.

---

### ⚠️ WARNING 2 — `structlog` Async Logger Usage

**Location:** `phase3_plan.md` Task 4 §4.1

**Issue:**
```python
await logger.ainfo("user_login", ...)
await logger.aerror("unhandled_error", ...)
```

`structlog`'s async logging (`ainfo`, `aerror`) requires `structlog.contextvars` to be configured properly for async contexts. In FastAPI (ASGI), each request runs in the same event loop thread but potentially different async contexts. Without `structlog.contextvars.clear_contextvars()` at the start of each request, context variables from a previous request may "leak" into the current one.

**Fix:** Add a middleware that calls `structlog.contextvars.clear_contextvars()` at the start of each request.

---

### ⚠️ WARNING 3 — Locust Load Test Uses Hardcoded Credentials

**Location:** `phase3_plan.md` Task 5 §5.3 `locustfile.py`

**Issue:**
```python
response = self.client.post("/api/v1/auth/student/login", json={
    "email": "loadtest-student@test.com",
    "password": "testpassword123",
})
```

Load test assumes pre-existing test users in the database. No setup/teardown is documented. Running load tests against a fresh production DB will result in 100% login failures (401 for all users), making load test results meaningless.

**Fix:** Add a `setup` phase in the locust file that creates test users via the API before running tasks, or document the required seed data.

---

## 🔁 Cross-Phase Inconsistencies

### Inconsistency 1 — `OTP` Purpose Values Mismatch

| Document | Values |
|---|---|
| `technical_prd.md` auth model comment | `registration \| forgot_password \| change_password` |
| `phase1_plan.md` `register_student()` | `purpose="registration"` |
| `technical_prd.md § 5.1` `VerifyOtpRequest` | `"registration" \| "forgot_password" \| "change_password"` |
| OTP model | `String(50)` (no enum validation) |

The `change_password` OTP purpose is documented in the model but the change-password flow in `technical_prd.md § 5.1` API table shows `change-password` as a single step (no OTP). This creates developer confusion about whether `change_password` OTP purpose is needed.

---

### Inconsistency 2 — Exam Attempt Result Data Discrepancy

**`product_feature.md`** Exam_Attempt fields:
| Field | Notes |
|---|---|
| `attempt_id` | No `is_submitted` column |

**`technical_prd.md`** ExamAttempt model has `is_submitted: bool`. The product feature doc's data model summary is missing:
- `is_submitted`
- `wallet_deducted_from`
- `cost_per_question_snapshot`

While the technical PRD is more authoritative, this inconsistency can cause confusion for business stakeholders reviewing the spec.

---

### Inconsistency 3 — Test Fixture Data Inconsistency

**`phase1_plan.md` Task 5 test:**
```python
"email": "admin@devophile.com",
"password": "testpassword"
```

**`phase3_plan.md` CI/CD workflow:**
```yaml
ADMIN_EMAIL: admin@test.com
ADMIN_PASSWORD: testpassword123
```

The admin credentials in Phase 1 tests (`admin@devophile.com` / `testpassword`) differ from Phase 3 CI env vars (`admin@test.com` / `testpassword123`). Tests from Phase 1 will fail in Phase 3's CI pipeline with the new env vars.

---

### Inconsistency 4 — `Section` Delete Behavior Undefined

**`product_feature.md § 4.5`** says sections can be created inline. There is no mention of what happens when a section is deleted that has questions in it.

**`technical_prd.md`** model: `section_id FK → sections.id, CASCADE delete` on Questions. This means deleting a section cascades to delete all its questions.

**But:** the question lock rule says: *"Once the first student submits an attempt, all questions in that exam are permanently locked."* If questions are locked (via `exam.is_locked = True`), but no check on `exam.is_locked` is made in the `DELETE /teacher/exams/{exam_id}/sections/{id}` endpoint, a teacher can delete a section (and cascade-delete all its questions) from a locked exam — bypassing the question lock.

**Fix:** `DELETE section` endpoint must check `exam.is_locked` and reject if true.

---

## 📋 Action Items Summary

### Must Fix Before Phase 1 Development

| # | Fix | Severity |
|---|---|---|
| 1 | Add `image_public_id` to Question model in Phase 1 | 🔴 Critical |
| 2 | Fix admin `created_by` UUID runtime error | 🔴 Critical |
| 3 | Fix Phase 1 conftest test isolation (add rollback) | 🔴 Critical |
| 4 | Add seeding to Docker start command | 🔴 Critical |
| 5 | Implement `GET /student/exams/my` with batch filter | 🔴 Critical |
| 6 | Define `is_submitted=False` cleanup policy | 🔴 Critical |
| 7 | Clarify change-password flow (OTP or current password) | 🔴 Critical |
| 8 | Add teacher soft-delete → exam hide join filter | 🟠 High |
| 9 | Add exam time-window enforcement in `start_exam` | 🟠 High |
| 10 | Add focused_zone/focused_exam enum validation | 🟠 High |
| 11 | Add admin password reset to Phase 1 task list | 🟠 High |
| 12 | Add all missing env vars to Phase 1 config | 🟠 High |
| 13 | Add basic rate limiting to Phase 1 auth endpoints | 🟠 High |

### Must Fix Before Phase 2 Development

| # | Fix | Severity |
|---|---|---|
| 14 | Fix Cloudinary sync call in async context | 🔴 Critical |
| 15 | Define `get_exam_attempts()` in analytics service | 🔴 Critical |
| 16 | Implement `single → bilingual` existing question validation | 🔴 Critical |
| 17 | Add exam ownership check to CSV export endpoint | 🟠 High |
| 18 | Register analytics router in `router.py` | 🟠 High |
| 19 | Implement "admin credits" and "teacher gifts" notification triggers | 🟠 High |
| 20 | Resolve `/notifications` vs `/student/notifications` prefix | 🟠 High |

### Must Fix Before Phase 3 Development

| # | Fix | Severity |
|---|---|---|
| 21 | Implement SSE with Redis Pub/Sub for multi-worker | 🔴 Critical |
| 22 | Separate prod/dev dependencies in requirements | 🔴 Critical |
| 23 | Add zero-downtime migration strategy | 🟠 High |
| 24 | Fix InMemoryCacheService TTL enforcement | 🟠 High |
| 25 | Add `request: Request` to all rate-limited routes | 🟠 High |

---

*Validation complete. All issues identified are based on direct cross-reference between the 5 source documents. Each finding cites specific line locations in the documents and explains the development/runtime impact.*
