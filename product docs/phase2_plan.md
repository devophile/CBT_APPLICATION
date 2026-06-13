# 📈 Phase 2 — Growth Features (Market Enhancement)

> **Goal:** Elevate the platform from functional MVP to a polished, data-rich product. Add the features that make teachers sticky and students engaged.  
> **Prerequisites:** Phase 1 fully deployed and stable  
> **Timeline:** 3–4 weeks  
> **Backend:** Python + FastAPI + SQLAlchemy (same as Phase 1)  
> **Outcome:** Enhanced UX, actionable analytics, visual content in exams, bilingual support, and a public-facing marketing page.

---

## What Ships in Phase 2

| ✅ Phase 2 Features | Depends On |
|---|---|
| Image uploads for questions (Cloudinary) | Storage service abstraction |
| Bilingual UI rendering (English + Bengali toggle) | Schema already has bn columns (Phase 1) |
| Analytics dashboards with charts (Recharts) | Exam attempt data (Phase 1) |
| Student performance growth graph (% marks over time) | Exam attempt history |
| Email notifications (low balance alerts to teachers) | Email service (Phase 1) |
| In-app notification system (bell icon + panel) | Notifications table (Phase 1) |
| Admin revenue dashboard (coin breakdown by type) | Transaction data (Phase 1) |
| SEO-optimized public landing page | New page |
| Export functionality (CSV export of exam results) | Exam analytics data |
| Teacher exam analytics page (per-exam stats) | Attempt data |

> [!NOTE]
> **Zero database migrations needed.** All tables and columns for Phase 2 features were created in Phase 1's Alembic migration. Phase 2 is purely application-layer code.

---

## Implementation Tasks (5 Tasks for AI Agent)

---

### Task 1: Image Upload — Cloudinary Integration

**Scope:** Implement the storage service abstraction and Cloudinary integration for question images.

#### 1.1 Backend — Storage Service

**Install dependency:**
```bash
pip install cloudinary python-multipart
pip freeze > requirements.txt
```

**Files to create/modify:**

| File | Action | Description |
|---|---|---|
| `app/services/storage_service.py` | CREATE | Abstract storage interface + Cloudinary implementation |
| `app/api/upload.py` | CREATE | `POST /api/v1/upload/image` |
| `app/schemas/exam.py` | MODIFY | Add `image_url` validation to question schemas |

**Storage service with abstraction:**

```python
# app/services/storage_service.py
from abc import ABC, abstractmethod
from typing import Optional, Tuple
import cloudinary
import cloudinary.uploader
from app.core.config import settings


class IStorageService(ABC):
    @abstractmethod
    async def upload_image(self, file_bytes: bytes, folder: str) -> Tuple[str, str]:
        """Returns (url, public_id)"""
        pass

    @abstractmethod
    async def delete_image(self, public_id: str) -> bool:
        pass


class CloudinaryStorageService(IStorageService):
    def __init__(self):
        cloudinary.config(
            cloud_name=settings.CLOUDINARY_CLOUD_NAME,
            api_key=settings.CLOUDINARY_API_KEY,
            api_secret=settings.CLOUDINARY_API_SECRET,
        )

    async def upload_image(self, file_bytes: bytes, folder: str = "questions") -> Tuple[str, str]:
        result = cloudinary.uploader.upload(
            file_bytes,
            folder=f"cbt_platform/{folder}",
            resource_type="image",
            transformation=[
                {"width": 800, "height": 600, "crop": "limit"},
                {"quality": "auto", "fetch_format": "auto"},
            ],
        )
        return result["secure_url"], result["public_id"]

    async def delete_image(self, public_id: str) -> bool:
        result = cloudinary.uploader.destroy(public_id)
        return result.get("result") == "ok"


# Factory — swap to S3StorageService or OracleStorageService later
def get_storage_service() -> IStorageService:
    provider = getattr(settings, "STORAGE_PROVIDER", "cloudinary")
    if provider == "cloudinary":
        return CloudinaryStorageService()
    # Future: elif provider == "s3": return S3StorageService()
    return CloudinaryStorageService()
```

