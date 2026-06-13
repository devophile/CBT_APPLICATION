# 🚀 Phase 1 — Core MVP (Market-Ready Launch)

> **Goal:** Ship a fully functional, production-stable exam platform that teachers can use to create and conduct exams, students can attempt and view results, and the admin can manage everything.  
> **Timeline:** 4–5 weeks  
> **Backend:** Python 3.12 + FastAPI + SQLAlchemy + Alembic + PostgreSQL  
> **Frontend:** React 18 + Vite + Tailwind CSS  
> **Outcome:** Live product. Real users. Revenue generating.

---

## What Ships in Phase 1

| ✅ Included | ❌ NOT Included (Phase 2/3) |
|---|---|
| All 3 roles (Admin, Teacher, Student) | Image uploads for questions |
| Full auth with JWT + OTP email | Bilingual (Bengali) UI rendering |
| Super Admin panel (teacher CRUD, student mgmt, coin mgmt, config) | Analytics charts & graphs |
| Teacher: student linking, batch CRUD, exam CRUD, sections, questions | Email notifications / Notification panel UI |
| Student: registration, home page, exam browsing, full exam attempt, results | CSV/PDF export |
| Wallet system (credit, debit, balance checks, transaction history) | SEO landing page (marketing) |
| Exam cost calculation & deduction (atomic transactions) | Cloudinary/S3 integration |
| Fullscreen violation detection & logging | Redis caching |
| Single-device session management | CI/CD pipelines |
| Docker Compose for local dev | Advanced rate limiting |
| Responsive UI (mobile + desktop) | Monitoring/APM |
| **Auto-generated Swagger docs at /docs** | — |

> [!IMPORTANT]
> **Phase 1 is bilingual-schema-ready.** The database stores Bengali fields (`qn_bn`, `op_bn_*`), but the UI does not render the language toggle in Phase 1. Teachers can enter bilingual content — it will render properly when Phase 2 ships the toggle. This avoids a database migration between phases.

> [!NOTE]
> **Notification records ARE created in Phase 1.** When a student is blocked from a private exam due to insufficient teacher wallet balance, the `start_exam` flow creates a `Notification` record in the database (see `technical_prd.md` § 5.5). The notification panel UI (bell icon, dropdown, mark-as-read) is deferred to Phase 2. Teachers can check their wallet transaction history in Phase 1 to understand why deductions stopped.

---

## Tech Stack (Phase 1)

| Layer | Technology |
|---|---|
| Backend Framework | **FastAPI** (with Uvicorn ASGI server) |
| ORM | **SQLAlchemy 2.0** (async mode with `asyncpg`) |
| Migrations | **Alembic** (auto-generated, reviewable, rollback-capable) |
| Validation | **Pydantic v2** (built-in to FastAPI — auto Swagger docs) |
| Auth | **python-jose** (JWT) + **passlib[bcrypt]** |
| Email | **fastapi-mail** (SMTP/Gmail) |
| Database | PostgreSQL 16 (Neon DB) |
| Frontend | React 18 + Vite + Tailwind CSS + React Router v6 |
| State | Zustand |
| HTTP Client | Axios with interceptors |
| Testing | **pytest + httpx** (backend), manual (frontend) |
| Deploy | Vercel (frontend) + Render (backend) + Neon DB |

---

## Database: Phase 1 Schema & Migrations

### Migration Workflow

All tables from [technical_prd.md](./technical_prd.md) § 4.1 are created in Phase 1. Alembic auto-generates migrations from SQLAlchemy models.

```bash
cd backend

# 1. Initialize Alembic (one-time)
alembic init alembic

# 2. Configure alembic/env.py to import all models and use DATABASE_URL_SYNC

# 3. Auto-generate migrations from models
alembic revision --autogenerate -m "init_all_tables"

# 4. Apply migrations (checks alembic_version stamp — only runs NEW revisions)
alembic upgrade head

# Seed data is embedded in migration 005_init_attempt_tables_and_seed.
# It runs exactly once when that migration is applied. No separate seed command needed.
```

**Rolling back if needed:**
```bash
# Downgrade one version
alembic downgrade -1

# Downgrade to a specific revision
alembic downgrade abc123

# View current state
alembic current

# View full history
alembic history --verbose
```

> [!NOTE]
> We create ALL tables in Phase 1 — including `notifications` and bilingual question columns. This avoids destructive migrations later. Phase 2 simply starts using the columns/tables that already exist.

---

## Implementation Tasks (5 Tasks for AI Agent)

Each task is self-contained and builds on the previous one. Feed them to an AI coding agent sequentially.

---

### Task 1: Project Scaffolding + Backend Foundation + Auth System

**Scope:** Initialize the FastAPI project, set up database connection, middleware, and implement the complete auth system.

