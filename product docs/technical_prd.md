# 📐 Technical PRD — Devophile CBT Platform

> **Version:** 2.0  
> **Author:** Technical Architect  
> **Date:** 2026-06-13  
> **Status:** Architecture Finalized (FastAPI)  
> **Reference:** [product_feature.md](./product_feature.md)

---

## Table of Contents

1. [Tech Stack & Rationale](#1-tech-stack--rationale)
2. [System Architecture](#2-system-architecture)
3. [Project Structure & File Tree](#3-project-structure--file-tree)
4. [Database Schema & Migrations (Alembic)](#4-database-schema--migrations-alembic)
5. [API Documentation (REST)](#5-api-documentation-rest)
6. [Authentication & Authorization](#6-authentication--authorization)
7. [Component Architecture (Frontend)](#7-component-architecture-frontend)
8. [Wallet & Subscription Logic Gates](#8-wallet--subscription-logic-gates)
9. [SEO & Meta Management](#9-seo--meta-management)
10. [Security Architecture](#10-security-architecture)
11. [Environment Variables](#11-environment-variables)
12. [Docker & DevOps](#12-docker--devops)
13. [Phase-wise Roadmap Summary](#13-phase-wise-roadmap-summary)

---

## 1. Tech Stack & Rationale

| Layer | Technology | Rationale |
|---|---|---|
| **Frontend** | React 18 + Vite + Tailwind CSS + react-icons | Already initialized in repo. Fast HMR, utility-first CSS, responsive out-of-box |
| **Backend** | **Python 3.12 + FastAPI** | Auto-generated Swagger docs, native async, Pydantic validation built-in, best AI agent code generation |
| **ORM** | **SQLAlchemy 2.0 (async)** | Industry-standard Python ORM, explicit relationship control, excellent PostgreSQL support |
| **Migrations** | **Alembic** | Versioned, reviewable migration files with upgrade/downgrade, full control over every schema change |
| **Database** | PostgreSQL 16 (Neon DB) | Relational integrity critical for wallet transactions; JSONB for flexible answer storage; easy migration to self-hosted PG on VPS/Oracle later |
| **Validation** | **Pydantic v2** (built-in to FastAPI) | Zero-config request/response validation, auto-generates OpenAPI schemas |
| **Cache** | In-memory dict → Redis (Phase 3) | Start simple; exam cost config cached in-process. Swap to Redis when scaling horizontally |
| **Email** | **fastapi-mail** (SMTP) | Phase 1 MVP. Abstracted behind `EmailService` class for drop-in SES replacement |
| **File Storage** | Cloudinary (Phase 2) | Abstracted behind `StorageService` class for drop-in S3/Oracle Object Storage replacement |
| **Auth** | **python-jose** (JWT) + **passlib[bcrypt]** | Industry standard. Access token (15min) + Refresh token (7d) rotation |
| **State Mgmt** | Zustand (frontend) | Lightweight, no boilerplate, perfect for wallet/auth/exam state |
| **HTTP Client** | Axios with interceptors (frontend) | Auto-attach JWT, handle 401 refresh, centralized error handling |
| **Charts** | Recharts (Phase 2) | React-native charting for analytics dashboards |
| **Testing** | **pytest + httpx** (backend) | Pythonic testing, async support, excellent fixtures |
| **Deployment** | Vercel (Frontend) + Render (Backend) | Free tier friendly for launch; Docker-ready for cloud migration |

### Abstraction Strategy (Future Migration Ready)

```
┌─────────────────────────────────────────────────┐
│                   Core Logic                     │
│  (Routers, Services, Business Rules)             │
├─────────────────────────────────────────────────┤
│              Service Interfaces                  │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ IEmail   │  │ IStorage │  │ ICache        │  │
│  │ Service  │  │ Service  │  │ Service       │  │
│  └────┬─────┘  └────┬─────┘  └──────┬────────┘  │
│       │              │               │           │
│  ┌────▼─────┐  ┌────▼─────┐  ┌──────▼────────┐  │
│  │fastapi-  │  │Cloudinary│  │ dict cache    │  │
│  │mail      │  │(Phase 2) │  │ (Phase 1)     │  │
│  │(Phase 1) │  │   ↓      │  │    ↓          │  │
│  │   ↓      │  │ S3/Oracle│  │  Redis        │  │
│  │ AWS SES  │  │(Phase 3) │  │ (Phase 3)     │  │
│  │(Phase 3) │  └──────────┘  └───────────────┘  │
│  └──────────┘                                    │
└─────────────────────────────────────────────────┘
```

---

## 2. System Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                        CLIENT (Browser)                            │
│                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │
│  │  /admin/*    │  │  /teacher/*  │  │  /student/*              │ │
│  │  Admin SPA   │  │  Teacher SPA │  │  Student SPA             │ │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────────┘ │
│         └─────────────────┼─────────────────────┘                 │
│                           │ HTTPS (Axios + JWT)                   │
└───────────────────────────┼───────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│                  API SERVER (FastAPI + Uvicorn)                    │
│                                                                    │
│  ┌─────────┐  ┌──────────┐  ┌────────────┐  ┌────────────────┐  │
│  │  CORS   │  │ Trusted  │  │ Rate Limit │  │ Access Logger  │  │
│  │Middleware│  │  Host    │  │ (SlowAPI)  │  │ (Middleware)   │  │
│  └─────────┘  └──────────┘  └────────────┘  └────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │              ROUTER LAYER (/api/v1/*)                       │  │
│  │  /auth  /admin  /teacher  /student  /exam  /wallet          │  │
│  └────────────────────────┬────────────────────────────────────┘  │
│                           │                                       │
│  ┌────────────────────────▼────────────────────────────────────┐  │
│  │            DEPENDENCY INJECTION LAYER                       │  │
│  │  get_current_user() → require_role(roles) → get_db()        │  │
│  └────────────────────────┬────────────────────────────────────┘  │
│                           │                                       │
│  ┌────────────────────────▼────────────────────────────────────┐  │
│  │            SERVICE LAYER (Business Logic)                   │  │
│  │  AuthService, ExamService, WalletService, EmailService,     │  │
│  │  StorageService, NotificationService                        │  │
│  └────────────────────────┬────────────────────────────────────┘  │
│                           │                                       │
│  ┌────────────────────────▼────────────────────────────────────┐  │
│  │            DATA ACCESS LAYER (SQLAlchemy 2.0 Async)         │  │
│  │  session.execute(select(User).where(...))                   │  │
│  └────────────────────────┬────────────────────────────────────┘  │
│                           │                                       │
└───────────────────────────┼───────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│               PostgreSQL (Neon DB / Self-hosted)                  │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌───────────────┐  │
│  │ users  │ │ wallet │ │ exams  │ │ quest. │ │ exam_attempts │  │
│  └────────┘ └────────┘ └────────┘ └────────┘ └───────────────┘  │
└───────────────────────────────────────────────────────────────────┘
```

---

## 3. Project Structure & File Tree

```
CBT_PLATFORM/
├── product docs/                    # Product documentation (this folder)
│   ├── product_feature.md
│   ├── technical_prd.md             # ← THIS FILE
│   ├── phase1_plan.md
│   ├── phase2_plan.md
│   └── phase3_plan.md
│
├── frontend/                        # React + Vite + Tailwind
│   ├── public/
│   │   ├── favicon.ico
│   │   └── robots.txt
│   ├── src/
│   │   ├── main.jsx                 # App entry point
│   │   ├── App.jsx                  # Root component with router
│   │   ├── index.css                # Tailwind directives + global styles
│   │   │
│   │   ├── api/                     # API layer
│   │   │   ├── axiosInstance.js      # Axios config, interceptors, token refresh
│   │   │   ├── authApi.js           # login, register, verifyOtp, refreshToken
│   │   │   ├── adminApi.js          # teacher CRUD, student mgmt, config
│   │   │   ├── teacherApi.js        # students, batches, exams
│   │   │   ├── studentApi.js        # exam listing, attempt, results
│   │   │   └── walletApi.js         # balance, transactions, credit
│   │   │
│   │   ├── store/                   # Zustand state management
│   │   │   ├── authStore.js         # user, tokens, role, isAuthenticated
│   │   │   ├── examStore.js         # current exam state (during attempt)
│   │   │   ├── walletStore.js       # balance, transactions cache
│   │   │   └── uiStore.js          # sidebar, modals, toasts, loading
│   │   │
│   │   ├── hooks/                   # Custom React hooks
│   │   │   ├── useAuth.js           # Auth state + guards
│   │   │   ├── useExamTimer.js      # Countdown timer logic
│   │   │   ├── useFullscreen.js     # Fullscreen API + violation detection
│   │   │   ├── useWallet.js         # Balance fetching + formatting
│   │   │   └── useDebounce.js       # Input debouncing
│   │   │
│   │   ├── utils/                   # Utility functions
│   │   │   ├── constants.js         # Role enums, exam types, routes
│   │   │   ├── formatters.js        # Date, currency, score formatting
│   │   │   ├── validators.js        # Client-side validation helpers
│   │   │   └── examHelpers.js       # Score calculation, answer mapping
│   │   │
│   │   ├── components/              # Shared/reusable components
│   │   │   ├── ui/                  # Atomic UI components
│   │   │   │   ├── Button.jsx
│   │   │   │   ├── Input.jsx
│   │   │   │   ├── Select.jsx
│   │   │   │   ├── Modal.jsx
│   │   │   │   ├── Toast.jsx
│   │   │   │   ├── Card.jsx
│   │   │   │   ├── Badge.jsx
│   │   │   │   ├── Table.jsx
│   │   │   │   ├── Pagination.jsx
│   │   │   │   ├── Spinner.jsx
│   │   │   │   ├── EmptyState.jsx
│   │   │   │   └── ConfirmDialog.jsx
│   │   │   │
│   │   │   ├── layout/              # Layout components
│   │   │   │   ├── AdminLayout.jsx   # Sidebar + topbar for /admin
│   │   │   │   ├── TeacherLayout.jsx # Sidebar + topbar for /teacher
│   │   │   │   ├── StudentLayout.jsx # Bottom nav + topbar for /student
│   │   │   │   ├── Sidebar.jsx
│   │   │   │   ├── Topbar.jsx
│   │   │   │   └── BottomNav.jsx
│   │   │   │
│   │   │   ├── auth/                # Auth-related components
│   │   │   │   ├── LoginForm.jsx     # Reusable login (teacher/student)
│   │   │   │   ├── RegisterForm.jsx  # Student registration
│   │   │   │   ├── OtpInput.jsx      # 6-digit OTP input
│   │   │   │   ├── ForgotPassword.jsx
│   │   │   │   └── ProtectedRoute.jsx # Role-based route guard
│   │   │   │
│   │   │   ├── exam/                # Exam-related components
│   │   │   │   ├── ExamCard.jsx      # Exam listing card
│   │   │   │   ├── ExamForm.jsx      # Create/edit exam settings
│   │   │   │   ├── SectionManager.jsx # Add/remove sections
│   │   │   │   ├── QuestionForm.jsx  # Add question (single/bilingual)
│   │   │   │   ├── QuestionList.jsx  # List questions in exam
│   │   │   │   ├── ExamAttempt.jsx   # Full exam attempt UI
│   │   │   │   ├── QuestionNav.jsx   # Question palette/navigator
│   │   │   │   ├── Timer.jsx         # Countdown timer display
│   │   │   │   ├── CostPreview.jsx   # Pre-attempt cost confirmation
│   │   │   │   └── SolutionView.jsx  # Post-attempt solution review
│   │   │   │
│   │   │   ├── wallet/              # Wallet components
│   │   │   │   ├── WalletCard.jsx    # Balance display widget
│   │   │   │   ├── TransactionList.jsx # Transaction history
│   │   │   │   └── CreditModal.jsx   # Admin/teacher coin credit form
│   │   │   │
│   │   │   ├── student/             # Student-specific components
│   │   │   │   ├── StudentTable.jsx  # Student listing table
│   │   │   │   ├── StudentSearch.jsx # Email search + add
│   │   │   │   └── StudentProfile.jsx # Profile detail view
│   │   │   │
│   │   │   └── batch/               # Batch management components
│   │   │       ├── BatchCard.jsx
│   │   │       ├── BatchForm.jsx
│   │   │       └── BatchStudentAssign.jsx
│   │   │
│   │   ├── pages/                   # Page-level components (route targets)
│   │   │   ├── Landing.jsx          # Public landing page (SEO)
│   │   │   │
│   │   │   ├── admin/
│   │   │   │   ├── AdminLogin.jsx
│   │   │   │   ├── AdminDashboard.jsx
│   │   │   │   ├── TeacherManagement.jsx
│   │   │   │   ├── TeacherCreate.jsx
│   │   │   │   ├── StudentManagement.jsx
│   │   │   │   ├── ExamCostConfig.jsx
│   │   │   │   └── AdminTransactions.jsx
│   │   │   │
│   │   │   ├── teacher/
│   │   │   │   ├── TeacherLogin.jsx
│   │   │   │   ├── TeacherDashboard.jsx
│   │   │   │   ├── MyStudents.jsx
│   │   │   │   ├── MyBatches.jsx
│   │   │   │   ├── BatchDetail.jsx
│   │   │   │   ├── ExamList.jsx
│   │   │   │   ├── ExamCreate.jsx
│   │   │   │   ├── ExamDetail.jsx
│   │   │   │   ├── ExamAnalytics.jsx
│   │   │   │   ├── StudentProfileView.jsx
│   │   │   │   ├── WalletPage.jsx
│   │   │   │   └── TeacherProfile.jsx
│   │   │   │
│   │   │   └── student/
│   │   │       ├── StudentLogin.jsx
│   │   │       ├── StudentRegister.jsx
│   │   │       ├── StudentHome.jsx
│   │   │       ├── ExamAttemptPage.jsx
│   │   │       ├── ExamResult.jsx
│   │   │       ├── MyResults.jsx
│   │   │       ├── WalletPage.jsx
│   │   │       └── StudentProfile.jsx
│   │   │
│   │   └── styles/
│   │       └── exam-attempt.css     # Exam-specific fullscreen styles
│   │
│   ├── index.html
│   ├── package.json
│   ├── vite.config.js
│   ├── tailwind.config.js
│   ├── postcss.config.js
│   └── .env.example
│
├── backend/                         # Python + FastAPI + SQLAlchemy + Alembic
│   ├── alembic/                     # Alembic migrations
│   │   ├── env.py                   # Alembic environment config
│   │   ├── script.py.mako           # Migration template
│   │   └── versions/               # Migration files (auto-generated)
│   │       ├── 001_init_core_tables.py
│   │       ├── 002_init_auth_tables.py
│   │       ├── 003_init_exam_tables.py
│   │       ├── 004_init_batch_tables.py
│   │       └── 005_init_attempt_tables.py
│   │
│   ├── alembic.ini                  # Alembic config
│   │
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py                  # FastAPI app entry point + lifespan
│   │   │
│   │   ├── core/                    # Configuration & cross-cutting concerns
│   │   │   ├── __init__.py
│   │   │   ├── config.py            # Pydantic Settings (env vars)
│   │   │   ├── database.py          # SQLAlchemy async engine + session factory
│   │   │   ├── security.py          # JWT creation/verification, password hashing
│   │   │   └── dependencies.py      # FastAPI dependencies (get_db, get_current_user)
│   │   │
│   │   ├── models/                  # SQLAlchemy ORM models
│   │   │   ├── __init__.py          # Import all models (for Alembic auto-detect)
│   │   │   ├── base.py              # Base model class + common mixins
│   │   │   ├── user.py              # User, TeacherProfile, StudentProfile
│   │   │   ├── wallet.py            # Wallet, Transaction
│   │   │   ├── exam.py              # Exam, Section, Question, ExamCostConfig
│   │   │   ├── batch.py             # Batch, TeacherStudent, BatchStudent, ExamBatch
│   │   │   ├── attempt.py           # ExamAttempt, Violation
│   │   │   ├── notification.py      # Notification
│   │   │   └── auth.py              # Otp, RefreshToken
│   │   │
│   │   ├── schemas/                 # Pydantic request/response schemas
│   │   │   ├── __init__.py
│   │   │   ├── auth.py              # LoginRequest, RegisterRequest, TokenResponse, etc.
│   │   │   ├── admin.py             # CreateTeacherRequest, CreditWalletRequest, etc.
│   │   │   ├── teacher.py           # LinkStudentRequest, CreateBatchRequest, etc.
│   │   │   ├── student.py           # ExamListResponse, SubmitExamRequest, etc.
│   │   │   ├── exam.py              # CreateExamRequest, QuestionRequest, etc.
│   │   │   ├── wallet.py            # WalletResponse, TransactionResponse, etc.
│   │   │   └── common.py            # Pagination, ApiResponse, ErrorResponse
│   │   │
│   │   ├── api/                     # Route definitions (routers)
│   │   │   ├── __init__.py
│   │   │   ├── router.py            # Main router aggregator (/api/v1/*)
│   │   │   ├── auth.py              # /auth/* routes
│   │   │   ├── admin.py             # /admin/* routes
│   │   │   ├── teacher.py           # /teacher/* routes
│   │   │   ├── student.py           # /student/* routes
│   │   │   ├── exam.py              # /exam/* routes
│   │   │   └── wallet.py            # /wallet/* routes
│   │   │
│   │   ├── services/                # Business logic layer
│   │   │   ├── __init__.py
│   │   │   ├── auth_service.py      # Login, register, OTP, JWT
│   │   │   ├── admin_service.py     # Teacher CRUD, student mgmt, config
│   │   │   ├── teacher_service.py   # Student linking, batch, exam mgmt
│   │   │   ├── student_service.py   # Exam listing, attempt, results
│   │   │   ├── exam_service.py      # Exam CRUD, questions, sections
│   │   │   ├── wallet_service.py    # Credit, debit, balance, transactions
│   │   │   ├── email_service.py     # IEmailService → fastapi-mail impl
│   │   │   ├── storage_service.py   # IStorageService → Cloudinary impl
│   │   │   ├── notification_service.py  # In-app notifications
│   │   │   └── otp_service.py       # OTP generation, storage, verification
│   │   │
│   │   └── utils/                   # Utility functions
│   │       ├── __init__.py
│   │       ├── cost_calculator.py   # Exam cost calculation logic
│   │       ├── score_calculator.py  # Student score calculation
│   │       ├── exceptions.py        # Custom exception classes
│   │       └── responses.py         # Standardized response helpers
│   │
│   ├── tests/                       # Test files
│   │   ├── __init__.py
│   │   ├── conftest.py              # pytest fixtures (test DB, client, auth headers)
│   │   ├── unit/
│   │   │   ├── test_score_calculator.py
│   │   │   └── test_cost_calculator.py
│   │   └── integration/
│   │       ├── test_auth.py
│   │       ├── test_exam.py
│   │       └── test_wallet.py
│   │
│   ├── requirements.txt             # Python dependencies
│   ├── pyproject.toml               # Project config (alternative to requirements.txt)
│   ├── .env.example
│   ├── Dockerfile
│   └── .dockerignore
│
├── docker-compose.yml               # Full stack local dev
├── .gitignore
└── README.md
```

---

## 4. Database Schema & Migrations (Alembic)

### 4.0 Entity Relationship Diagram

> Complete ER diagram showing all 15 tables, their columns, data types, and relationships.

```mermaid
erDiagram
    users {
        UUID id PK
        enum role "super_admin | teacher | student"
        varchar email UK
        varchar password
        varchar name
        varchar phone
        boolean is_active
        boolean is_deleted
        varchar active_session_id
        timestamp created_at
        timestamp updated_at
    }

    teacher_profiles {
        UUID id PK
        UUID user_id FK,UK
        varchar image_url
        varchar subject
        text[] focused_zones "NEET, JEE, SSC"
        int teaching_since
        text[] degrees
        varchar locality
        varchar city
        varchar state
    }

    student_profiles {
        UUID id PK
        UUID user_id FK,UK
        varchar institute_name
        varchar focused_exam "NEET | JEE | SSC"
    }

    wallets {
        UUID id PK
        UUID user_id FK,UK
        decimal balance "Decimal 12,4"
        timestamp updated_at
    }

    transactions {
        UUID id PK
        UUID wallet_id FK
        UUID user_id FK
        enum transaction_type "credit | debit"
        enum credit_type "paid | gift | system"
        decimal amount "Decimal 12,4"
        enum reason "admin_credit | gift | exam_attempt | publishing_fee | teacher_gift_to_student"
        varchar description
        UUID reference_id "nullable exam/user ref"
        decimal balance_after "Decimal 12,4"
        timestamp created_at
    }

    exam_cost_config {
        UUID id PK
        decimal cost_per_question "Decimal 10,4"
        decimal global_exam_publishing_fee "Decimal 10,4"
        UUID updated_by
        timestamp updated_at
    }

    exams {
        UUID id PK
        UUID teacher_id FK
        varchar name
        date scheduled_date
        varchar start_time "HH:mm"
        varchar end_time "HH:mm"
        int duration_minutes
        enum language "single | bilingual"
        enum visibility "private | global"
        decimal positive_marks "Decimal 5,2"
        decimal negative_marks "Decimal 5,2"
        boolean is_active
        boolean is_deleted
        boolean is_locked "true after first attempt"
        timestamp created_at
        timestamp updated_at
    }

    sections {
        UUID id PK
        UUID exam_id FK
        varchar name
    }

    questions {
        UUID id PK
        UUID exam_id FK
        UUID section_id FK
        int order_index
        text qn_en
        text op_en_1
        text op_en_2
        text op_en_3
        text op_en_4
        text qn_bn "nullable - bilingual"
        text op_bn_1 "nullable - bilingual"
        text op_bn_2 "nullable - bilingual"
        text op_bn_3 "nullable - bilingual"
        text op_bn_4 "nullable - bilingual"
        varchar correct_option "1 | 2 | 3 | 4"
        varchar image_url "nullable - Phase 2"
        varchar image_public_id "nullable - Phase 2 Cloudinary cleanup"
        text solution "nullable"
    }

    batches {
        UUID id PK
        UUID teacher_id FK
        varchar name
        timestamp created_at
    }

    teacher_students {
        UUID id PK
        UUID teacher_id FK
        UUID student_id FK "nullable until registered"
        varchar invited_email
        enum status "pending | active"
        boolean is_active
        timestamp linked_at
        timestamp created_at
    }

    batch_students {
        UUID id PK
        UUID batch_id FK
        UUID student_id FK
    }

    exam_batches {
        UUID id PK
        UUID exam_id FK
        UUID batch_id FK
    }

    exam_attempts {
        UUID id PK
        UUID exam_id FK
        UUID student_id FK
        jsonb answers "question_id to selected_option"
        decimal score "Decimal 8,2"
        decimal total_marks "Decimal 8,2"
        int time_taken_seconds
        decimal coin_deducted "Decimal 12,4"
        decimal cost_per_question_snapshot "Decimal 10,4"
        varchar wallet_deducted_from "teacher | student"
        timestamp attempted_at
        timestamp submitted_at
        boolean is_submitted
    }

    violations {
        UUID id PK
        UUID attempt_id FK
        UUID student_id FK
        varchar type "fullscreen_exit"
        timestamp recorded_at
    }

    otps {
        UUID id PK
        varchar email
        varchar code "6-digit"
        varchar purpose "registration | forgot_password"
        timestamp expires_at
        boolean is_used
        timestamp created_at
    }

    refresh_tokens {
        UUID id PK
        UUID user_id
        varchar token UK
        timestamp expires_at
        timestamp created_at
    }

    notifications {
        UUID id PK
        UUID user_id FK
        enum type "low_balance | exam_blocked | system"
        varchar title
        text message
        boolean is_read
        jsonb metadata
        timestamp created_at
    }

    %% ─── Relationships ───

    users ||--o| teacher_profiles : "has profile"
    users ||--o| student_profiles : "has profile"
    users ||--|| wallets : "has wallet"
    users ||--o{ exams : "creates (teacher)"
    users ||--o{ exam_attempts : "attempts (student)"
    users ||--o{ violations : "commits"
    users ||--o{ notifications : "receives"
    users ||--o{ transactions : "has"
    users ||--o{ batches : "creates (teacher)"

    wallets ||--o{ transactions : "records"

    exams ||--o{ sections : "contains"
    exams ||--o{ questions : "contains"
    exams ||--o{ exam_attempts : "has"
    exams ||--o{ exam_batches : "assigned to"

    sections ||--o{ questions : "groups"

    exam_attempts ||--o{ violations : "has"

    batches ||--o{ batch_students : "contains"
    batches ||--o{ exam_batches : "assigned to"

    teacher_students }o--|| users : "teacher_id"
    teacher_students }o--o| users : "student_id (nullable)"
    batch_students }o--|| users : "student_id"
```

#### Relationship Summary

| Relationship | Type | Constraint |
|---|---|---|
| User → TeacherProfile | 1:1 | `user_id UNIQUE` |
| User → StudentProfile | 1:1 | `user_id UNIQUE` |
| User → Wallet | 1:1 | `user_id UNIQUE` |
| Wallet → Transactions | 1:N | `wallet_id FK` |
| User (teacher) → Exams | 1:N | `teacher_id FK` |
| Exam → Sections | 1:N | `exam_id FK, CASCADE delete` |
| Section → Questions | 1:N | `section_id FK, CASCADE delete` |
| Exam → Questions | 1:N | `exam_id FK, CASCADE delete` |
| Exam ↔ Batches | N:M | via `exam_batches` join table |
| Batch ↔ Students | N:M | via `batch_students` join table |
| Teacher ↔ Students | N:M | via `teacher_students` (with invite flow) |
| Exam + Student → ExamAttempt | 1:1 | `UNIQUE(exam_id, student_id)` |
| ExamAttempt → Violations | 1:N | `attempt_id FK, CASCADE delete` |
| User → Notifications | 1:N | `user_id FK` |

#### Key Design Decisions

| Decision | Rationale |
|---|---|
| **`teacher_students.student_id` is nullable** | Teacher can invite by email before student registers. On registration, `student_id` is populated and `status` flips to `active`. |
| **`exam_attempts` has UNIQUE(exam_id, student_id)** | One attempt per student per exam — enforced at DB level, not just application level. |
| **`is_locked` on Exam** | Once the first student submits, questions can't be edited. Preserves exam integrity. |
| **`cost_per_question_snapshot` on ExamAttempt** | Records the cost rate at time of attempt. Even if admin changes pricing later, historical records stay accurate. |
| **`wallet_deducted_from` on ExamAttempt** | Tracks whether teacher or student paid. Private exams → teacher pays. Global exams → student pays. |
| **`answers` as JSONB** | Flexible `{questionId: selectedOption}` map. No need for a separate answers table. Fast read/write. |
| **Bengali columns always present** | Created in Phase 1, nullable. Avoids schema migration when Phase 2 enables bilingual UI. |

---

### 4.1 SQLAlchemy Models

#### Base Model (`app/models/base.py`)

```python
import uuid
from datetime import datetime
from sqlalchemy import DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy.dialects.postgresql import UUID


class Base(DeclarativeBase):
    pass


class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), onupdate=func.now()
    )


class UUIDMixin:
    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), primary_key=True, default=uuid.uuid4
    )
```

#### User Models (`app/models/user.py`)

```python
import uuid
import enum
from datetime import datetime
from typing import Optional, List
from sqlalchemy import String, Boolean, Enum, Integer, ARRAY, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.dialects.postgresql import UUID
from app.models.base import Base, UUIDMixin, TimestampMixin


class Role(str, enum.Enum):
    super_admin = "super_admin"
    teacher = "teacher"
    student = "student"


class User(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "users"

    role: Mapped[Role] = mapped_column(Enum(Role), nullable=False)
    email: Mapped[str] = mapped_column(String(255), unique=True, nullable=False, index=True)
    password: Mapped[str] = mapped_column(String(255), nullable=False)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    phone: Mapped[Optional[str]] = mapped_column(String(20))
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    is_deleted: Mapped[bool] = mapped_column(Boolean, default=False)
    active_session_id: Mapped[Optional[str]] = mapped_column(String(255))

    # Relationships
    teacher_profile: Mapped[Optional["TeacherProfile"]] = relationship(
        back_populates="user", uselist=False, cascade="all, delete-orphan"
    )
    student_profile: Mapped[Optional["StudentProfile"]] = relationship(
        back_populates="user", uselist=False, cascade="all, delete-orphan"
    )
    wallet: Mapped[Optional["Wallet"]] = relationship(
        back_populates="user", uselist=False, cascade="all, delete-orphan"
    )
    exams_created: Mapped[List["Exam"]] = relationship(back_populates="teacher")
    exam_attempts: Mapped[List["ExamAttempt"]] = relationship(back_populates="student")
    violations: Mapped[List["Violation"]] = relationship(back_populates="student")
    notifications: Mapped[List["Notification"]] = relationship(back_populates="user")
    transactions: Mapped[List["Transaction"]] = relationship(back_populates="user")
    batches_created: Mapped[List["Batch"]] = relationship(back_populates="teacher")


class TeacherProfile(Base, UUIDMixin):
    __tablename__ = "teacher_profiles"

    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), unique=True
    )
    image_url: Mapped[Optional[str]] = mapped_column(String(500))
    subject: Mapped[Optional[str]] = mapped_column(String(100))
    focused_zones: Mapped[Optional[list]] = mapped_column(ARRAY(String))  # ["NEET", "JEE"]
    teaching_since: Mapped[Optional[int]] = mapped_column(Integer)
    degrees: Mapped[Optional[list]] = mapped_column(ARRAY(String))  # ["BSC", "B.ED"]
    locality: Mapped[Optional[str]] = mapped_column(String(255))
    city: Mapped[Optional[str]] = mapped_column(String(100))
    state: Mapped[Optional[str]] = mapped_column(String(100))

    user: Mapped["User"] = relationship(back_populates="teacher_profile")


class StudentProfile(Base, UUIDMixin):
    __tablename__ = "student_profiles"

    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), unique=True
    )
    institute_name: Mapped[Optional[str]] = mapped_column(String(255))
    focused_exam: Mapped[Optional[str]] = mapped_column(String(50))  # "NEET" | "JEE" | "SSC"

    user: Mapped["User"] = relationship(back_populates="student_profile")
```

#### Wallet Models (`app/models/wallet.py`)

```python
import uuid
import enum
from datetime import datetime
from typing import Optional, List
from decimal import Decimal
from sqlalchemy import String, Numeric, Enum, ForeignKey, Index
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.dialects.postgresql import UUID
from app.models.base import Base, UUIDMixin, TimestampMixin


class TransactionType(str, enum.Enum):
    credit = "credit"
    debit = "debit"


class CreditType(str, enum.Enum):
    paid = "paid"
    gift = "gift"
    system = "system"


class TransactionReason(str, enum.Enum):
    admin_credit = "admin_credit"
    gift = "gift"
    exam_attempt = "exam_attempt"
    publishing_fee = "publishing_fee"
    teacher_gift_to_student = "teacher_gift_to_student"


class Wallet(Base, UUIDMixin):
    __tablename__ = "wallets"

    user_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"), unique=True
    )
    balance: Mapped[Decimal] = mapped_column(Numeric(12, 4), default=Decimal("0"))
    updated_at: Mapped[datetime] = mapped_column(
        default=datetime.utcnow, onupdate=datetime.utcnow
    )

    user: Mapped["User"] = relationship(back_populates="wallet")
    transactions: Mapped[List["Transaction"]] = relationship(back_populates="wallet")


class Transaction(Base, UUIDMixin):
    __tablename__ = "transactions"

    wallet_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("wallets.id"))
    user_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("users.id"))
    transaction_type: Mapped[TransactionType] = mapped_column(Enum(TransactionType))
    credit_type: Mapped[Optional[CreditType]] = mapped_column(Enum(CreditType))
    amount: Mapped[Decimal] = mapped_column(Numeric(12, 4))
    reason: Mapped[TransactionReason] = mapped_column(Enum(TransactionReason))
    description: Mapped[Optional[str]] = mapped_column(String(500))
    reference_id: Mapped[Optional[uuid.UUID]] = mapped_column(UUID(as_uuid=True))
    balance_after: Mapped[Decimal] = mapped_column(Numeric(12, 4))
    created_by: Mapped[Optional[uuid.UUID]] = mapped_column(UUID(as_uuid=True))  # Audit: admin who issued credit
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

    wallet: Mapped["Wallet"] = relationship(back_populates="transactions")
    user: Mapped["User"] = relationship(back_populates="transactions")

    __table_args__ = (
        Index("ix_transactions_wallet_id", "wallet_id"),
        Index("ix_transactions_user_id", "user_id"),
        Index("ix_transactions_created_at", "created_at"),
    )
```

#### Exam Models (`app/models/exam.py`)

```python
import uuid
import enum
from datetime import datetime, date
from typing import Optional, List
from decimal import Decimal
from sqlalchemy import (
    String, Boolean, Integer, Text, Numeric, Date,
    Enum, ForeignKey, Index
)
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.dialects.postgresql import UUID
from app.models.base import Base, UUIDMixin, TimestampMixin


class ExamLanguage(str, enum.Enum):
    single = "single"
    bilingual = "bilingual"


class ExamVisibility(str, enum.Enum):
    private = "private"
    global_ = "global"  # 'global' is reserved in Python


class ExamCostConfig(Base, UUIDMixin):
    __tablename__ = "exam_cost_config"

    cost_per_question: Mapped[Decimal] = mapped_column(Numeric(10, 4))
    global_exam_publishing_fee: Mapped[Decimal] = mapped_column(
        Numeric(10, 4), default=Decimal("10")
    )
    updated_by: Mapped[Optional[uuid.UUID]] = mapped_column(UUID(as_uuid=True))
    updated_at: Mapped[datetime] = mapped_column(
        default=datetime.utcnow, onupdate=datetime.utcnow
    )


class Exam(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "exams"

    teacher_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id")
    )
    name: Mapped[str] = mapped_column(String(255))
    scheduled_date: Mapped[date] = mapped_column(Date)
    start_time: Mapped[str] = mapped_column(String(5))      # "HH:mm"
    end_time: Mapped[str] = mapped_column(String(5))        # "HH:mm"
    duration_minutes: Mapped[int] = mapped_column(Integer)
    language: Mapped[ExamLanguage] = mapped_column(
        Enum(ExamLanguage), default=ExamLanguage.single
    )
    visibility: Mapped[ExamVisibility] = mapped_column(
        Enum(ExamVisibility), default=ExamVisibility.private
    )
    positive_marks: Mapped[Decimal] = mapped_column(Numeric(5, 2))
    negative_marks: Mapped[Decimal] = mapped_column(Numeric(5, 2), default=Decimal("0"))
    # ⚠️ Pydantic schema (CreateExamRequest) must enforce: positive_marks > 0, negative_marks >= 0
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    is_deleted: Mapped[bool] = mapped_column(Boolean, default=False)
    is_locked: Mapped[bool] = mapped_column(Boolean, default=False)
    question_count: Mapped[int] = mapped_column(Integer, default=0)  # Denormalized — maintained by add/delete question services

    teacher: Mapped["User"] = relationship(back_populates="exams_created")
    sections: Mapped[List["Section"]] = relationship(
        back_populates="exam", cascade="all, delete-orphan"
    )
    questions: Mapped[List["Question"]] = relationship(
        back_populates="exam", cascade="all, delete-orphan"
    )
    attempts: Mapped[List["ExamAttempt"]] = relationship(back_populates="exam")
    exam_batches: Mapped[List["ExamBatch"]] = relationship(
        back_populates="exam", cascade="all, delete-orphan"
    )

    __table_args__ = (
        # Partial indexes — only index non-deleted rows (smaller, faster)
        Index("ix_exams_teacher_active", "teacher_id",
              postgresql_where=text("is_deleted = false")),
        Index("ix_exams_visibility_active", "visibility",
              postgresql_where=text("is_deleted = false AND is_active = true")),
        Index("ix_exams_scheduled_date", "scheduled_date"),
    )


class Section(Base, UUIDMixin):
    __tablename__ = "sections"

    exam_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("exams.id", ondelete="CASCADE")
    )
    name: Mapped[str] = mapped_column(String(255))

    exam: Mapped["Exam"] = relationship(back_populates="sections")
    questions: Mapped[List["Question"]] = relationship(back_populates="section")


class Question(Base, UUIDMixin):
    __tablename__ = "questions"

    exam_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("exams.id", ondelete="CASCADE")
    )
    section_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("sections.id", ondelete="CASCADE")
    )
    order_index: Mapped[int] = mapped_column(Integer, default=0)

    # English (always required)
    qn_en: Mapped[str] = mapped_column(Text)
    op_en_1: Mapped[str] = mapped_column(Text)
    op_en_2: Mapped[str] = mapped_column(Text)
    op_en_3: Mapped[str] = mapped_column(Text)
    op_en_4: Mapped[str] = mapped_column(Text)

    # Bengali (bilingual only — nullable)
    qn_bn: Mapped[Optional[str]] = mapped_column(Text)
    op_bn_1: Mapped[Optional[str]] = mapped_column(Text)
    op_bn_2: Mapped[Optional[str]] = mapped_column(Text)
    op_bn_3: Mapped[Optional[str]] = mapped_column(Text)
    op_bn_4: Mapped[Optional[str]] = mapped_column(Text)

    correct_option: Mapped[str] = mapped_column(String(1))  # "1", "2", "3", or "4"
    image_url: Mapped[Optional[str]] = mapped_column(String(500))
    # image_public_id: stored in Phase 1 so Phase 2 can delete orphaned Cloudinary images
    # without needing a new migration (satisfies the "zero migrations" Phase 2 guarantee)
    image_public_id: Mapped[Optional[str]] = mapped_column(String(500))
    solution: Mapped[Optional[str]] = mapped_column(Text)

    exam: Mapped["Exam"] = relationship(back_populates="questions")
    section: Mapped["Section"] = relationship(back_populates="questions")

    __table_args__ = (
        Index("ix_questions_exam_id", "exam_id"),
        Index("ix_questions_section_id", "section_id"),
    )
```

#### Batch Models (`app/models/batch.py`)

```python
import uuid
import enum
from datetime import datetime
from typing import Optional, List
from sqlalchemy import String, Boolean, Enum, ForeignKey, UniqueConstraint, Index
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.dialects.postgresql import UUID
from app.models.base import Base, UUIDMixin, TimestampMixin


class TeacherStudentStatus(str, enum.Enum):
    pending = "pending"
    active = "active"


class Batch(Base, UUIDMixin):
    __tablename__ = "batches"

    teacher_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id")
    )
    name: Mapped[str] = mapped_column(String(255))
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

    teacher: Mapped["User"] = relationship(back_populates="batches_created")
    students: Mapped[List["BatchStudent"]] = relationship(
        back_populates="batch", cascade="all, delete-orphan"
    )
    exam_batches: Mapped[List["ExamBatch"]] = relationship(
        back_populates="batch", cascade="all, delete-orphan"
    )

    __table_args__ = (Index("ix_batches_teacher_id", "teacher_id"),)


class TeacherStudent(Base, UUIDMixin):
    __tablename__ = "teacher_students"

    teacher_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id")
    )
    student_id: Mapped[Optional[uuid.UUID]] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id")
    )
    invited_email: Mapped[str] = mapped_column(String(255))
    status: Mapped[TeacherStudentStatus] = mapped_column(
        Enum(TeacherStudentStatus), default=TeacherStudentStatus.pending
    )
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    linked_at: Mapped[Optional[datetime]] = mapped_column()
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

    __table_args__ = (
        UniqueConstraint("teacher_id", "invited_email", name="uq_teacher_invited_email"),
        # Prevent duplicate (teacher_id, student_id) pairs after a student registers.
        # The partial unique constraint only applies to non-null student_id rows.
        # Using a conditional unique index at DB level handles this cleanly.
        # For SQLAlchemy, we add it here and rely on the migration to create it:
        # CREATE UNIQUE INDEX uq_teacher_student ON teacher_students(teacher_id, student_id)
        #     WHERE student_id IS NOT NULL;
        Index(
            "uq_teacher_student",
            "teacher_id", "student_id",
            unique=True,
            postgresql_where="student_id IS NOT NULL",  # Partial index: nulls allowed
        ),
        Index("ix_teacher_students_invited_email", "invited_email"),
    )


class BatchStudent(Base, UUIDMixin):
    __tablename__ = "batch_students"

    batch_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("batches.id", ondelete="CASCADE")
    )
    student_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id")
    )

    batch: Mapped["Batch"] = relationship(back_populates="students")

    __table_args__ = (
        UniqueConstraint("batch_id", "student_id", name="uq_batch_student"),
    )


class ExamBatch(Base, UUIDMixin):
    __tablename__ = "exam_batches"

    exam_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("exams.id", ondelete="CASCADE")
    )
    batch_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("batches.id", ondelete="CASCADE")
    )

    exam: Mapped["Exam"] = relationship(back_populates="exam_batches")
    batch: Mapped["Batch"] = relationship(back_populates="exam_batches")

    __table_args__ = (
        UniqueConstraint("exam_id", "batch_id", name="uq_exam_batch"),
    )
```

#### Attempt Models (`app/models/attempt.py`)

```python
import uuid
from datetime import datetime
from typing import Optional, List
from decimal import Decimal
from sqlalchemy import String, Boolean, Integer, Numeric, ForeignKey, UniqueConstraint, Index
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.dialects.postgresql import UUID, JSONB
from app.models.base import Base, UUIDMixin


class ExamAttempt(Base, UUIDMixin):
    __tablename__ = "exam_attempts"

    exam_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("exams.id")
    )
    student_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id")
    )
    answers: Mapped[Optional[dict]] = mapped_column(JSONB)  # {questionId: selectedOption}
    score: Mapped[Optional[Decimal]] = mapped_column(Numeric(8, 2))
    total_marks: Mapped[Optional[Decimal]] = mapped_column(Numeric(8, 2))
    time_taken_seconds: Mapped[Optional[int]] = mapped_column(Integer)
    coin_deducted: Mapped[Decimal] = mapped_column(Numeric(12, 4))
    cost_per_question_snapshot: Mapped[Decimal] = mapped_column(Numeric(10, 4))
    wallet_deducted_from: Mapped[str] = mapped_column(String(10))  # "teacher" | "student"
    attempted_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    submitted_at: Mapped[Optional[datetime]] = mapped_column()
    is_submitted: Mapped[bool] = mapped_column(Boolean, default=False)

    exam: Mapped["Exam"] = relationship(back_populates="attempts")
    student: Mapped["User"] = relationship(back_populates="exam_attempts")
    violations: Mapped[List["Violation"]] = relationship(
        back_populates="attempt", cascade="all, delete-orphan"
    )

    __table_args__ = (
        UniqueConstraint("exam_id", "student_id", name="uq_exam_student_attempt"),
        Index("ix_attempts_student_id", "student_id"),
        Index("ix_attempts_exam_id", "exam_id"),
    )


class Violation(Base, UUIDMixin):
    __tablename__ = "violations"

    attempt_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("exam_attempts.id", ondelete="CASCADE")
    )
    student_id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True), ForeignKey("users.id")
    )
    type: Mapped[str] = mapped_column(String(50), default="fullscreen_exit")
    recorded_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

    attempt: Mapped["ExamAttempt"] = relationship(back_populates="violations")
    student: Mapped["User"] = relationship(back_populates="violations")

    __table_args__ = (Index("ix_violations_attempt_id", "attempt_id"),)
```

#### Auth & Notification Models (`app/models/auth.py`, `app/models/notification.py`)

```python
# app/models/auth.py
import uuid
from datetime import datetime
from sqlalchemy import String, Boolean, DateTime, Index
from sqlalchemy.orm import Mapped, mapped_column
from sqlalchemy.dialects.postgresql import UUID
from app.models.base import Base, UUIDMixin


class Otp(Base, UUIDMixin):
    __tablename__ = "otps"

    email: Mapped[str] = mapped_column(String(255))
    code: Mapped[str] = mapped_column(String(6))
    purpose: Mapped[str] = mapped_column(String(50))  # registration | forgot_password | change_password
    expires_at: Mapped[datetime] = mapped_column(DateTime(timezone=True))
    is_used: Mapped[bool] = mapped_column(Boolean, default=False)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

    __table_args__ = (Index("ix_otps_email_purpose", "email", "purpose"),)


class RefreshToken(Base, UUIDMixin):
    __tablename__ = "refresh_tokens"

    user_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True))
    token: Mapped[str] = mapped_column(String(500), unique=True)
    expires_at: Mapped[datetime] = mapped_column(DateTime(timezone=True))
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

    __table_args__ = (Index("ix_refresh_tokens_user_id", "user_id"),)
```

```python
# app/models/notification.py
import uuid
import enum
from datetime import datetime
from typing import Optional
from sqlalchemy import String, Boolean, Text, Enum, ForeignKey, Index
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.dialects.postgresql import UUID, JSONB
from app.models.base import Base, UUIDMixin


class NotificationType(str, enum.Enum):
    low_balance = "low_balance"
    exam_blocked = "exam_blocked"
    system = "system"


class Notification(Base, UUIDMixin):
    __tablename__ = "notifications"

    user_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("users.id"))
    type: Mapped[NotificationType] = mapped_column(Enum(NotificationType))
    title: Mapped[str] = mapped_column(String(255))
    message: Mapped[str] = mapped_column(Text)
    is_read: Mapped[bool] = mapped_column(Boolean, default=False)
    metadata_: Mapped[Optional[dict]] = mapped_column("metadata", JSONB)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)

    user: Mapped["User"] = relationship(back_populates="notifications")

    __table_args__ = (Index("ix_notifications_user_read", "user_id", "is_read"),)
```

### 4.2 Alembic Migration Strategy

```bash
# Initialize Alembic (one-time)
cd backend
alembic init alembic

# After editing models:
alembic revision --autogenerate -m "description_of_change"

# Apply migrations (only runs NEW migrations — checks alembic_version stamp)
alembic upgrade head

# Check current stamp in DB
alembic current

# View pending migrations (current vs head)
alembic heads

# Rollback one step
alembic downgrade -1

# View migration history
alembic history
```

> [!IMPORTANT]
> **How Alembic stamping works:** Alembic maintains a single-row `alembic_version` table in PostgreSQL that records the current migration revision hash. When `alembic upgrade head` runs, it compares the DB stamp against the latest revision in the `alembic/versions/` directory. If they match, **nothing happens** — no migrations are executed. Only new, unapplied revisions are run. This makes `alembic upgrade head` **idempotent and safe to call on every deploy**.

**`alembic/env.py` (key section):**
```python
from app.core.config import settings
from app.models.base import Base
# Import ALL models so Alembic auto-detects them
from app.models import user, wallet, exam, batch, attempt, auth, notification

config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)
target_metadata = Base.metadata
```

**Phase 1 Migrations (in order):**
1. `001_init_core_tables` — users, teacher_profiles, student_profiles, wallets, transactions
2. `002_init_auth_tables` — otps, refresh_tokens
3. `003_init_exam_tables` — exam_cost_config, exams, sections, questions
4. `004_init_batch_tables` — batches, teacher_students, batch_students, exam_batches
5. `005_init_attempt_tables_and_seed` — exam_attempts, violations, notifications + **seed ExamCostConfig**

**Seed data via Alembic data migration (inside `005_init_attempt_tables_and_seed`):**

> [!IMPORTANT]
> **No separate seed script.** Seed data is embedded in the final Alembic migration as a data migration. This ensures seeding runs **exactly once** (when the migration is first applied) and is tracked by the Alembic version stamp. Re-running `alembic upgrade head` on a stamped DB will skip this entirely.

```python
# alembic/versions/005_init_attempt_tables_and_seed.py

from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import UUID
import uuid
from decimal import Decimal

revision = "005"
down_revision = "004"

SEED_CONFIG_ID = uuid.UUID("00000000-0000-0000-0000-000000000001")

def upgrade():
    # --- Schema: create tables ---
    op.create_table(
        "exam_attempts",
        # ... (columns defined by autogenerate)
    )
    op.create_table("violations", ...)
    op.create_table("notifications", ...)

    # --- Data: seed ExamCostConfig ---
    # This runs exactly once when this migration is applied.
    # Alembic stamps the DB after this — subsequent `alembic upgrade head` calls skip it.
    exam_cost_config = sa.table(
        "exam_cost_config",
        sa.column("id", UUID(as_uuid=True)),
        sa.column("cost_per_question", sa.Numeric(10, 4)),
        sa.column("global_exam_publishing_fee", sa.Numeric(10, 4)),
    )
    op.execute(
        exam_cost_config.insert().values(
            id=SEED_CONFIG_ID,
            cost_per_question=Decimal("0.15"),
            global_exam_publishing_fee=Decimal("10"),
        )
    )
    print("✅ Exam cost config seeded (₹0.15/question, 10 coin publishing fee)")

def downgrade():
    # Remove seed data first, then drop tables
    op.execute(f"DELETE FROM exam_cost_config WHERE id = '{SEED_CONFIG_ID}'")
    op.drop_table("notifications")
    op.drop_table("violations")
    op.drop_table("exam_attempts")
```

**Smart startup script (`backend/scripts/startup.py`):**

> Used instead of raw `alembic upgrade head` in Docker CMD. Checks the stamp first.

```python
# backend/scripts/startup.py
import asyncio
import subprocess
import sys
import uuid

from app.core.config import settings
from app.core.security import hash_password
from app.core.database import async_session_factory
from app.models.user import User, Role
from app.models.wallet import Wallet
from sqlalchemy import select
from decimal import Decimal


def get_current_revision() -> str | None:
    """Get the current Alembic stamp from the database."""
    result = subprocess.run(
        ["alembic", "current"],
        capture_output=True, text=True
    )
    output = result.stdout.strip()
    if not output or "(head)" not in output:
        return None
    return output.split()[0]


def get_head_revision() -> str:
    """Get the latest migration revision from the alembic/versions/ directory."""
    result = subprocess.run(
        ["alembic", "heads"],
        capture_output=True, text=True
    )
    return result.stdout.strip().split()[0]


def run_migrations():
    """Check stamp and run migrations only if needed."""
    current = get_current_revision()
    head = get_head_revision()

    if current is None:
        print("🔄 No Alembic stamp found — running all migrations (fresh DB)...")
        subprocess.run(["alembic", "upgrade", "head"], check=True)
        print("✅ All migrations applied (including seed data).")
    elif current != head.replace(" (head)", ""):
        print(f"🔄 DB at revision {current}, head is {head} — applying new migrations...")
        subprocess.run(["alembic", "upgrade", "head"], check=True)
        print("✅ New migrations applied.")
    else:
        print(f"✅ DB already at head ({current}). No migrations needed.")


async def ensure_admin_user():
    """
    Upsert the Super Admin user from .env on EVERY startup.

    - Reads ADMIN_EMAIL and ADMIN_PASSWORD from environment.
    - If no admin user exists → creates one with role=super_admin + wallet.
    - If admin user exists → overwrites email and password hash.

    This ensures that if you update .env credentials, the DB record
    is always in sync after the next restart. The admin gets a real
    UUID, a real User record, a real Wallet, and single-device
    session enforcement — identical to teacher/student auth.
    """
    async with async_session_factory() as session:
        async with session.begin():
            # Find existing admin user (by role, not email — email can change)
            result = await session.execute(
                select(User).where(User.role == Role.super_admin)
            )
            admin = result.scalar_one_or_none()

            if admin is None:
                # First boot — create admin user + wallet
                admin = User(
                    email=settings.ADMIN_EMAIL,
                    password=hash_password(settings.ADMIN_PASSWORD),
                    name="Super Admin",
                    phone="",
                    role=Role.super_admin,
                    is_active=True,
                    is_verified=True,
                )
                session.add(admin)
                await session.flush()  # Get admin.id

                wallet = Wallet(
                    user_id=admin.id,
                    balance=Decimal("0"),
                )
                session.add(wallet)
                print(f"✅ Super Admin created in DB (email: {settings.ADMIN_EMAIL})")
            else:
                # Existing admin — overwrite creds from .env
                admin.email = settings.ADMIN_EMAIL
                admin.password = hash_password(settings.ADMIN_PASSWORD)
                print(f"🔄 Super Admin credentials synced from .env (email: {settings.ADMIN_EMAIL})")


if __name__ == "__main__":
    # 1. Run pending migrations (stamp-aware)
    run_migrations()

    # 2. Upsert admin user from .env (runs EVERY startup)
    asyncio.run(ensure_admin_user())
```

---

## 5. API Documentation (REST)

**Base URL:** `https://api.domain.com/api/v1`

> [!TIP]
> FastAPI auto-generates interactive API docs. After starting the server:
> - **Swagger UI:** `http://localhost:8000/docs`
> - **ReDoc:** `http://localhost:8000/redoc`
> - **OpenAPI JSON:** `http://localhost:8000/openapi.json`

### Standardized Response Format

```python
# app/schemas/common.py
from pydantic import BaseModel
from typing import Any, Optional, List

class ApiResponse(BaseModel):
    success: bool
    message: str
    data: Optional[Any] = None
    meta: Optional[dict] = None

class ErrorResponse(BaseModel):
    success: bool = False
    message: str
    errors: Optional[List[dict]] = None

class PaginationParams(BaseModel):
    page: int = 1
    limit: int = 20
```

### 5.1 Auth Routes (`/api/v1/auth`)

| Method | Endpoint | Body | Auth | Description |
|---|---|---|---|---|
| `POST` | `/auth/admin/login` | `{ email, password }` | ❌ | Super Admin login (validates against env) |
| `POST` | `/auth/teacher/login` | `{ email, password }` | ❌ | Teacher login |
| `POST` | `/auth/student/login` | `{ email, password }` | ❌ | Student login |
| `POST` | `/auth/student/register` | `{ name, phone, institute_name, focused_exam, email, password }` | ❌ | Student registration (sends OTP) |
| `POST` | `/auth/verify-otp` | `{ email, code, purpose }` | ❌ | Verify OTP |
| `POST` | `/auth/forgot-password` | `{ email, role }` | ❌ | Send OTP for password reset |
| `POST` | `/auth/reset-password` | `{ email, code, new_password }` | ❌ | Reset password after OTP |
| `POST` | `/auth/change-password/initiate` | `{ }` | ✅ | Step 1: Send OTP to logged-in user's email (for change-password flow) |
| `POST` | `/auth/change-password/confirm` | `{ code, new_password }` | ✅ | Step 2: Verify OTP + set new password |
| `POST` | `/auth/refresh-token` | `{ refresh_token }` | ❌ | Get new access token |
| `POST` | `/auth/logout` | `{}` | ✅ | Invalidate session |

> [!IMPORTANT]
> **Change-password is a 2-step OTP flow** (not single-step with `current_password`).
> This is the authoritative decision — it aligns with `product_feature.md § 5.1` which states OTP verification is required. The `change_password` purpose value in the `Otp` model is actively used.
> 1. `POST /auth/change-password/initiate` — authenticated, sends OTP to user's email, purpose=`"change_password"`
> 2. `POST /auth/change-password/confirm` — authenticated, verifies OTP, sets new password

**Pydantic schemas:**
```python
# app/schemas/auth.py
from pydantic import BaseModel, EmailStr, Field

class LoginRequest(BaseModel):
    email: EmailStr
    password: str = Field(min_length=6, max_length=128)

class RegisterStudentRequest(BaseModel):
    name: str = Field(min_length=2, max_length=100)
    phone: str = Field(min_length=10, max_length=15)
    institute_name: str = Field(min_length=2, max_length=200)
    focused_exam: str  # "NEET" | "JEE" | "SSC"
    email: EmailStr
    password: str = Field(min_length=8, max_length=128)

class VerifyOtpRequest(BaseModel):
    email: EmailStr
    code: str = Field(min_length=6, max_length=6)
    purpose: str  # "registration" | "forgot_password" | "change_password"

class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int = 900

class UserResponse(BaseModel):
    id: str
    email: str
    name: str
    role: str
    is_active: bool

    class Config:
        from_attributes = True

class LoginResponse(BaseModel):
    user: UserResponse
    tokens: TokenResponse
```

### 5.2 Admin Routes (`/api/v1/admin`)
> All routes require `get_current_user` + `require_role("super_admin")`

| Method | Endpoint | Body / Params | Description |
|---|---|---|---|
| `GET` | `/admin/dashboard` | — | Dashboard metrics |
| `POST` | `/admin/teachers` | Full teacher profile body | Create teacher account |
| `GET` | `/admin/teachers` | `?page, limit, search, status` | List all teachers |
| `GET` | `/admin/teachers/{id}` | — | Get single teacher |
| `PATCH` | `/admin/teachers/{id}` | Partial update | Update teacher |
| `PATCH` | `/admin/teachers/{id}/status` | `{ is_active }` | Toggle status |
| `DELETE` | `/admin/teachers/{id}` | — | Soft delete |
| `PATCH` | `/admin/teachers/{id}/reset-password` | `{ new_password }` | Admin resets password |
| `GET` | `/admin/students` | `?page, limit, search, status` | List all students |
| `PATCH` | `/admin/students/{id}/status` | `{ is_active }` | Toggle status |
| `DELETE` | `/admin/students/{id}` | — | Soft delete |
| `POST` | `/admin/wallet/credit` | `{ user_id, amount, credit_type, description }` | Credit coins |
| `GET` | `/admin/config/exam-cost` | — | Get exam cost config |
| `PATCH` | `/admin/config/exam-cost` | `{ cost_per_question, global_exam_publishing_fee }` | Update config |
| `GET` | `/admin/transactions` | `?page, limit, type, date_from, date_to` | All transactions |
| `GET` | `/admin/revenue` | `?date_from, date_to` | Revenue summary |

### 5.3 Teacher Routes (`/api/v1/teacher`)
> All routes require `get_current_user` + `require_role("teacher")`

| Method | Endpoint | Body / Params | Description |
|---|---|---|---|
| `GET` | `/teacher/dashboard` | — | Dashboard metrics |
| `GET` | `/teacher/profile` | — | Own profile |
| `PATCH` | `/teacher/profile` | Partial update | Update profile |
| **Student Management** | | | |
| `GET` | `/teacher/students` | `?page, limit, search, status` | Linked students |
| `POST` | `/teacher/students/search` | `{ email }` | Search by email |
| `POST` | `/teacher/students/link` | `{ email }` | Link/invite student |
| `PATCH` | `/teacher/students/{link_id}/status` | `{ is_active }` | Toggle link status |
| `DELETE` | `/teacher/students/{link_id}` | — | Remove link |
| `POST` | `/teacher/students/{student_id}/gift` | `{ amount }` | Gift coins |
| `GET` | `/teacher/students/{student_id}/profile` | — | Student performance |
| `GET` | `/teacher/students/{student_id}/attempts` | `?page, limit` | Student's history |
| `GET` | `/teacher/students/{student_id}/violations` | — | Violation records |
| **Batch Management** | | | |
| `POST` | `/teacher/batches` | `{ name }` | Create batch |
| `GET` | `/teacher/batches` | — | List batches |
| `GET` | `/teacher/batches/{id}` | — | Batch detail |
| `PATCH` | `/teacher/batches/{id}` | `{ name }` | Rename |
| `DELETE` | `/teacher/batches/{id}` | — | Delete batch |
| `POST` | `/teacher/batches/{id}/students` | `{ student_ids: [] }` | Add students |
| `DELETE` | `/teacher/batches/{id}/students/{student_id}` | — | Remove student |
| **Exam Management** | | | |
| `POST` | `/teacher/exams` | Full exam body | Create exam |
| `GET` | `/teacher/exams` | `?page, limit, visibility, status` | List exams |
| `GET` | `/teacher/exams/{id}` | — | Exam detail |
| `PATCH` | `/teacher/exams/{id}` | Partial update | Update settings |
| `PATCH` | `/teacher/exams/{id}/status` | `{ is_active }` | Toggle status |
| `DELETE` | `/teacher/exams/{id}` | — | Soft delete |
| `POST` | `/teacher/exams/{id}/batches` | `{ batch_ids: [] }` | Assign batches |
| `DELETE` | `/teacher/exams/{id}/batches/{batch_id}` | — | Remove batch |
| **Sections & Questions** | | | |
| `POST` | `/teacher/exams/{exam_id}/sections` | `{ name }` | Add section |
| `PATCH` | `/teacher/exams/{exam_id}/sections/{id}` | `{ name }` | Rename |
| `DELETE` | `/teacher/exams/{exam_id}/sections/{id}` | — | Delete section |
| `POST` | `/teacher/exams/{exam_id}/questions` | Full question body | Add question |
| `GET` | `/teacher/exams/{exam_id}/questions` | — | List questions |
| `PATCH` | `/teacher/exams/{exam_id}/questions/{id}` | Partial update | Edit (if not locked) |
| `DELETE` | `/teacher/exams/{exam_id}/questions/{id}` | — | Delete (if not locked) |
| **Analytics & Wallet** | | | |
| `GET` | `/teacher/exams/{id}/analytics` | — | Exam analytics |
| `GET` | `/teacher/exams/{id}/attempts` | `?page, limit` | All attempts |
| `GET` | `/teacher/wallet` | — | Wallet balance |
| `GET` | `/teacher/wallet/transactions` | `?page, limit, type` | History |

> [!WARNING]
> **Bilingual Validation on Exam Update:** When `PATCH /teacher/exams/{id}` changes the `language` field from `single` to `bilingual`, the service layer must validate that **all existing questions** have non-null Bengali fields (`qn_bn`, `op_bn_1` through `op_bn_4`). If any question is missing Bengali content, reject the update with `400: "All questions must have Bengali translations before switching to bilingual mode."` This prevents empty text rendering on the student exam UI.

### 5.4 Student Routes (`/api/v1/student`)
> All routes require `get_current_user` + `require_role("student")`

| Method | Endpoint | Body / Params | Description |
|---|---|---|---|
| `GET` | `/student/profile` | — | Own profile |
| `PATCH` | `/student/profile` | Partial update | Update profile |
| `GET` | `/student/exams/my` | `?category` | My assigned exams |
| `GET` | `/student/exams/global` | `?category, page, limit` | All global exams |
| `GET` | `/student/exams/{id}` | — | Exam metadata |
| `POST` | `/student/exams/{id}/start` | `{}` | Start exam (cost deduction + questions) |
| `POST` | `/student/exams/{id}/submit` | `{ answers, time_taken_seconds }` | Submit answers |
| `GET` | `/student/exams/{id}/result` | — | Result + solutions |
| `POST` | `/student/exams/{id}/violations` | `{ type }` | Record violation |
| `GET` | `/student/results` | `?page, limit` | All my attempts |
| `GET` | `/student/wallet` | — | Wallet balance |
| `GET` | `/student/wallet/transactions` | `?page, limit` | History |
| `GET` | `/student/notifications` | `?page, limit` | Notifications |
| `PATCH` | `/student/notifications/{id}/read` | `{}` | Mark read |

### 5.5 Start Exam — Critical Flow

**`POST /api/v1/student/exams/{exam_id}/start`**

```python
# app/services/student_service.py

async def start_exam(
    student_id: uuid.UUID,
    exam_id: uuid.UUID,
    db: AsyncSession
) -> dict:
    """
    Critical atomic operation:
    1. Validate exam eligibility
    2. Check wallet balance
    3. Deduct coins
    4. Create attempt record
    5. Lock exam (if first attempt)
    6. Return questions (without answers)
    """
    async with db.begin():  # Atomic transaction

        # 1. Fetch exam with question count
        exam = await db.execute(
            select(Exam)
            .options(selectinload(Exam.questions))
            .where(Exam.id == exam_id, Exam.is_deleted == False)
            .with_for_update()  # Row-level lock
        )
        exam = exam.scalar_one_or_none()
        if not exam:
            raise HTTPException(404, "Exam not found")

        if not exam.is_active:
            raise HTTPException(400, "Exam is not active")

        # Use denormalized count for cost calculation (questions are still
        # loaded via selectinload for the response payload)
        if exam.question_count == 0:
            raise HTTPException(400, "Exam has no questions")

        # 2. Check if already attempted
        existing = await db.execute(
            select(ExamAttempt).where(
                ExamAttempt.exam_id == exam_id,
                ExamAttempt.student_id == student_id
            )
        )
        if existing.scalar_one_or_none():
            raise HTTPException(409, "You have already attempted this exam")

        # 3. Get current cost config
        config = await db.execute(select(ExamCostConfig))
        config = config.scalar_one()
        total_cost = config.cost_per_question * exam.question_count

        # 4. Determine which wallet to charge
        if exam.visibility == ExamVisibility.private:
            wallet_user_id = exam.teacher_id
            wallet_deducted_from = "teacher"

            # Verify student has batch access
            has_access = await db.execute(
                select(ExamBatch)
                .join(BatchStudent, BatchStudent.batch_id == ExamBatch.batch_id)
                .where(
                    ExamBatch.exam_id == exam_id,
                    BatchStudent.student_id == student_id
                )
            )
            if not has_access.first():
                raise HTTPException(403, "You are not assigned to this exam")
        else:
            wallet_user_id = student_id
            wallet_deducted_from = "student"

        # 5. Check & deduct wallet (atomic, with row lock)
        wallet = await db.execute(
            select(Wallet)
            .where(Wallet.user_id == wallet_user_id)
            .with_for_update()
        )
        wallet = wallet.scalar_one()

        if wallet.balance < total_cost:
            if wallet_deducted_from == "teacher":
                # Notify teacher
                notification = Notification(
                    user_id=exam.teacher_id,
                    type=NotificationType.low_balance,
                    title="Insufficient Balance",
                    message=f'Student blocked from "{exam.name}". '
                            f'Balance: {wallet.balance}, Required: {total_cost}',
                    metadata_={"exam_id": str(exam_id), "student_id": str(student_id)},
                )
                db.add(notification)
                raise HTTPException(
                    402,
                    "Exam temporarily unavailable. Your teacher has been notified."
                )
            else:
                raise HTTPException(
                    402,
                    "Insufficient balance. Please contact admin or your teacher."
                )

        # 6. Deduct coins
        new_balance = wallet.balance - total_cost
        wallet.balance = new_balance

        # 7. Record transaction
        transaction = Transaction(
            wallet_id=wallet.id,
            user_id=wallet_user_id,
            transaction_type=TransactionType.debit,
            amount=total_cost,
            reason=TransactionReason.exam_attempt,
            description=f"Exam: {exam.name}",
            reference_id=exam_id,
            balance_after=new_balance,
        )
        db.add(transaction)

        # 8. Create attempt record
        attempt = ExamAttempt(
            exam_id=exam_id,
            student_id=student_id,
            coin_deducted=total_cost,
            cost_per_question_snapshot=config.cost_per_question,
            wallet_deducted_from=wallet_deducted_from,
        )
        db.add(attempt)

        # 9. Lock exam if first attempt
        if not exam.is_locked:
            exam.is_locked = True

        try:
            await db.flush()  # Get attempt.id
        except IntegrityError:  # from sqlalchemy.exc import IntegrityError
            # Defense-in-depth: UniqueConstraint(exam_id, student_id) caught
            # The with_for_update() lock above prevents this in normal operation,
            # but this handles edge cases gracefully instead of returning 500.
            raise HTTPException(409, "You have already attempted this exam")

        # 10. Prepare questions (EXCLUDE correct answers)
        questions_out = [
            {
                "id": str(q.id),
                "qn_en": q.qn_en,
                "op_en_1": q.op_en_1, "op_en_2": q.op_en_2,
                "op_en_3": q.op_en_3, "op_en_4": q.op_en_4,
                "qn_bn": q.qn_bn,
                "op_bn_1": q.op_bn_1, "op_bn_2": q.op_bn_2,
                "op_bn_3": q.op_bn_3, "op_bn_4": q.op_bn_4,
                "image_url": q.image_url,
                "section_id": str(q.section_id),
                "order_index": q.order_index,
                # NOTE: correct_option and solution EXCLUDED
            }
            for q in sorted(exam.questions, key=lambda q: q.order_index)
        ]

        sections = await db.execute(
            select(Section).where(Section.exam_id == exam_id)
        )

        return {
            "attempt_id": str(attempt.id),
            "exam": {
                "name": exam.name,
                "duration_minutes": exam.duration_minutes,
                "language": exam.language.value,
                "positive_marks": float(exam.positive_marks),
                "negative_marks": float(exam.negative_marks),
            },
            "sections": [{"id": str(s.id), "name": s.name} for s in sections.scalars()],
            "questions": questions_out,
            "coin_deducted": float(total_cost),
        }
```

---

## 6. Authentication & Authorization

### 6.1 Security Module (`app/core/security.py`)

```python
from datetime import datetime, timedelta
from typing import Optional
import uuid
from jose import jwt, JWTError
from passlib.context import CryptContext
from app.core.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)


def create_access_token(user_id: str, email: str, role: str, session_id: str) -> str:
    payload = {
        "sub": user_id,
        "email": email,
        "role": role,
        "session_id": session_id,
        "exp": datetime.utcnow() + timedelta(minutes=15),
        "iat": datetime.utcnow(),
    }
    return jwt.encode(payload, settings.JWT_ACCESS_SECRET, algorithm="HS256")


def create_refresh_token(user_id: str) -> str:
    payload = {
        "sub": user_id,
        "exp": datetime.utcnow() + timedelta(days=7),
        "iat": datetime.utcnow(),
        "jti": str(uuid.uuid4()),
    }
    return jwt.encode(payload, settings.JWT_REFRESH_SECRET, algorithm="HS256")


def decode_token(token: str, secret: str) -> dict:
    return jwt.decode(token, secret, algorithms=["HS256"])
```

### 6.2 FastAPI Dependencies (`app/core/dependencies.py`)

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.ext.asyncio import AsyncSession
from jose import JWTError
from app.core.database import get_db
from app.core.security import decode_token
from app.core.config import settings
from app.models.user import User
from sqlalchemy import select

security = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db),
) -> User:
    """Verify JWT and return the authenticated user."""
    try:
        payload = decode_token(credentials.credentials, settings.JWT_ACCESS_SECRET)
    except JWTError:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid or expired token")

    user_id = payload.get("sub")
    session_id = payload.get("session_id")

    result = await db.execute(select(User).where(User.id == user_id))
    user = result.scalar_one_or_none()

    if not user:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "User not found")
    if user.is_deleted:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Account deleted")
    if not user.is_active:
        raise HTTPException(status.HTTP_403_FORBIDDEN, "Account deactivated")
    if user.active_session_id != session_id:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Session expired — logged in elsewhere")

    return user


def require_role(*roles: str):
    """Factory that returns a dependency checking user role."""
    async def role_checker(current_user: User = Depends(get_current_user)) -> User:
        if current_user.role.value not in roles:
            raise HTTPException(status.HTTP_403_FORBIDDEN, "Insufficient permissions")
        return current_user
    return role_checker
```

**Usage in routers:**
```python
# app/api/admin.py
from fastapi import APIRouter, Depends
from app.core.dependencies import require_role

router = APIRouter(prefix="/admin", tags=["Admin"])

@router.get("/dashboard")
async def get_dashboard(
    current_user=Depends(require_role("super_admin")),
    db: AsyncSession = Depends(get_db),
):
    ...
```

### 6.3 Super Admin Auth

- Admin credentials are **read from `.env`** (`ADMIN_EMAIL`, `ADMIN_PASSWORD`) and **upserted into the `users` table** on every application startup via `scripts/startup.py`.
- The admin gets a **real `User` record** with `role: "super_admin"`, a **real UUID**, and a **real `Wallet`**.
- Admin login does a **standard DB lookup** — identical to teacher/student login (password hash verification, `active_session_id` session management).
- If `.env` credentials are updated, the next restart **overwrites** the DB record's email and password hash. No manual DB intervention needed.
- This eliminates the sentinel UUID workaround — admin `created_by` in Transaction records now references a real user ID.

### 6.4 Session Management (Single Device)

Same as before — `active_session_id` in user record vs JWT. New login overwrites session ID.

### 6.5 OTP Flow

Same logic — `Otp` model with 10-min expiry. `fastapi-mail` sends emails.

---

## 7. Component Architecture (Frontend)

> Frontend is **identical** to the previous version — React + Vite + Tailwind. No changes from the Express version. See § 7 in the previous PRD for full routing structure and component specs.

The routing structure, Zustand stores, hooks (`useExamTimer`, `useFullscreen`), and all component specifications remain exactly the same since the frontend only communicates via REST API.

---

## 8. Wallet & Subscription Logic Gates

> Logic is identical. Implementation is now in Python/SQLAlchemy instead of JavaScript/Prisma. See § 5.5 for the complete `start_exam` implementation. All wallet operations use SQLAlchemy's `async with db.begin()` for atomic transactions with `with_for_update()` row-level locks.

---

## 9. SEO & Meta Management

> Identical to previous version — `react-helmet-async` for meta tags. No backend changes.

---

## 10. Security Architecture

| Measure | Implementation |
|---|---|
| Password hashing | **passlib[bcrypt]** with auto-rounds |
| JWT signing | HS256 with strong secret (min 64 chars) |
| Token expiry | Access: 15 min, Refresh: 7 days |
| Token rotation | Old refresh token deleted on use |
| Session binding | `active_session_id` in user record vs JWT |
| OTP expiry | 10 minutes, single-use, rate-limited |
| Rate limiting | **SlowAPI** (starlette-based) — per-route config |
| CORS | FastAPI `CORSMiddleware` — whitelist frontend domain |
| Trusted host | `TrustedHostMiddleware` — prevent host header attacks |
| Input validation | **Pydantic v2** — automatic on every route |
| SQL injection | **SQLAlchemy** (parameterized queries) |
| Request size | `app.add_middleware(RequestSizeLimitMiddleware, max_size=10*1024*1024)` |
| Exam security | Correct answers excluded, server-side scoring, DB unique constraint for one attempt |

---

## 11. Environment Variables

### Backend `.env`

```bash
# ─── Server ───
ENV=development
PORT=8000

# ─── Database ───
DATABASE_URL=postgresql+asyncpg://user:password@host:5432/cbt_platform
DATABASE_URL_SYNC=postgresql://user:password@host:5432/cbt_platform  # For Alembic

# ─── JWT ───
JWT_ACCESS_SECRET=your-64-char-access-secret-here
JWT_REFRESH_SECRET=your-64-char-refresh-secret-here

# ─── Super Admin ───
ADMIN_EMAIL=admin@devophile.com
ADMIN_PASSWORD=super-secure-admin-password

# ─── Email (SMTP) ───
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=noreply@devophile.com
SMTP_PASS=app-password-here
SMTP_FROM=noreply@devophile.com
SMTP_FROM_NAME=Devophile CBT

# ─── Cloudinary (Phase 2) ───
CLOUDINARY_CLOUD_NAME=
CLOUDINARY_API_KEY=
CLOUDINARY_API_SECRET=

# ─── Frontend URL (CORS) ───
FRONTEND_URL=http://localhost:5173

# ─── Redis (Phase 3) ───
REDIS_URL=redis://localhost:6379
```

**Pydantic Settings:**
```python
# app/core/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    ENV: str = "development"
    PORT: int = 8000

    DATABASE_URL: str
    DATABASE_URL_SYNC: str = ""  # For Alembic

    JWT_ACCESS_SECRET: str
    JWT_REFRESH_SECRET: str

    ADMIN_EMAIL: str
    ADMIN_PASSWORD: str

    SMTP_HOST: str = "smtp.gmail.com"
    SMTP_PORT: int = 587
    SMTP_USER: str = ""
    SMTP_PASS: str = ""
    SMTP_FROM: str = ""
    SMTP_FROM_NAME: str = "Devophile CBT"

    FRONTEND_URL: str = "http://localhost:5173"

    CLOUDINARY_CLOUD_NAME: str = ""
    CLOUDINARY_API_KEY: str = ""
    CLOUDINARY_API_SECRET: str = ""

    REDIS_URL: str = ""

    class Config:
        env_file = ".env"

settings = Settings()
```

### Frontend `.env`

```bash
VITE_API_BASE_URL=http://localhost:8000/api/v1
VITE_APP_NAME=Devophile CBT
```

---

## 12. Docker & DevOps

### `docker-compose.yml` (Local Development)

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: cbt_postgres
    environment:
      POSTGRES_DB: cbt_platform
      POSTGRES_USER: cbt_user
      POSTGRES_PASSWORD: cbt_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U cbt_user -d cbt_platform"]
      interval: 5s
      timeout: 5s
      retries: 5

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: cbt_backend
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql+asyncpg://cbt_user:cbt_password@postgres:5432/cbt_platform
      DATABASE_URL_SYNC: postgresql://cbt_user:cbt_password@postgres:5432/cbt_platform
      ENV: development
    env_file:
      - ./backend/.env
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ./backend:/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: cbt_frontend
    ports:
      - "5173:5173"
    environment:
      VITE_API_BASE_URL: http://localhost:8000/api/v1
    volumes:
      - ./frontend:/app
      - /app/node_modules
    command: npm run dev

volumes:
  postgres_data:
```

### Backend `Dockerfile`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install system deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python deps
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy app code
COPY . .

# Run smart startup (checks stamp, migrates only if needed) then start server
EXPOSE 8000
CMD ["sh", "-c", "python -m scripts.startup && uvicorn app.main:app --host 0.0.0.0 --port 8000"]
```

### Backend `requirements.txt`

```
# Core
fastapi==0.115.*
uvicorn[standard]==0.32.*
pydantic==2.*
pydantic-settings==2.*

# Database
sqlalchemy[asyncio]==2.0.*
asyncpg==0.30.*
alembic==1.14.*
psycopg2-binary==2.9.*  # For Alembic (sync)

# Auth
python-jose[cryptography]==3.3.*
passlib[bcrypt]==1.7.*

# Email
fastapi-mail==1.4.*

# Validation & Serialization
python-multipart==0.0.*
email-validator==2.*

# Utilities
python-dotenv==1.*
httpx==0.28.*

# Dev & Testing
pytest==8.*
pytest-asyncio==0.24.*
httpx==0.28.*
```

### Deployment Targets

| Component | Phase 1 | Phase 3 |
|---|---|---|
| Frontend | Vercel | AWS CloudFront + S3 / Oracle |
| Backend | Render (Docker) | AWS ECS / Oracle OCI / GCP Cloud Run |
| Database | Neon DB (managed PG) | Self-hosted PG on VPS / RDS |
| Storage | — (text only) | Cloudinary → S3/Oracle Object Storage |

---

## 13. Phase-wise Roadmap Summary

| Phase | Scope | Goal | Timeline |
|---|---|---|---|
| **Phase 1** | Core MVP — Auth, Admin, Teacher, Student, Wallet, Violations | **Market launch.** Fully functional exam platform | 4–5 weeks |
| **Phase 2** | Images (Cloudinary), Bilingual, Analytics, Notifications, SEO, Export | **Growth.** Enhanced UX & data insights | 3–4 weeks |
| **Phase 3** | Redis, CI/CD, Security hardening, Monitoring, Cloud migration, PWA | **Scale.** Production-grade infrastructure | 3–4 weeks |

> **Detailed plans:** See [phase1_plan.md](./phase1_plan.md), [phase2_plan.md](./phase2_plan.md), [phase3_plan.md](./phase3_plan.md)

---

*This document is the single source of truth for the CBT Platform architecture. All phase plans reference this PRD for schema, API, and component specifications.*