**Upload endpoint:**
```python
# app/api/upload.py
from fastapi import APIRouter, Depends, UploadFile, File, HTTPException
from app.core.dependencies import require_role
from app.services.storage_service import get_storage_service

router = APIRouter(prefix="/upload", tags=["Upload"])

ALLOWED_TYPES = {"image/jpeg", "image/png", "image/webp", "image/gif"}
MAX_SIZE = 5 * 1024 * 1024  # 5MB


@router.post("/image")
async def upload_image(
    file: UploadFile = File(...),
    current_user=Depends(require_role("teacher")),
):
    if file.content_type not in ALLOWED_TYPES:
        raise HTTPException(400, "Only JPEG, PNG, WebP, and GIF allowed")

    contents = await file.read()
    if len(contents) > MAX_SIZE:
        raise HTTPException(400, "File too large. Max 5MB.")

    storage = get_storage_service()
    url, public_id = await storage.upload_image(contents)

    return {"success": True, "data": {"url": url, "public_id": public_id}}
```

#### 1.2 Frontend — Question Image Upload

**Modify `QuestionForm.jsx`:** Add image upload field with preview + remove.  
**Modify `ExamAttempt.jsx`:** Render question images above question text.  
**Modify `SolutionView.jsx`:** Render question images in review.

> [!IMPORTANT]
> **Image Cleanup on Question Deletion/Update:** When a question with an `image_url` is deleted (`DELETE /teacher/exams/{exam_id}/questions/{id}`) or its image is replaced via update (`PATCH`), the service layer must call `storage_service.delete_image(old_public_id)` to remove the orphaned image from Cloudinary. Store the Cloudinary `public_id` alongside `image_url` in the `Question` model (add `image_public_id: Mapped[Optional[str]]` column) to enable cleanup.

**Deliverables after Task 1:**
- ✅ Teachers can upload images to questions (max 5MB)
- ✅ Images display during exam and in solution view
- ✅ Images optimized and served via Cloudinary CDN
- ✅ Storage service is swappable (abstract interface)
- ✅ Orphaned images cleaned up on question delete/update

---

### Task 2: Bilingual Support (English + Bengali)

**Scope:** Enable the language toggle during exam attempts for bilingual exams.

#### 2.1 Backend Changes (Minimal)

Bengali columns already exist in the schema. Only validation changes:

```python
# app/schemas/exam.py — update QuestionRequest
class QuestionRequest(BaseModel):
    qn_en: str = Field(min_length=1)
    op_en_1: str = Field(min_length=1)
    op_en_2: str = Field(min_length=1)
    op_en_3: str = Field(min_length=1)
    op_en_4: str = Field(min_length=1)
    correct_option: Literal["1", "2", "3", "4"]
    solution: Optional[str] = None
    image_url: Optional[str] = None
    section_id: str

    # Bengali (optional — enforced at service level for bilingual exams)
    qn_bn: Optional[str] = None
    op_bn_1: Optional[str] = None
    op_bn_2: Optional[str] = None
    op_bn_3: Optional[str] = None
    op_bn_4: Optional[str] = None
```

```python
# In exam_service.py — enforce Bengali fields for bilingual exams
async def add_question(exam_id, data, db):
    exam = await db.get(Exam, exam_id)
    if exam.language == ExamLanguage.bilingual:
        if not all([data.qn_bn, data.op_bn_1, data.op_bn_2, data.op_bn_3, data.op_bn_4]):
            raise ApiError(400, "Bengali fields required for bilingual exams")
    ...
```

#### 2.2 Frontend Changes

**Modify `QuestionForm.jsx`:** Show Bengali input fields when exam language is `bilingual`.