#### 1.1 Backend Project Setup

```bash
mkdir backend && cd backend

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Linux/Mac
# or: venv\Scripts\activate  # Windows

# Install dependencies
pip install fastapi uvicorn[standard] sqlalchemy[asyncio] asyncpg alembic \
    psycopg2-binary pydantic-settings python-jose[cryptography] passlib[bcrypt] \
    fastapi-mail python-multipart email-validator python-dotenv httpx

# Save requirements
pip freeze > requirements.txt
```

#### 1.2 Core Files to Create

**`app/main.py`** — FastAPI entry point:
```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.core.config import settings
from app.core.database import engine
from app.api.router import api_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    print("🚀 Starting CBT Platform API")
    yield
    # Shutdown
    await engine.dispose()
    print("👋 Shutting down")


app = FastAPI(
    title="Devophile CBT Platform",
    description="Computer Based Testing Platform API",
    version="1.0.0",
    lifespan=lifespan,
    docs_url="/docs",      # Swagger UI
    redoc_url="/redoc",    # ReDoc
)

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=[settings.FRONTEND_URL],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Health check
@app.get("/health")
async def health():
    return {"status": "ok"}

# API routes
app.include_router(api_router, prefix="/api/v1")
```

**`app/core/database.py`** — SQLAlchemy async engine:
```python
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    async_sessionmaker,
    AsyncSession,
)
from app.core.config import settings

engine = create_async_engine(
    settings.DATABASE_URL,
    echo=settings.ENV == "development",
    pool_size=10,
    max_overflow=20,
)

async_session_factory = async_sessionmaker(
    engine, class_=AsyncSession, expire_on_commit=False
)


async def get_db():
    """FastAPI dependency — yields an async DB session."""
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()
```

**`app/core/config.py`** — Pydantic Settings:
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    ENV: str = "development"
    PORT: int = 8000
    DATABASE_URL: str
    DATABASE_URL_SYNC: str = ""  # For Alembic sync operations
    JWT_ACCESS_SECRET: str
    JWT_REFRESH_SECRET: str
    ADMIN_EMAIL: str
    ADMIN_PASSWORD: str
    # ─── Admin Sentinel UUID (no DB record for admin) ───
    # Used as created_by in Transaction records for admin-issued credits
    ADMIN_SENTINEL_UUID: str = "00000000-0000-0000-0000-000000000000"
    SMTP_HOST: str = "smtp.gmail.com"
    SMTP_PORT: int = 587
    SMTP_USER: str = ""
    SMTP_PASS: str = ""
    SMTP_FROM: str = ""
    SMTP_FROM_NAME: str = "Devophile CBT"
    FRONTEND_URL: str = "http://localhost:5173"
    # ─── Cloudinary (Phase 2) — defined here to avoid AttributeError on import ───
    CLOUDINARY_CLOUD_NAME: str = ""
    CLOUDINARY_API_KEY: str = ""
    CLOUDINARY_API_SECRET: str = ""
    STORAGE_PROVIDER: str = "cloudinary"  # swap to "s3" or "oracle" in Phase 3
    # ─── Redis (Phase 3) ───
    REDIS_URL: str = ""
    # ─── Sentry (Phase 3) ───
    SENTRY_DSN: str = ""

    class Config:
        env_file = ".env"

settings = Settings()
```

**`app/utils/exceptions.py`** — Custom exceptions:
```python
from fastapi import HTTPException


class ApiError(HTTPException):
    def __init__(self, status_code: int, message: str, errors: list = None):
        super().__init__(status_code=status_code, detail=message)
        self.errors = errors or []


class NotFoundError(ApiError):
    def __init__(self, resource: str = "Resource"):
        super().__init__(404, f"{resource} not found")


class ConflictError(ApiError):
    def __init__(self, message: str = "Resource already exists"):
        super().__init__(409, message)


class InsufficientBalanceError(ApiError):
    def __init__(self, message: str = "Insufficient balance"):
        super().__init__(402, message)
```

**`app/utils/responses.py`** — Response helpers:
```python
from typing import Any, Optional


def success_response(
    data: Any = None,
    message: str = "Success",
    meta: Optional[dict] = None
) -> dict:
    response = {"success": True, "message": message, "data": data}
    if meta:
        response["meta"] = meta
    return response


def paginated_response(
    data: list,
    total: int,
    page: int,
    limit: int,
    message: str = "Success",
) -> dict:
    return {
        "success": True,
        "message": message,
        "data": data,
        "meta": {
            "page": page,
            "limit": limit,
            "total": total,
            "total_pages": (total + limit - 1) // limit,
        },
    }