**Modify `ExamAttempt.jsx` + `SolutionView.jsx`:** Add language toggle:
```jsx
{examMeta.language === 'bilingual' && (
  <div className="flex rounded-lg overflow-hidden border">
    <button className={lang==='en' ? 'bg-indigo-600 text-white' : 'bg-white'} onClick={()=>setLang('en')}>
      English
    </button>
    <button className={lang==='bn' ? 'bg-indigo-600 text-white' : 'bg-white'} onClick={()=>setLang('bn')}>
      বাংলা
    </button>
  </div>
)}
```

**Add Bengali font support in `index.css`:**
```css
@import url('https://fonts.googleapis.com/css2?family=Noto+Sans+Bengali:wght@400;500;600;700&display=swap');
```

**Deliverables after Task 2:**
- ✅ Teachers can enter Bengali content for bilingual exams
- ✅ Students see EN / বাংলা toggle during exam
- ✅ Toggle works in both attempt and solution views
- ✅ Correct Bengali font renders

---

### Task 3: Analytics Dashboards

**Scope:** Data visualization for teachers (per-exam, student performance) and admin (revenue).

#### 3.1 Install Frontend Dependencies

```bash
cd frontend
npm install recharts date-fns
```

#### 3.2 Backend — Analytics Endpoints

**Files to create:**

| File | Description |
|---|---|
| `app/services/analytics_service.py` | Aggregation queries |
| `app/api/analytics.py` | Analytics routes |

**Analytics endpoints:**

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `GET` | `/teacher/exams/{id}/analytics` | Teacher | Per-exam stats |
| `GET` | `/teacher/students/{id}/performance` | Teacher | Student performance trend |
| `GET` | `/admin/analytics/revenue` | Admin | Revenue breakdown |
| `GET` | `/admin/analytics/overview` | Admin | Platform-wide stats |

**Analytics service:**
```python
# app/services/analytics_service.py

async def get_exam_analytics(exam_id: uuid.UUID, teacher_id: uuid.UUID, db: AsyncSession):
    """Per-exam: total attempts, avg score, distribution, top performers."""
    # Verify exam belongs to teacher
    exam = await db.execute(
        select(Exam).where(Exam.id == exam_id, Exam.teacher_id == teacher_id)
    )
    exam = exam.scalar_one_or_none()
    if not exam:
        raise NotFoundError("Exam")

    # Get all submitted attempts with student names
    attempts = await db.execute(
        select(ExamAttempt, User.name, User.email)
        .join(User, ExamAttempt.student_id == User.id)
        .where(ExamAttempt.exam_id == exam_id, ExamAttempt.is_submitted == True)
        .order_by(ExamAttempt.score.desc())
    )
    attempts = attempts.all()

    if not attempts:
        return {"exam_name": exam.name, "total_attempts": 0}

    scores = [float(a[0].score) for a in attempts]
    total_marks = float(attempts[0][0].total_marks) if attempts else 0

    # Score distribution (buckets)
    distribution = build_score_distribution(scores, total_marks)

    # Violation count
    violation_count = await db.execute(
        select(func.count(Violation.id))
        .join(ExamAttempt, Violation.attempt_id == ExamAttempt.id)
        .where(ExamAttempt.exam_id == exam_id)
    )

    return {
        "exam_name": exam.name,
        "total_attempts": len(attempts),
        "avg_score": round(sum(scores) / len(scores), 2),
        "max_score": max(scores, default=0),
        "min_score": min(scores, default=0),
        "total_possible_marks": total_marks,
        "score_distribution": distribution,
        "avg_time_taken": round(sum(a[0].time_taken_seconds or 0 for a in attempts) / len(attempts) / 60, 1),
        "violation_count": violation_count.scalar(),
        "top_performers": [
            {"name": a[1], "score": float(a[0].score), "time_taken": a[0].time_taken_seconds}
            for a in attempts[:5]
        ],
        "attempts": [
            {"student_name": a[1], "email": a[2], "score": float(a[0].score),
             "time_taken": a[0].time_taken_seconds, "submitted_at": a[0].submitted_at.isoformat()}
            for a in attempts
        ],
    }


async def get_student_performance(student_id: uuid.UUID, teacher_id: uuid.UUID, db: AsyncSession):
    """Student's score percentage trend over time."""
    attempts = await db.execute(
        select(ExamAttempt, Exam.name)
        .join(Exam, ExamAttempt.exam_id == Exam.id)
        .where(
            ExamAttempt.student_id == student_id,
            ExamAttempt.is_submitted == True,
            Exam.teacher_id == teacher_id,
        )
        .order_by(ExamAttempt.submitted_at.asc())
    )

    return {
        "performance_trend": [
            {
                "exam_name": name,
                "date": a.submitted_at.isoformat(),
                "percentage": round(float(a.score) / float(a.total_marks) * 100, 1) if a.total_marks else 0,
            }
            for a, name in attempts.all()
        ]
    }


async def get_revenue_analytics(date_from, date_to, db: AsyncSession):
    """Admin revenue: total, by type, by month."""
    query = select(Transaction).where(Transaction.transaction_type == TransactionType.debit)
    if date_from:
        query = query.where(Transaction.created_at >= date_from)
    if date_to:
        query = query.where(Transaction.created_at <= date_to)

    transactions = await db.execute(query)
    txns = transactions.scalars().all()

    total = sum(float(t.amount) for t in txns)
    by_reason = {}
    for t in txns:
        by_reason.setdefault(t.reason.value, Decimal("0"))
        by_reason[t.reason.value] += t.amount

    return {
        "total_revenue": total,
        "revenue_by_type": {k: float(v) for k, v in by_reason.items()},
    }
```

#### 3.3 Frontend — Chart Components

**Files to create:**

| File | Description |
|---|---|
| `src/pages/teacher/ExamAnalytics.jsx` | Per-exam analytics with charts |
| `src/pages/admin/RevenueAnalytics.jsx` | Revenue dashboard |
| `src/components/charts/ScoreDistributionChart.jsx` | Bar chart |
| `src/components/charts/PerformanceTrendChart.jsx` | Line/area chart |
| `src/components/charts/RevenueChart.jsx` | Bar chart |

**Deliverables after Task 3:**
- ✅ Per-exam analytics (score distribution, top performers, avg time)
- ✅ Student performance growth graph (% over time)
- ✅ Admin revenue dashboard (total, by type)
- ✅ All charts render with Recharts

---

### Task 4: Notification System + Email Alerts

**Scope:** In-app notifications and email alerts for critical events.

#### 4.1 Backend — Notification Service

**Files to create/modify:**

| File | Action |
|---|---|
| `app/services/notification_service.py` | CREATE — create, fetch, mark-read |
| `app/api/notification.py` | CREATE — notification routes |
| `app/services/student_service.py` | MODIFY — trigger notification on low balance |
| `app/services/email_service.py` | MODIFY — add low balance email template |

**Notification triggers:**

| Event | Notify | Email? |
|---|---|---|
| Private exam — teacher wallet insufficient | Teacher | ✅ |
| Admin credits coins | Recipient | ❌ |
| Teacher gifts coins to student | Student | ❌ |

**Notification endpoints:**

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/notifications` | Get notifications (paginated) |
| `GET` | `/notifications/unread-count` | Unread count |
| `PATCH` | `/notifications/{id}/read` | Mark read |
| `PATCH` | `/notifications/read-all` | Mark all read |

**Email service update:**
```python
# app/services/email_service.py