```

#### 1.3 Auth System Implementation

**Files to create:**

| File | Responsibility |
|---|---|
| `app/core/security.py` | `hash_password()`, `verify_password()`, `create_access_token()`, `create_refresh_token()`, `decode_token()` |
| `app/core/dependencies.py` | `get_db()`, `get_current_user()`, `require_role()` |
| `app/models/user.py` | User, TeacherProfile, StudentProfile SQLAlchemy models |
| `app/models/auth.py` | Otp, RefreshToken models |
| `app/models/wallet.py` | Wallet, Transaction models |
| `app/schemas/auth.py` | Pydantic schemas: LoginRequest, RegisterRequest, TokenResponse, etc. |
| `app/services/auth_service.py` | `admin_login()`, `teacher_login()`, `student_login()`, `register_student()`, `verify_otp()`, `forgot_password()`, `reset_password()`, `change_password()`, `refresh_token()`, `logout()` |
| `app/services/otp_service.py` | `create_otp()`, `verify_otp()` — stores in DB with 10-min expiry |
| `app/services/email_service.py` | `send_otp()`, `send_password_reset()` — fastapi-mail wrapper |
| `app/api/auth.py` | Auth router with all `/auth/*` endpoints |

**Key `auth_service.py` behaviors:**

```python
async def admin_login(email: str, password: str, db: AsyncSession) -> dict:
    """DB lookup — admin is a real User record seeded from .env on every startup.

    The startup script (scripts/startup.py) upserts the admin user from
    ADMIN_EMAIL / ADMIN_PASSWORD env vars on every boot. If .env changes,
    the DB record is updated on next restart.
    """
    user = await db.execute(
        select(User).where(User.email == email, User.role == Role.super_admin, User.is_deleted == False)
    )
    user = user.scalar_one_or_none()
    if not user:
        raise ApiError(401, "Invalid credentials")
    if not verify_password(password, user.password):
        raise ApiError(401, "Invalid credentials")

    # Single device — overwrite session (same as teacher/student)
    session_id = str(uuid.uuid4())
    user.active_session_id = session_id
    await db.flush()

    access_token = create_access_token(str(user.id), user.email, "super_admin", session_id)
    refresh_token = create_refresh_token(str(user.id))
    return {"user": {"id": str(user.id), "email": user.email, "name": user.name, "role": "super_admin"},
            "tokens": {"access_token": access_token, "refresh_token": refresh_token}}


async def student_login(email: str, password: str, db: AsyncSession) -> dict:
    """Find user, verify password, enforce single-device session."""
    user = await db.execute(
        select(User).where(User.email == email, User.role == Role.student, User.is_deleted == False)
    )
    user = user.scalar_one_or_none()
    if not user:
        raise ApiError(401, "Invalid credentials")
    if not user.is_active:
        raise ApiError(403, "Account deactivated")
    if not verify_password(password, user.password):
        raise ApiError(401, "Invalid credentials")

    # Single device — overwrite session
    session_id = str(uuid.uuid4())
    user.active_session_id = session_id
    await db.flush()

    access_token = create_access_token(str(user.id), user.email, "student", session_id)
    refresh_token = create_refresh_token(str(user.id))

    # Store refresh token in DB
    db.add(RefreshToken(user_id=user.id, token=refresh_token, expires_at=...))
    return {"user": UserResponse.from_orm(user), "tokens": {...}}


async def register_student(data: RegisterStudentRequest, db: AsyncSession) -> dict:
    """Step 1: Validate email uniqueness → generate OTP → send email."""
    existing = await db.execute(select(User).where(User.email == data.email))
    if existing.scalar_one_or_none():
        raise ConflictError("Email already registered")

    otp_code = generate_otp()  # 6-digit random
    db.add(Otp(email=data.email, code=otp_code, purpose="registration",
               expires_at=datetime.utcnow() + timedelta(minutes=10)))
    await db.flush()
    await send_otp_email(data.email, otp_code)
    return {"message": "OTP sent to email"}


async def verify_registration_otp(
    email: str, code: str, registration_data: dict, db: AsyncSession
) -> dict:
    """Step 2: Verify OTP → create user + wallet + profile → link pending invites."""
    # Verify OTP
    otp = await db.execute(
        select(Otp).where(
            Otp.email == email, Otp.purpose == "registration",
            Otp.is_used == False, Otp.expires_at > datetime.utcnow()
        ).order_by(Otp.created_at.desc())
    )
    otp = otp.scalar_one_or_none()
    if not otp or otp.code != code:
        raise ApiError(400, "Invalid or expired OTP")

    otp.is_used = True

    # Create user
    user = User(
        role=Role.student, email=email,
        password=hash_password(registration_data["password"]),
        name=registration_data["name"], phone=registration_data["phone"],
    )
    db.add(user)
    await db.flush()

    # Create student profile
    db.add(StudentProfile(
        user_id=user.id,
        institute_name=registration_data["institute_name"],
        focused_exam=registration_data["focused_exam"],
    ))

    # Create wallet
    db.add(Wallet(user_id=user.id, balance=Decimal("0")))

    # Check for pending teacher invites
    pending_links = await db.execute(
        select(TeacherStudent).where(
            TeacherStudent.invited_email == email,
            TeacherStudent.status == TeacherStudentStatus.pending,
        )
    )
    for link in pending_links.scalars():
        link.student_id = user.id
        link.status = TeacherStudentStatus.active
        link.linked_at = datetime.utcnow()

    # Issue tokens
    session_id = str(uuid.uuid4())
    user.active_session_id = session_id
    access_token = create_access_token(str(user.id), email, "student", session_id)
    refresh_token = create_refresh_token(str(user.id))
    return {"user": ..., "tokens": ...}
```

**Router example (`app/api/auth.py`):**
```python
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.database import get_db
from app.schemas.auth import LoginRequest, RegisterStudentRequest, VerifyOtpRequest
from app.services import auth_service

router = APIRouter(prefix="/auth", tags=["Authentication"])


@router.post("/admin/login")
async def admin_login(body: LoginRequest, db: AsyncSession = Depends(get_db)):
    return await auth_service.admin_login(body.email, body.password, db)


@router.post("/teacher/login")
async def teacher_login(body: LoginRequest, db: AsyncSession = Depends(get_db)):
    return await auth_service.teacher_login(body.email, body.password, db)


@router.post("/student/login")
async def student_login(body: LoginRequest, db: AsyncSession = Depends(get_db)):
    return await auth_service.student_login(body.email, body.password, db)


@router.post("/student/register")
async def register_student(body: RegisterStudentRequest, db: AsyncSession = Depends(get_db)):
    return await auth_service.register_student(body, db)


@router.post("/verify-otp")
async def verify_otp(body: VerifyOtpRequest, db: AsyncSession = Depends(get_db)):
    return await auth_service.verify_otp(body.email, body.code, body.purpose, db)
```

#### 1.4 Frontend Initialization

```bash
cd frontend
npm install react-router-dom axios zustand react-hook-form @hookform/resolvers zod \
    react-helmet-async react-hot-toast lucide-react
```

Create the same frontend structure as described in [technical_prd.md](./technical_prd.md) § 3 (frontend tree). The frontend is framework-agnostic — it communicates via REST API only. All Axios, Zustand, hooks, and component code is identical to the previous version.

**Key point:** The only difference is the API base URL port changes from `5000` to `8000`:
```bash
VITE_API_BASE_URL=http://localhost:8000/api/v1
```

**Deliverables after Task 1:**
- ✅ FastAPI server starts with `uvicorn app.main:app --reload`
- ✅ Swagger docs accessible at `http://localhost:8000/docs`
- ✅ Database connected (Neon DB or local PG)
- ✅ Alembic migrations applied
- ✅ All auth endpoints functional (login, register, OTP, forgot password)
- ✅ JWT tokens issued and verified
- ✅ Single-device session enforcement works
- ✅ Frontend login/register pages render and connect to API

---

### Task 2: Super Admin Panel

**Scope:** Complete Admin dashboard, teacher management, student management, wallet crediting, and exam cost configuration.

#### 2.1 Backend — Admin Service & Routes

**Files to create:**

| File | Responsibility |
|---|---|
| `app/services/admin_service.py` | Teacher CRUD, student management, wallet credits, config |
| `app/services/wallet_service.py` | `credit_wallet()`, `debit_wallet()`, `get_balance()`, `get_transactions()` |
| `app/schemas/admin.py` | Pydantic schemas for admin operations |
| `app/schemas/wallet.py` | Pydantic schemas for wallet operations |
| `app/api/admin.py` | All `/admin/*` routes |
| `app/api/wallet.py` | `/wallet/*` routes |

**Key service functions:**

```python
# app/services/admin_service.py

async def create_teacher(data: CreateTeacherRequest, db: AsyncSession) -> dict:
    """Create teacher with profile and wallet in one transaction."""
    async with db.begin_nested():  # Savepoint
        # Hash password
        hashed = hash_password(data.password)

        # Create user
        user = User(role=Role.teacher, email=data.email, password=hashed,
                     name=data.name, phone=data.phone)
        db.add(user)
        await db.flush()

        # Create teacher profile
        profile = TeacherProfile(
            user_id=user.id, subject=data.subject,
            focused_zones=data.focused_zones, teaching_since=data.teaching_since,
            degrees=data.degrees, locality=data.locality,
            city=data.city, state=data.state, image_url=data.image_url,
        )
        db.add(profile)

        # Create wallet
        db.add(Wallet(user_id=user.id, balance=Decimal("0")))

    return success_response(UserResponse.from_orm(user), "Teacher created")


async def credit_wallet(
    user_id: uuid.UUID, amount: Decimal, credit_type: CreditType,
    description: str, admin_id: str, db: AsyncSession
) -> dict:
    """Credit coins to any user's wallet (atomic).

    admin_id is the JWT `sub` claim. For the Super Admin it is the sentinel
    UUID "00000000-0000-0000-0000-000000000000" (no real DB record).
    uuid.UUID() handles this safely since it is a valid UUID string.
    """
    wallet = await db.execute(
        select(Wallet).where(Wallet.user_id == user_id).with_for_update()
    )
    wallet = wallet.scalar_one_or_none()
    if not wallet:
        raise NotFoundError("Wallet")

    new_balance = wallet.balance + amount
    wallet.balance = new_balance

    # admin_id is the JWT `sub` claim — a real UUID from the admin's User record
    db.add(Transaction(
        wallet_id=wallet.id, user_id=user_id,
        transaction_type=TransactionType.credit, credit_type=credit_type,
        amount=amount, reason=TransactionReason.admin_credit,
        description=description, balance_after=new_balance,
        created_by=uuid.UUID(admin_id),  # Audit: real admin UUID from DB
    ))

    return success_response({"balance": float(new_balance)}, "Coins credited")
```

**Router pattern:**
```python
# app/api/admin.py
from fastapi import APIRouter, Depends, Query
from app.core.dependencies import require_role, get_db

router = APIRouter(prefix="/admin", tags=["Admin"])


@router.get("/dashboard")
async def dashboard(
    current_user=Depends(require_role("super_admin")),
    db: AsyncSession = Depends(get_db),
):
    return await admin_service.get_dashboard_metrics(db)


@router.post("/teachers")
async def create_teacher(
    body: CreateTeacherRequest,
    current_user=Depends(require_role("super_admin")),
    db: AsyncSession = Depends(get_db),
):
    return await admin_service.create_teacher(body, db)


@router.get("/teachers")
async def list_teachers(
    page: int = Query(1, ge=1),
    limit: int = Query(20, ge=1, le=100),
    search: str = Query(None),
    current_user=Depends(require_role("super_admin")),
    db: AsyncSession = Depends(get_db),
):
    return await admin_service.get_teachers(page, limit, search, db)
```

#### 2.2 Frontend — Admin Pages

Same as [technical_prd.md](./technical_prd.md) § 3 — AdminLayout, AdminDashboard, TeacherManagement, TeacherCreate, StudentManagement, ExamCostConfig, AdminTransactions. All frontend code is identical.

**Deliverables after Task 2:**
- ✅ Admin can login and see dashboard with real metrics
- ✅ Admin can create teacher accounts (full profile)
- ✅ Admin can view/search/filter all teachers and students
- ✅ Admin can activate/deactivate/soft-delete teachers and students
- ✅ Admin can credit Devo-coins (paid/gift) to any user
- ✅ Admin can **reset teacher password** (`PATCH /admin/teachers/{id}/reset-password` with `{ new_password }`) — overrides any password the teacher set
- ✅ Admin can set per-question cost and publishing fee
- ✅ Admin can view all platform transactions
- ✅ All admin endpoints visible in Swagger at `/docs`

> [!IMPORTANT]
> `admin_service.reset_teacher_password(teacher_id, new_password, db)` must hash the new password with `hash_password()` before saving. This is a required Phase 1 endpoint per `product_feature.md § Appendix` and `technical_prd.md § 5.2`.

---

### Task 3: Teacher Panel — Student + Batch + Exam Management

**Scope:** Teacher dashboard, student linking, batch CRUD, and full exam creation with sections and questions.

#### 3.1 Backend — Teacher & Exam Services

**Files to create:**

| File | Responsibility |
|---|---|
| `app/models/exam.py` | Exam, Section, Question, ExamCostConfig models |
| `app/models/batch.py` | Batch, TeacherStudent, BatchStudent, ExamBatch models |
| `app/schemas/teacher.py` | Pydantic schemas for teacher operations |
| `app/schemas/exam.py` | Pydantic schemas for exam/question bodies |
| `app/services/teacher_service.py` | Student linking, batch management, gift coins |
| `app/services/exam_service.py` | Exam CRUD, section CRUD, question CRUD, batch assignment |
| `app/api/teacher.py` | All `/teacher/*` routes |
| `app/utils/cost_calculator.py` | `calculate_exam_cost(question_count, config)` |

**Key exam service functions:**

```python
# app/services/exam_service.py

async def create_exam(
    teacher_id: uuid.UUID, data: CreateExamRequest, db: AsyncSession
) -> dict:
    """Create exam. If global → deduct 10-coin publishing fee."""
    if data.visibility == "global":
        # Check teacher wallet
        wallet = await db.execute(
            select(Wallet).where(Wallet.user_id == teacher_id).with_for_update()
        )
        wallet = wallet.scalar_one()
        config = await db.execute(select(ExamCostConfig))
        config = config.scalar_one()

        if wallet.balance < config.global_exam_publishing_fee:
            raise InsufficientBalanceError(
                f"Need {config.global_exam_publishing_fee} coins to publish a global exam"
            )

        # Deduct publishing fee
        wallet.balance -= config.global_exam_publishing_fee
        db.add(Transaction(
            wallet_id=wallet.id, user_id=teacher_id,
            transaction_type=TransactionType.debit,
            amount=config.global_exam_publishing_fee,
            reason=TransactionReason.publishing_fee,
            description=f"Publishing fee: {data.name}",
            balance_after=wallet.balance,
        ))

    exam = Exam(
        teacher_id=teacher_id, name=data.name,
        scheduled_date=data.scheduled_date, start_time=data.start_time,
        end_time=data.end_time, duration_minutes=data.duration_minutes,
        language=data.language, visibility=data.visibility,
        positive_marks=data.positive_marks, negative_marks=data.negative_marks,
    )
    db.add(exam)
    await db.flush()
    return success_response(ExamResponse.from_orm(exam), "Exam created")


async def add_question(
    exam_id: uuid.UUID, data: QuestionRequest, db: AsyncSession
) -> dict:
    """Add question to exam. Blocked if exam is locked."""
    exam = await db.get(Exam, exam_id)
    if not exam:
        raise NotFoundError("Exam")
    if exam.is_locked:
        raise ApiError(403, "Questions are locked after the first student attempt")

    question = Question(
        exam_id=exam_id, section_id=data.section_id,
        order_index=data.order_index or 0,
        qn_en=data.qn_en, op_en_1=data.op_en_1, op_en_2=data.op_en_2,
        op_en_3=data.op_en_3, op_en_4=data.op_en_4,
        qn_bn=data.qn_bn, op_bn_1=data.op_bn_1, op_bn_2=data.op_bn_2,
        op_bn_3=data.op_bn_3, op_bn_4=data.op_bn_4,
        correct_option=data.correct_option,
        image_url=data.image_url,
        image_public_id=data.image_public_id,  # Phase 2 Cloudinary cleanup — stored in Phase 1 schema
        solution=data.solution,
    )
    db.add(question)
    await db.flush()
    return success_response(QuestionResponse.from_orm(question), "Question added")
```

#### 3.2 Frontend — Teacher Pages

Same component structure as [technical_prd.md](./technical_prd.md) § 3 — TeacherLayout, TeacherDashboard, MyStudents, MyBatches, BatchDetail, ExamList, ExamCreate (multi-step), ExamDetail, WalletPage, TeacherProfile.

**Deliverables after Task 3:**
- ✅ Teacher dashboard with real metrics
- ✅ Search students by email, add/link (pending + active)
- ✅ Gift coins from teacher wallet to student
- ✅ Batch CRUD + add/remove students
- ✅ Full exam creation (settings + sections + questions)
- ✅ Question lock enforcement (after first attempt)
- ✅ Batch assignment for private exams
- ✅ Active/inactive toggle + soft delete for exams
- ✅ Global exam publishing fee deduction (10 coins)

---

### Task 4: Student Panel — Home + Exam Attempt + Results

**Scope:** Student home page, exam browsing, full exam attempt experience (fullscreen, timer, question nav), submission, scoring, and results.

#### 4.1 Backend — Student Service

**Files to create:**

| File | Responsibility |
|---|---|
| `app/models/attempt.py` | ExamAttempt, Violation models |
| `app/schemas/student.py` | Pydantic schemas |
| `app/services/student_service.py` | Exam listing, start, submit, results |
| `app/api/student.py` | All `/student/*` routes |
| `app/utils/score_calculator.py` | `calculate_score(questions, answers, pos, neg)` |

**Score calculator:**
```python
# app/utils/score_calculator.py
from decimal import Decimal
from typing import Dict, List, NamedTuple


class ScoreResult(NamedTuple):
    score: Decimal
    total_marks: Decimal
    correct: int
    wrong: int
    unanswered: int


def calculate_score(
    questions: List[dict],
    answers: Dict[str, str],
    positive_marks: Decimal,
    negative_marks: Decimal,
) -> ScoreResult:
    correct = 0
    wrong = 0
    unanswered = 0

    for q in questions:
        student_answer = answers.get(str(q["id"]))
        if not student_answer:
            unanswered += 1
        elif student_answer == q["correct_option"]:
            correct += 1
        else:
            wrong += 1

    score = (Decimal(correct) * positive_marks) - (Decimal(wrong) * negative_marks)
    total_marks = Decimal(len(questions)) * positive_marks

    return ScoreResult(score, total_marks, correct, wrong, unanswered)
```

**`start_exam` and `submit_exam`:** See [technical_prd.md](./technical_prd.md) § 5.5 for complete implementation.

#### 4.2 Frontend — Student Pages

Same as [technical_prd.md](./technical_prd.md) — StudentLayout, StudentHome (carousel + tabs), ExamAttemptPage (fullscreen), ExamResult, MyResults, WalletPage.

**Key frontend files (identical to previous version):**
- `src/store/examStore.js` — Zustand store for exam attempt state
- `src/hooks/useExamTimer.js` — Timer countdown with auto-submit
- `src/hooks/useFullscreen.js` — Fullscreen API + violation detection
- `src/components/exam/ExamAttempt.jsx` — Core exam UI
- `src/components/exam/QuestionNav.jsx` — Question palette
- `src/components/exam/Timer.jsx` — Countdown display
- `src/components/exam/CostPreview.jsx` — Pre-attempt cost modal

**Deliverables after Task 4:**
- ✅ Student home with category carousel and My Exam / All Exam tabs
- ✅ Exam cards with Start Exam / View Solution states
- ✅ Cost preview + wallet check + deduction on start
- ✅ Full exam attempt UI (fullscreen, timer, question palette)
- ✅ Answer selection, clearing, mark-for-review
- ✅ Auto-submit on timer expiry
- ✅ Violation detection (fullscreen exit)
- ✅ Server-side score calculation
- ✅ Post-attempt solution view
- ✅ Student wallet + transaction history

---

### Task 5: Integration, Polish, Testing & Deployment

**Scope:** Wire everything together, fix edge cases, build UI components, test critical flows, deploy.

#### 5.1 Edge Cases to Handle

| Edge Case | Implementation |
|---|---|
| Teacher adds unregistered email → student registers later | On registration, query `teacher_students` by `invited_email` → flip to active |
| Teacher soft-deleted → exams hidden | `get_active_exams_query()` in `exam_service.py` must JOIN `users` table: `.join(User, Exam.teacher_id == User.id).where(Exam.is_deleted == False, User.is_deleted == False)`. Use this shared query builder for ALL student-facing exam endpoints. |
| Student inactive → blocked from everything | `get_current_user` dependency checks `is_active` on every request |
| Mid-exam browser close — abandoned attempt | Attempt stays `is_submitted=False`. On `GET /student/exams/{id}`, if an `is_submitted=False` attempt exists and `attempted_at + duration_minutes + 30 min buffer < now`, treat it as timed-out/incomplete. Show a "Exam timed out" result page. The student cannot re-attempt (coins already deducted per product policy). |
| Race condition: concurrent exam start | `UniqueConstraint("exam_id", "student_id")` + `with_for_update()` row lock on Wallet row only. Wrap `db.flush()` in `try/except IntegrityError` → return `409 Conflict` as defense-in-depth. |
| Negative wallet balance prevention | Wallet check inside same transaction as deduction |
| Negative marks entered as negative number | Pydantic schema enforces `positive_marks > 0` and `negative_marks >= 0` (use `ge=0`) to prevent score formula inversion |
| Exam time-window not enforced | In `start_exam`, after checking `exam.is_active`, validate the current server time is within `[scheduled_date + start_time, scheduled_date + end_time]`. Use `datetime.combine(exam.scheduled_date, datetime.strptime(exam.start_time, "%H:%M").time())`. Return `400 "Exam has not started yet"` or `400 "Exam window has passed"` accordingly. |
| Section deleted on locked exam | `DELETE /teacher/exams/{exam_id}/sections/{id}` must check `exam.is_locked`. If `True`, return `403 "Cannot delete sections after the exam has been attempted"`. This prevents bypassing question lock via cascade delete. |
| Focused zone / exam validation | `focused_zones` (teacher) and `focused_exam` (student) must be validated against an allowed list. Add `ExamCategory` enum: `NEET, JEE, SSC, UPSC, Banking`. Use `Literal` in Pydantic schemas. |

#### 5.2 Shared UI Components

```
src/components/ui/
├── Button.jsx        → variants: primary, secondary, danger, ghost; loading state
├── Input.jsx         → label, error, icon
├── Select.jsx        → single select with search
├── Modal.jsx         → overlay, close on escape
├── Toast.jsx         → react-hot-toast integration
├── Card.jsx          → shadow, padding, hover
├── Badge.jsx         → active (green), inactive (gray), pending (yellow)
├── Table.jsx         → sortable columns, loading skeleton
├── Pagination.jsx    → page numbers + prev/next
├── Spinner.jsx       → loading spinner
├── EmptyState.jsx    → icon + message + action button
└── ConfirmDialog.jsx → cancel/confirm
```

#### 5.3 Testing

**Backend tests (`pytest + httpx`):**

```python
# tests/conftest.py
# ⚠️ Uses isolated test engine with per-test rollback for data isolation.
# Do NOT use the production async_session_factory here.
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from app.main import app
from app.core.database import get_db
from app.core.config import settings

# Separate test engine — points to TEST database (set via env in CI)
test_engine = create_async_engine(settings.DATABASE_URL, echo=False)
TestSessionLocal = async_sessionmaker(test_engine, class_=AsyncSession, expire_on_commit=False)


@pytest_asyncio.fixture
async def db_session():
    """Yields a session that rolls back after each test — no data pollution."""
    async with TestSessionLocal() as session:
        yield session
        await session.rollback()  # ← KEY: undo all test writes


@pytest_asyncio.fixture
async def client(db_session):
    """Override get_db to use the isolated test session."""
    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()


@pytest_asyncio.fixture
async def admin_headers(client):
    """Auth headers for Super Admin — uses env vars, not hardcoded strings."""
    response = await client.post("/api/v1/auth/admin/login", json={
        "email": settings.ADMIN_EMAIL,
        "password": settings.ADMIN_PASSWORD,
    })
    token = response.json()["data"]["tokens"]["access_token"]
    return {"Authorization": f"Bearer {token}"}
```

```python
# tests/integration/test_auth.py
import pytest
from app.core.config import settings  # Always use env vars — never hardcode credentials

@pytest.mark.asyncio
async def test_admin_login_success(client):
    response = await client.post("/api/v1/auth/admin/login", json={
        "email": settings.ADMIN_EMAIL,      # From .env — matches CI env vars
        "password": settings.ADMIN_PASSWORD,
    })
    assert response.status_code == 200
    data = response.json()
    assert "access_token" in data["data"]["tokens"]

@pytest.mark.asyncio
async def test_admin_login_wrong_password(client):
    response = await client.post("/api/v1/auth/admin/login", json={
        "email": settings.ADMIN_EMAIL,
        "password": "wrong-password-xyz",
    })
    assert response.status_code == 401

@pytest.mark.asyncio
async def test_student_register_and_login(client):
    # Register
    response = await client.post("/api/v1/auth/student/register", json={
        "name": "Test Student", "phone": "9876543210",
        "institute_name": "Test School", "focused_exam": "NEET",
        "email": "student@test.com", "password": "testpass123"
    })
    assert response.status_code == 200
    # ... verify OTP, login, etc.
```

**Run tests:**
```bash
cd backend
pytest tests/ -v --tb=short
```

#### 5.4 Deployment

**Backend → Render:**
- New Web Service → Docker → Root directory: `backend`
- Build command: `pip install -r requirements.txt`
- Start command: `python -m scripts.startup && uvicorn app.main:app --host 0.0.0.0 --port $PORT`

> [!IMPORTANT]
> **No separate seed command needed.** Seed data (ExamCostConfig) is embedded in Alembic migration `005_init_attempt_tables_and_seed`. It runs exactly once when the migration is first applied, tracked by the Alembic `alembic_version` stamp. The `scripts/startup.py` script checks the current DB stamp against head — if they match, no migrations run. Only new, unapplied revisions are executed. This is safe for every deploy.
- Set all environment variables

**Frontend → Vercel:**
- Connect GitHub → auto-deploy on push
- Root directory: `frontend`
- Framework: Vite
- Build command: `npm run build`
- Set `VITE_API_BASE_URL=https://your-backend.onrender.com/api/v1`
- Add `vercel.json` for SPA routing:
```json
{ "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }] }
```

**Post-deployment checklist:**
```
[ ] FastAPI Swagger docs accessible at /docs
[ ] Admin login works
[ ] CORS allows frontend domain
[ ] Database connection stable (Neon DB)
[ ] OTP emails sending via SMTP
[ ] JWT tokens working
[ ] Full exam attempt flow end-to-end
[ ] Wallet operations atomic and correct
[ ] No 500 errors in production
```

---

## Summary — Phase 1 Deliverables

When Phase 1 ships:

- **Admin:** Fully manages teachers, students, wallets, and pricing
- **Teachers:** Create exams, manage students/batches, assign exams, view results
- **Students:** Register, browse exams, attempt with fullscreen proctoring, view scores
- **Platform revenue** generated through coin deductions on every exam attempt
- **API docs** automatically available at `/docs` (Swagger) and `/redoc`
- **Database migrations** versioned and reviewable via Alembic

---

*For Phase 2 features (analytics, image uploads, bilingual UI, notifications), see [phase2_plan.md](./phase2_plan.md)*