async def send_low_balance_alert(
    teacher_email: str, teacher_name: str,
    exam_name: str, student_name: str,
    required: Decimal, current_balance: Decimal,
):
    message = MessageSchema(
        subject="⚠️ Student Blocked — Insufficient Wallet Balance",
        recipients=[teacher_email],
        body=f"""
        Hi {teacher_name},

        A student was unable to start your exam due to insufficient balance.

        Exam: {exam_name}
        Student: {student_name}
        Required: {required} coins
        Your Balance: {current_balance} coins

        Please top up your wallet.

        — Devophile CBT Platform
        """,
        subtype="plain",
    )
    await fast_mail.send_message(message)
```

#### 4.2 Frontend — Notification UI

**Files to create:**

| File | Description |
|---|---|
| `src/components/layout/NotificationBell.jsx` | Bell icon with badge |
| `src/components/layout/NotificationPanel.jsx` | Dropdown with list |
| `src/store/notificationStore.js` | Zustand store + polling (60s) |

**Deliverables after Task 4:**
- ✅ Teachers receive notifications when students blocked
- ✅ Teachers receive email alerts for low balance
- ✅ Bell icon with unread count in topbar
- ✅ Notification panel with mark-as-read

---

### Task 5: SEO Landing Page + CSV Export + Polish

**Scope:** Marketing landing page, data export, final polish.

#### 5.1 SEO Landing Page

Create `src/pages/Landing.jsx` with hero section, features grid, teacher/student sections, how-it-works, and footer. Full `react-helmet-async` meta tags + structured data (JSON-LD).

**`public/robots.txt`:**
```
User-agent: *
Allow: /
Disallow: /admin
Disallow: /teacher
Disallow: /student
Sitemap: https://cbt.devophile.com/sitemap.xml
```

#### 5.2 CSV Export

**Backend endpoint:**
```python
# app/api/teacher.py

@router.get("/exams/{exam_id}/export/csv")
async def export_exam_csv(
    exam_id: uuid.UUID,
    current_user=Depends(require_role("teacher")),
    db: AsyncSession = Depends(get_db),
):
    import csv
    from io import StringIO
    from fastapi.responses import StreamingResponse

    attempts = await analytics_service.get_exam_attempts(exam_id, current_user.id, db)

    output = StringIO()
    writer = csv.writer(output)
    writer.writerow(["Student Name", "Email", "Score", "Total Marks", "Percentage", "Time (min)", "Submitted"])

    for a in attempts:
        pct = round(float(a["score"]) / float(a["total_marks"]) * 100, 2) if a["total_marks"] else 0
        writer.writerow([
            a["student_name"], a["email"], a["score"], a["total_marks"],
            f"{pct}%", round(a["time_taken"] / 60, 1) if a["time_taken"] else "N/A",
            a["submitted_at"],
        ])

    output.seek(0)
    return StreamingResponse(
        iter([output.getvalue()]),
        media_type="text/csv",
        headers={"Content-Disposition": f"attachment; filename=exam_results_{exam_id}.csv"},
    )
```

#### 5.3 Final Polish

- [ ] Loading skeletons for all lists/tables
- [ ] Toast notifications for all success/error
- [ ] Empty states with illustrations
- [ ] Responsive testing (375px, 768px, 1440px)
- [ ] Confirmation dialogs for destructive actions
- [ ] Timer warning (red flash at < 5 min)
- [ ] Error boundary components
- [ ] 404 page styled
- [ ] Favicon + app branding

**Deliverables after Task 5:**
- ✅ Public landing page with full SEO
- ✅ CSV export for exam results
- ✅ Polished UI with skeletons and animations
- ✅ Mobile-responsive experience

---

## Phase 2 — New Dependencies

| Package | Purpose | Location |
|---|---|---|
| `cloudinary` | Image upload/storage | Backend (pip) |
| `recharts` | Data visualization charts | Frontend (npm) |
| `date-fns` | Date formatting | Frontend (npm) |

---

*For Phase 3 (Redis, CI/CD, monitoring, cloud migration), see [phase3_plan.md](./phase3_plan.md)*
