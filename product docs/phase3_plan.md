# 🏗️ Phase 3 — Scale, Harden & Enterprise Readiness

> **Goal:** Transform the platform from a "works well" product to a "works reliably at scale" product. Infrastructure migration, CI/CD, monitoring, security hardening, and performance optimization.  
> **Prerequisites:** Phase 1 & Phase 2 fully deployed and stable  
> **Timeline:** 3–4 weeks  
> **Backend:** Python 3.12 + FastAPI + SQLAlchemy + Alembic (same stack, hardened)  
> **Outcome:** Production-grade infrastructure ready to handle 10,000+ concurrent users, deployable on any cloud provider, fully automated deployment pipeline.

---

## What Ships in Phase 3

| ✅ Phase 3 Features | Impact |
|---|---|
| Cloud migration readiness (AWS / GCP / Oracle) | Deploy anywhere |
| Redis caching layer | 10x faster config + session lookups |
| CI/CD pipeline (GitHub Actions) | Automated testing + deployment |
| Advanced rate limiting (per-role, per-endpoint) | API abuse prevention |
| Application monitoring (APM + error tracking) | Proactive issue detection |
| Database optimization (indexes, connection pooling, query analysis) | Sub-100ms API responses |
| Backup & disaster recovery plan | Data safety |
| Load testing & performance benchmarks | Confidence at scale |
| PWA (Progressive Web App) features | Mobile-native feel |
| API versioning & documentation (already Swagger — enhance) | Developer experience |
| Logging infrastructure (structured, centralized) | Debug in production |
| SSE for real-time notifications | No more polling |

---

## Implementation Tasks (5 Tasks for AI Agent)

---

### Task 1: Redis Caching + Database Optimization

**Scope:** Add Redis as a caching layer, optimize database queries, add performance indexes.

#### 1.1 Redis Integration

**Install:**
```bash
pip install redis[hiredis]
pip freeze > requirements.txt
```

**Cache service with abstraction:**

```python
# app/services/cache_service.py
from abc import ABC, abstractmethod
from typing import Any, Optional
import json
import redis.asyncio as redis
from app.core.config import settings


class ICacheService(ABC):
    @abstractmethod
    async def get(self, key: str) -> Optional[Any]:
        pass

    @abstractmethod
    async def set(self, key: str, value: Any, ttl_seconds: int = 300) -> None:
        pass

    @abstractmethod
    async def delete(self, key: str) -> None:
        pass

    @abstractmethod
    async def delete_pattern(self, pattern: str) -> None:
        pass


class RedisCacheService(ICacheService):
    def __init__(self):
        self.client = redis.from_url(
            settings.REDIS_URL,
            encoding="utf-8",
            decode_responses=True,
        )

    async def get(self, key: str) -> Optional[Any]:
        value = await self.client.get(key)
        return json.loads(value) if value else None

    async def set(self, key: str, value: Any, ttl_seconds: int = 300) -> None:
        await self.client.set(key, json.dumps(value, default=str), ex=ttl_seconds)

    async def delete(self, key: str) -> None:
        await self.client.delete(key)

    async def delete_pattern(self, pattern: str) -> None:
        async for key in self.client.scan_iter(match=pattern):
            await self.client.delete(key)

    async def close(self):
        await self.client.close()


class InMemoryCacheService(ICacheService):
    """Fallback for local dev without Redis."""
    def __init__(self):
        self._store: dict[str, Any] = {}

    async def get(self, key: str) -> Optional[Any]:
        return self._store.get(key)

    async def set(self, key: str, value: Any, ttl_seconds: int = 300) -> None:
        self._store[key] = value

    async def delete(self, key: str) -> None:
        self._store.pop(key, None)

    async def delete_pattern(self, pattern: str) -> None:
        import fnmatch
        keys_to_delete = [k for k in self._store if fnmatch.fnmatch(k, pattern)]
        for k in keys_to_delete:
            del self._store[k]


def get_cache_service() -> ICacheService:
    if settings.REDIS_URL:
        return RedisCacheService()
    return InMemoryCacheService()
```

**FastAPI lifespan integration:**
```python
# app/main.py — updated lifespan
from app.services.cache_service import get_cache_service

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    app.state.cache = get_cache_service()
    print("🚀 Starting CBT Platform API")
    yield
    # Shutdown
    if hasattr(app.state.cache, "close"):
        await app.state.cache.close()
    await engine.dispose()
    print("👋 Shutting down")
```

**What to cache:**

| Data | Cache Key | TTL | Invalidation |
|---|---|---|---|
| Exam cost config | `config:exam_cost` | 5 min | Admin updates config |
| User session ID | `session:{user_id}` | 15 min | Login/logout |
| Teacher dashboard metrics | `dashboard:teacher:{id}` | 2 min | Exam/student changes |
| Admin dashboard metrics | `dashboard:admin` | 2 min | Any mutation |
| Exam question count | `exam:{id}:qcount` | 10 min | Question add/delete |

**Cache-aside pattern:**
```python
# app/services/exam_service.py — example usage

async def get_exam_cost_config(db: AsyncSession, cache: ICacheService) -> ExamCostConfig:
    """Fetch cost config with cache-aside."""
    cached = await cache.get("config:exam_cost")
    if cached:
        return ExamCostConfig(**cached)

    result = await db.execute(select(ExamCostConfig))
    config = result.scalar_one()

    await cache.set("config:exam_cost", {
        "id": str(config.id),
        "cost_per_question": str(config.cost_per_question),
        "global_exam_publishing_fee": str(config.global_exam_publishing_fee),
    }, ttl_seconds=300)

    return config


# When admin updates config — invalidate cache
async def update_exam_cost_config(data, admin_id, db, cache):
    config = await db.execute(select(ExamCostConfig).with_for_update())
    config = config.scalar_one()
    config.cost_per_question = data.cost_per_question
    config.global_exam_publishing_fee = data.global_exam_publishing_fee
    config.updated_by = admin_id
    await db.flush()
    await cache.delete("config:exam_cost")  # Invalidate
    return config
```

#### 1.2 Database Optimization

**Add performance indexes via Alembic:**

```bash
alembic revision --autogenerate -m "add_performance_indexes"
```

The indexes already defined in the SQLAlchemy models (Phase 1) cover the critical paths. Additional composite indexes to add:

```python
# app/models/attempt.py — add to __table_args__
Index("ix_attempts_student_submitted", "student_id", "submitted_at"),
Index("ix_attempts_exam_submitted", "exam_id", "is_submitted"),

# app/models/wallet.py — add to Transaction __table_args__
Index("ix_transactions_wallet_created", "wallet_id", "created_at"),
Index("ix_transactions_reason_created", "reason", "created_at"),
```

**Connection pooling optimization:**
```python
# app/core/database.py — production tuning
engine = create_async_engine(
    settings.DATABASE_URL,
    echo=False,                    # No SQL logging in prod
    pool_size=20,                  # Base pool connections
    max_overflow=30,               # Extra connections under load
    pool_timeout=30,               # Seconds to wait for connection
    pool_recycle=1800,             # Recycle connections every 30 min
    pool_pre_ping=True,            # Test connections before use
)
```

**For Neon DB (serverless PG) — add to `DATABASE_URL`:**
```
?sslmode=require&prepared_statement_cache_size=0
```

**Query optimization checklist:**
- [ ] All list queries use `.options(load_only(...))` to select only needed columns
- [ ] Pagination uses `offset/limit` (switch to cursor-based for 100k+ rows in future)
- [ ] N+1 resolved with `selectinload()` / `joinedload()` in SQLAlchemy
- [ ] Exam cost config cached (most frequent read)
- [ ] Dashboard metrics use `func.count()` aggregations, not load-all

**Deliverables after Task 1:**
- ✅ Redis caching for hot data (config, sessions, dashboard)
- ✅ Cache invalidation on mutations
- ✅ Performance indexes on critical queries
- ✅ Connection pooling tuned for production
- ✅ Fallback in-memory cache for dev without Redis

---

### Task 2: CI/CD Pipeline + Automated Testing

**Scope:** GitHub Actions for automated testing, linting, and deployment.

#### 2.1 GitHub Actions Workflows

**`.github/workflows/backend-ci.yml`:**
```yaml
name: Backend CI

on:
  push:
    branches: [main, develop]
    paths: ['backend/**']
  pull_request:
    branches: [main]
    paths: ['backend/**']

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: cbt_test
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip
          cache-dependency-path: backend/requirements.txt

      - name: Install dependencies
        working-directory: backend
        run: pip install -r requirements.txt

      - name: Run linter
        working-directory: backend
        run: |
          pip install ruff
          ruff check app/ tests/

      - name: Run migrations
        working-directory: backend
        env:
          DATABASE_URL: postgresql+asyncpg://test_user:test_password@localhost:5432/cbt_test
          DATABASE_URL_SYNC: postgresql://test_user:test_password@localhost:5432/cbt_test
        run: alembic upgrade head

      - name: Run tests
        working-directory: backend
        env:
          DATABASE_URL: postgresql+asyncpg://test_user:test_password@localhost:5432/cbt_test
          DATABASE_URL_SYNC: postgresql://test_user:test_password@localhost:5432/cbt_test
          REDIS_URL: redis://localhost:6379
          JWT_ACCESS_SECRET: test-access-secret-64-chars-long-for-testing-purposes-only-here
          JWT_REFRESH_SECRET: test-refresh-secret-64-chars-long-for-testing-purposes-only-here
          ADMIN_EMAIL: admin@test.com
          ADMIN_PASSWORD: testpassword123
          ENV: test
        run: pytest tests/ -v --tb=short --cov=app --cov-report=term-missing

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Render
        uses: johnbeynon/render-deploy-action@v0.0.8
        with:
          service-id: ${{ secrets.RENDER_SERVICE_ID }}
          api-key: ${{ secrets.RENDER_API_KEY }}
```

**`.github/workflows/frontend-ci.yml`:**
```yaml
name: Frontend CI

on:
  push:
    branches: [main, develop]
    paths: ['frontend/**']
  pull_request:
    branches: [main]
    paths: ['frontend/**']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        working-directory: frontend
        run: npm ci

      - name: Lint
        working-directory: frontend
        run: npm run lint

      - name: Build
        working-directory: frontend
        env:
          VITE_API_BASE_URL: https://api.cbt.devophile.com/api/v1
        run: npm run build
      # Vercel auto-deploys from main — no manual step needed
```

#### 2.2 Test Suite

**Install test dependencies:**
```bash
pip install pytest pytest-asyncio httpx ruff pytest-cov
```

**`backend/tests/conftest.py`:**
```python
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from app.main import app
from app.core.database import get_db
from app.models.base import Base
from app.core.config import settings


# Test database engine
test_engine = create_async_engine(settings.DATABASE_URL, echo=False)
TestSessionLocal = async_sessionmaker(test_engine, class_=AsyncSession, expire_on_commit=False)


@pytest_asyncio.fixture
async def db_session():
    async with TestSessionLocal() as session:
        yield session
        await session.rollback()


@pytest_asyncio.fixture
async def client(db_session):
    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()


@pytest_asyncio.fixture
async def admin_headers(client):
    """Get auth headers for admin."""
    response = await client.post("/api/v1/auth/admin/login", json={
        "email": settings.ADMIN_EMAIL,
        "password": settings.ADMIN_PASSWORD,
    })
    token = response.json()["data"]["tokens"]["access_token"]
    return {"Authorization": f"Bearer {token}"}
```

**Critical test files:**

```python
# tests/integration/test_auth.py

import pytest

@pytest.mark.asyncio
async def test_admin_login_success(client):
    response = await client.post("/api/v1/auth/admin/login", json={
        "email": "admin@test.com",
        "password": "testpassword123",
    })
    assert response.status_code == 200
    data = response.json()
    assert data["success"] is True
    assert "access_token" in data["data"]["tokens"]

@pytest.mark.asyncio
async def test_admin_login_wrong_password(client):
    response = await client.post("/api/v1/auth/admin/login", json={
        "email": "admin@test.com",
        "password": "wrongpassword",
    })
    assert response.status_code == 401

@pytest.mark.asyncio
async def test_protected_route_without_token(client):
    response = await client.get("/api/v1/admin/dashboard")
    assert response.status_code == 403  # No bearer token
```

```python
# tests/integration/test_wallet.py

@pytest.mark.asyncio
async def test_credit_wallet(client, admin_headers, db_session):
    # Create a teacher first, then credit
    ...
    response = await client.post("/api/v1/admin/wallet/credit", json={
        "user_id": str(teacher_id),
        "amount": 100,
        "credit_type": "paid",
        "description": "Initial credit",
    }, headers=admin_headers)
    assert response.status_code == 200
    assert response.json()["data"]["balance"] == 100.0

@pytest.mark.asyncio
async def test_insufficient_balance_blocks_exam(client, ...):
    # Student starts exam with 0 balance → should get 402
    ...
    assert response.status_code == 402
```

```python
# tests/unit/test_score_calculator.py

from decimal import Decimal
from app.utils.score_calculator import calculate_score

def test_all_correct():
    questions = [
        {"id": "1", "correct_option": "1"},
        {"id": "2", "correct_option": "3"},
    ]
    answers = {"1": "1", "2": "3"}
    result = calculate_score(questions, answers, Decimal("4"), Decimal("1"))
    assert result.score == Decimal("8")
    assert result.correct == 2
    assert result.wrong == 0

def test_negative_marking():
    questions = [{"id": "1", "correct_option": "1"}]
    answers = {"1": "2"}  # Wrong
    result = calculate_score(questions, answers, Decimal("4"), Decimal("1"))
    assert result.score == Decimal("-1")
    assert result.wrong == 1

def test_unanswered_no_penalty():
    questions = [{"id": "1", "correct_option": "1"}]
    answers = {}
    result = calculate_score(questions, answers, Decimal("4"), Decimal("1"))
    assert result.score == Decimal("0")
    assert result.unanswered == 1
```

**Run tests:**
```bash
cd backend
pytest tests/ -v --tb=short --cov=app --cov-report=term-missing
```

**Deliverables after Task 2:**
- ✅ GitHub Actions CI (lint → test → deploy) for both backend and frontend
- ✅ Automated test suite (integration + unit) with pytest
- ✅ Auto-deploy to Render on main push
- ✅ Code coverage reporting (target: 70%+)
- ✅ Ruff linter enforcing code quality

---

### Task 3: Security Hardening + Rate Limiting

**Scope:** Production-grade security — granular rate limiting, input sanitization, audit logging.

#### 3.1 Rate Limiting with SlowAPI

**Install:**
```bash
pip install slowapi
```

**Rate limit configuration:**
```python
# app/core/rate_limiter.py
from slowapi import Limiter
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from fastapi import Request
from fastapi.responses import JSONResponse

limiter = Limiter(
    key_func=get_remote_address,
    default_limits=["200/15minutes"],
    storage_uri=settings.REDIS_URL or "memory://",
)


async def rate_limit_exceeded_handler(request: Request, exc: RateLimitExceeded):
    return JSONResponse(
        status_code=429,
        content={
            "success": False,
            "message": "Too many requests. Please try again later.",
        },
    )
```

**Apply to `app/main.py`:**
```python
from slowapi import _rate_limit_exceeded_handler
from slowapi.errors import RateLimitExceeded
from app.core.rate_limiter import limiter

app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, rate_limit_exceeded_handler)
```

**Per-route rate limits:**
```python
# app/api/auth.py
from app.core.rate_limiter import limiter

@router.post("/admin/login")
@limiter.limit("10/15minutes")  # 10 login attempts per 15 min
async def admin_login(request: Request, body: LoginRequest):
    ...

@router.post("/student/register")
@limiter.limit("5/15minutes")   # 5 registration attempts per 15 min
async def register_student(request: Request, body: RegisterStudentRequest, ...):
    ...

@router.post("/verify-otp")
@limiter.limit("3/15minutes")   # 3 OTP verifications per 15 min
async def verify_otp(request: Request, body: VerifyOtpRequest, ...):
    ...

# app/api/student.py
@router.post("/exams/{exam_id}/start")
@limiter.limit("5/minute")      # 5 exam start attempts per minute
async def start_exam(request: Request, exam_id: uuid.UUID, ...):
    ...

# app/api/upload.py
@router.post("/image")
@limiter.limit("100/hour")      # 100 uploads per hour
async def upload_image(request: Request, ...):
    ...
```

#### 3.2 Security Middleware

```python
# app/main.py — security middleware stack

from fastapi.middleware.trustedhost import TrustedHostMiddleware

# Trusted hosts (prevent host header attacks)
if settings.ENV == "production":
    app.add_middleware(
        TrustedHostMiddleware,
        allowed_hosts=["api.cbt.devophile.com", "*.onrender.com"],
    )

# CORS (already configured in Phase 1, tighten for production)
app.add_middleware(
    CORSMiddleware,
    allow_origins=[settings.FRONTEND_URL],  # Exact match only
    allow_credentials=True,
    allow_methods=["GET", "POST", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    max_age=600,  # Cache preflight for 10 min
)
```

**Security headers middleware:**
```python
# app/middleware/security_headers.py
from starlette.middleware.base import BaseHTTPMiddleware
from fastapi import Request, Response


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response: Response = await call_next(request)
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        response.headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()"
        return response

# Add to app
app.add_middleware(SecurityHeadersMiddleware)
```

#### 3.3 Input Sanitization

Pydantic v2 handles most sanitization automatically. Additional measures:

```python
# app/utils/sanitize.py
import re
from markupsafe import escape


def sanitize_html(text: str) -> str:
    """Strip HTML tags and escape special characters."""
    clean = re.sub(r'<[^>]+>', '', text)
    return str(escape(clean))


def sanitize_search_query(query: str) -> str:
    """Prevent SQL injection in search parameters."""
    # SQLAlchemy parameterizes queries, but belt-and-suspenders
    return re.sub(r'[;\'\"\\]', '', query).strip()[:200]
```

#### 3.4 Audit Logging

```python
# app/middleware/audit_logger.py
import json
from datetime import datetime
from starlette.middleware.base import BaseHTTPMiddleware
from fastapi import Request
import logging

audit_logger = logging.getLogger("audit")


AUDIT_ROUTES = {
    "POST /api/v1/admin/wallet/credit": "WALLET_CREDIT",
    "PATCH /api/v1/admin/config/exam-cost": "CONFIG_UPDATE",
    "POST /api/v1/admin/teachers": "TEACHER_CREATED",
    "DELETE /api/v1/admin/teachers": "TEACHER_DELETED",
    "POST /api/v1/student/exams": "EXAM_STARTED",
}


class AuditLogMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)

        route_key = f"{request.method} {request.url.path}"
        for pattern, action in AUDIT_ROUTES.items():
            if route_key.startswith(pattern.split(" ", 1)[1]) and request.method == pattern.split(" ")[0]:
                if response.status_code < 400:
                    audit_logger.info(json.dumps({
                        "action": action,
                        "user": getattr(request.state, "user_id", None),
                        "method": request.method,
                        "path": str(request.url.path),
                        "status": response.status_code,
                        "ip": request.client.host if request.client else None,
                        "timestamp": datetime.utcnow().isoformat(),
                    }))
                break

        return response
```

**Deliverables after Task 3:**
- ✅ Granular rate limiting per endpoint (Redis-backed via SlowAPI)
- ✅ Security headers (HSTS, X-Frame, XSS protection)
- ✅ Trusted host middleware
- ✅ Input sanitization layer
- ✅ Audit logging for sensitive admin operations

---

### Task 4: Monitoring, Logging & Real-time Notifications

**Scope:** Structured logging, error tracking, and real-time notification delivery.

#### 4.1 Structured Logging

**Install:**
```bash
pip install structlog
```

**Logger configuration:**
```python
# app/core/logging_config.py
import structlog
import logging

def setup_logging(env: str = "development"):
    """Configure structured logging with structlog."""
    processors = [
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
    ]

    if env == "production":
        # JSON output for production (parseable by log aggregators)
        processors.append(structlog.processors.JSONRenderer())
    else:
        # Pretty console output for development
        processors.append(structlog.dev.ConsoleRenderer())

    structlog.configure(
        processors=processors,
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )

logger = structlog.get_logger()
```

**Usage throughout the app:**
```python
# Replace all print() and logging.info() with structured logger
from app.core.logging_config import logger

# In auth_service.py
await logger.ainfo("user_login", user_id=str(user.id), role=user.role.value, event="login_success")

# In wallet_service.py
await logger.ainfo("wallet_debit", user_id=str(user_id), amount=float(amount),
                    reason=reason.value, balance_after=float(new_balance))

# In error handler
await logger.aerror("unhandled_error", error=str(err), path=request.url.path,
                    method=request.method)
```

**Request logging middleware:**
```python
# app/middleware/request_logger.py
import time
from starlette.middleware.base import BaseHTTPMiddleware
from app.core.logging_config import logger


class RequestLogMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        start_time = time.perf_counter()
        response = await call_next(request)
        duration_ms = round((time.perf_counter() - start_time) * 1000, 2)

        log_level = "info" if response.status_code < 400 else "warning" if response.status_code < 500 else "error"
        getattr(logger, log_level)(
            "http_request",
            method=request.method,
            path=str(request.url.path),
            status=response.status_code,
            duration_ms=duration_ms,
            client_ip=request.client.host if request.client else None,
        )

        return response
```

#### 4.2 Error Tracking (Sentry)

**Install:**
```bash
pip install sentry-sdk[fastapi]

# Frontend
cd frontend && npm install @sentry/react
```

**Backend Sentry setup:**
```python
# app/core/sentry.py
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration
from sentry_sdk.integrations.sqlalchemy import SqlalchemyIntegration
from app.core.config import settings


def init_sentry():
    if not settings.SENTRY_DSN:
        return

    sentry_sdk.init(
        dsn=settings.SENTRY_DSN,
        environment=settings.ENV,
        traces_sample_rate=0.1 if settings.ENV == "production" else 1.0,
        profiles_sample_rate=0.1,
        integrations=[
            FastApiIntegration(transaction_style="endpoint"),
            SqlalchemyIntegration(),
        ],
        send_default_pii=False,  # Don't send user emails/IPs
    )
```

**Call in `app/main.py`:**
```python
from app.core.sentry import init_sentry
init_sentry()
```

**Frontend Sentry:**
```jsx
// src/main.jsx
import * as Sentry from '@sentry/react';

Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  environment: import.meta.env.MODE,
  tracesSampleRate: 0.1,
  integrations: [Sentry.browserTracingIntegration(), Sentry.replayIntegration()],
  replaysOnErrorSampleRate: 1.0,
});
```

#### 4.3 Real-time Notifications (SSE)

**FastAPI has native SSE support via `StreamingResponse`:**

```python
# app/api/notification.py
import asyncio
import json
from fastapi import APIRouter, Depends, Request
from fastapi.responses import StreamingResponse
from app.core.dependencies import get_current_user
from app.models.user import User

router = APIRouter(prefix="/notifications", tags=["Notifications"])

# In-memory connection store (use Redis pub/sub for multi-worker)
active_connections: dict[str, asyncio.Queue] = {}


async def notification_event_generator(user_id: str, request: Request):
    """SSE generator — yields events as they arrive."""
    queue = asyncio.Queue()
    active_connections[user_id] = queue

    try:
        while True:
            # Check if client disconnected
            if await request.is_disconnected():
                break

            try:
                # Wait for notification with timeout (heartbeat every 30s)
                data = await asyncio.wait_for(queue.get(), timeout=30.0)
                yield f"event: notification\ndata: {json.dumps(data)}\n\n"
            except asyncio.TimeoutError:
                # Send heartbeat
                yield f"event: heartbeat\ndata: ping\n\n"

    finally:
        active_connections.pop(user_id, None)


@router.get("/stream")
async def notification_stream(
    request: Request,
    current_user: User = Depends(get_current_user),
):
    return StreamingResponse(
        notification_event_generator(str(current_user.id), request),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",
        },
    )


# Called by notification_service when creating a notification
async def push_to_user(user_id: str, notification: dict):
    queue = active_connections.get(user_id)
    if queue:
        await queue.put(notification)
```

**Frontend SSE client (`src/hooks/useNotificationStream.js`):**
```javascript
import { useEffect, useRef } from 'react';
import { useAuthStore } from '../store/authStore';
import { useNotificationStore } from '../store/notificationStore';
import toast from 'react-hot-toast';

export function useNotificationStream() {
  const accessToken = useAuthStore((s) => s.accessToken);
  const addNotification = useNotificationStore((s) => s.addNotification);
  const eventSourceRef = useRef(null);

  useEffect(() => {
    if (!accessToken) return;

    // Use EventSource with token in query param (SSE limitation)
    const url = `${import.meta.env.VITE_API_BASE_URL}/notifications/stream?token=${accessToken}`;
    const es = new EventSource(url);

    es.addEventListener('notification', (event) => {
      const notification = JSON.parse(event.data);
      addNotification(notification);
      if (['low_balance', 'exam_blocked'].includes(notification.type)) {
        toast.error(notification.message);
      }
    });

    es.onerror = () => {
      // EventSource auto-reconnects
    };

    eventSourceRef.current = es;
    return () => es.close();
  }, [accessToken]);
}
```

> [!NOTE]
> For multi-worker deployments (e.g., Render with 2+ workers), replace the in-memory `active_connections` dict with **Redis Pub/Sub**. Each worker subscribes to a Redis channel, and `push_to_user()` publishes to the channel instead.

**Deliverables after Task 4:**
- ✅ Structured logging with structlog (JSON in prod, pretty in dev)
- ✅ Error tracking with Sentry (backend + frontend)
- ✅ Real-time notifications via SSE (no polling)
- ✅ Request logging middleware (method, path, status, duration)

---

### Task 5: Cloud Migration Prep, Load Testing & PWA

**Scope:** Production Docker, cloud deployment guides, performance benchmarks, PWA.

#### 5.1 Production Dockerfile

```dockerfile
# backend/Dockerfile.production
FROM python:3.12-slim AS builder

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.12-slim AS runner
WORKDIR /app

# Security: non-root user
RUN groupadd --system appgroup && useradd --system --gid appgroup appuser

# Copy installed packages from builder
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin

# Install runtime-only system deps
RUN apt-get update && apt-get install -y --no-install-recommends libpq5 \
    && rm -rf /var/lib/apt/lists/*

COPY . .

USER appuser
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD python -c "import httpx; r = httpx.get('http://localhost:8000/health'); assert r.status_code == 200"

CMD ["sh", "-c", "alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4"]
```

**Docker Compose for production:**
```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.production
    container_name: cbt_backend
    restart: unless-stopped
    ports:
      - "8000:8000"
    env_file:
      - ./backend/.env.production
    depends_on:
      redis:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  redis:
    image: redis:7-alpine
    container_name: cbt_redis
    restart: unless-stopped
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s

  nginx:
    image: nginx:alpine
    container_name: cbt_nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - backend

volumes:
  redis_data:
```

#### 5.2 Cloud Deployment Guides

**AWS:**
```
Frontend: S3 + CloudFront
Backend:  ECS Fargate (Docker) or Lambda + API Gateway
Database: RDS PostgreSQL or Aurora Serverless
Cache:    ElastiCache Redis
Storage:  S3 (swap StorageService)
Email:    SES (swap EmailService)
```

**Oracle Cloud:**
```
Frontend: Object Storage + CDN
Backend:  Container Instances or Compute VM (Docker)
Database: Autonomous DB or self-hosted PG
Cache:    OCI Cache with Redis
Storage:  Object Storage (swap StorageService)
```

**GCP:**
```
Frontend: Cloud Storage + Cloud CDN
Backend:  Cloud Run (Docker, auto-scaling, serverless)
Database: Cloud SQL PostgreSQL
Cache:    Memorystore Redis
Storage:  Cloud Storage (swap StorageService)
```

**Service swap pattern — only change the factory function:**
```python
# app/services/storage_service.py
def get_storage_service() -> IStorageService:
    match settings.STORAGE_PROVIDER:
        case "s3":       return S3StorageService()
        case "oracle":   return OracleStorageService()
        case "gcs":      return GCSStorageService()
        case _:          return CloudinaryStorageService()

# app/services/email_service.py
def get_email_service() -> IEmailService:
    match settings.EMAIL_PROVIDER:
        case "ses":      return SESEmailService()
        case _:          return SMTPEmailService()
```

#### 5.3 Load Testing

**Install:**
```bash
pip install locust
```

**`backend/locustfile.py`:**
```python
from locust import HttpUser, task, between
import json


class StudentUser(HttpUser):
    wait_time = between(1, 5)
    weight = 6  # 60% of traffic

    def on_start(self):
        response = self.client.post("/api/v1/auth/student/login", json={
            "email": "loadtest-student@test.com",
            "password": "testpassword123",
        })
        if response.status_code == 200:
            self.token = response.json()["data"]["tokens"]["access_token"]
            self.headers = {"Authorization": f"Bearer {self.token}"}
        else:
            self.headers = {}

    @task(3)
    def browse_global_exams(self):
        self.client.get("/api/v1/student/exams/global", headers=self.headers)

    @task(2)
    def browse_my_exams(self):
        self.client.get("/api/v1/student/exams/my", headers=self.headers)

    @task(1)
    def check_wallet(self):
        self.client.get("/api/v1/student/wallet", headers=self.headers)


class TeacherUser(HttpUser):
    wait_time = between(2, 8)
    weight = 3  # 30% of traffic

    def on_start(self):
        response = self.client.post("/api/v1/auth/teacher/login", json={
            "email": "loadtest-teacher@test.com",
            "password": "testpassword123",
        })
        if response.status_code == 200:
            self.token = response.json()["data"]["tokens"]["access_token"]
            self.headers = {"Authorization": f"Bearer {self.token}"}
        else:
            self.headers = {}

    @task(3)
    def view_dashboard(self):
        self.client.get("/api/v1/teacher/dashboard", headers=self.headers)

    @task(2)
    def list_exams(self):
        self.client.get("/api/v1/teacher/exams", headers=self.headers)

    @task(1)
    def list_students(self):
        self.client.get("/api/v1/teacher/students", headers=self.headers)


class HealthCheckUser(HttpUser):
    wait_time = between(1, 2)
    weight = 1  # 10% of traffic

    @task
    def health(self):
        self.client.get("/health")
```

**Run load test:**
```bash
# Start locust web UI
locust -f locustfile.py --host=https://api.cbt.devophile.com

# Or headless mode
locust -f locustfile.py --host=https://api.cbt.devophile.com \
    --headless -u 200 -r 10 --run-time 5m
```

**Performance benchmarks (targets):**

| Metric | Target |
|---|---|
| P95 response time | < 200ms |
| P99 response time | < 500ms |
| Error rate | < 0.1% |
| Throughput | > 100 req/sec |
| Exam start (atomic tx) | < 300ms |
| Concurrent exam takers | > 500 |

#### 5.4 PWA Configuration

**`frontend/public/manifest.json`:**
```json
{
  "name": "Devophile CBT Platform",
  "short_name": "CBT",
  "description": "Professional Computer Based Testing",
  "start_url": "/student",
  "display": "standalone",
  "background_color": "#0f172a",
  "theme_color": "#6366f1",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

**Register service worker in `src/main.jsx`:**
```javascript
if ('serviceWorker' in navigator && import.meta.env.PROD) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/sw.js');
  });
}
```

#### 5.5 Enhanced API Documentation

FastAPI already provides Swagger at `/docs`. Enhance with:

```python
# app/main.py — enhanced OpenAPI config
app = FastAPI(
    title="Devophile CBT Platform",
    description="""
    ## Computer Based Testing Platform API

    ### Roles
    - **Super Admin** — Manages teachers, students, wallets, and platform config
    - **Teacher** — Creates exams, manages students and batches
    - **Student** — Browses and attempts exams, views results

    ### Authentication
    All protected routes require a Bearer JWT token in the Authorization header.
    """,
    version="1.0.0",
    docs_url="/docs",
    redoc_url="/redoc",
    openapi_tags=[
        {"name": "Authentication", "description": "Login, register, OTP, token management"},
        {"name": "Admin", "description": "Super Admin operations"},
        {"name": "Teacher", "description": "Teacher panel operations"},
        {"name": "Student", "description": "Student panel operations"},
        {"name": "Wallet", "description": "Wallet and transaction management"},
        {"name": "Upload", "description": "File upload (images)"},
        {"name": "Notifications", "description": "In-app notifications"},
    ],
)
```

#### 5.6 Backup & Disaster Recovery

**Neon DB:** Built-in PITR (Point-in-Time Recovery) — 7 days free, 30 days paid.

**For self-hosted PostgreSQL:**
```bash
#!/bin/bash
# scripts/backup.sh — Run via cron: 0 2 * * *

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/cbt"
BACKUP_FILE="$BACKUP_DIR/cbt_platform_$TIMESTAMP.sql.gz"

mkdir -p $BACKUP_DIR
pg_dump $DATABASE_URL | gzip > $BACKUP_FILE

# Upload to cloud storage (example for Oracle OCI)
# oci os object put --bucket-name cbt-backups --file $BACKUP_FILE

# Cleanup older than 30 days
find $BACKUP_DIR -type f -mtime +30 -delete
echo "Backup completed: $BACKUP_FILE"
```

**Pre-deployment checklist:**
```bash
# Before every deployment:
# 1. Run tests
pytest tests/ -v

# 2. Check migration
alembic check  # Verify no pending model changes

# 3. Backup database
pg_dump $DATABASE_URL > backup_pre_deploy.sql

# 4. Deploy
git push origin main  # Triggers CI/CD

# 5. Verify
curl https://api.cbt.devophile.com/health
curl https://api.cbt.devophile.com/docs  # Swagger loads
```

**Deliverables after Task 5:**
- ✅ Production Docker (multi-stage, non-root user, health checks, 4 workers)
- ✅ Docker Compose for prod (backend + Redis + Nginx)
- ✅ Cloud migration guides (AWS, GCP, Oracle)
- ✅ Load testing with Locust (benchmarked)
- ✅ PWA manifest + service worker
- ✅ Enhanced Swagger documentation
- ✅ Backup & DR plan

---

## Phase 3 — Full Deliverables Summary

| Feature | Status |
|---|---|
| Redis caching layer | ✅ Live |
| Database performance indexes | ✅ Live |
| CI/CD pipeline (GitHub Actions) | ✅ Live |
| Automated test suite (pytest, 70%+ coverage) | ✅ Live |
| Granular rate limiting (SlowAPI + Redis) | ✅ Live |
| Security hardening (headers, trusted host, audit) | ✅ Live |
| Structured logging (structlog) | ✅ Live |
| Error tracking (Sentry) | ✅ Live |
| Real-time notifications (SSE) | ✅ Live |
| Production Docker (multi-stage, 4 workers) | ✅ Ready |
| Cloud migration guides | ✅ Documented |
| Load testing (Locust) | ✅ Benchmarked |
| PWA (mobile installable) | ✅ Live |
| Enhanced API docs (Swagger) | ✅ Live |
| Backup & DR plan | ✅ Documented |

---

## New Dependencies Added in Phase 3

| Package | Purpose | Location |
|---|---|---|
| `redis[hiredis]` | Redis client (async) | Backend (pip) |
| `slowapi` | Rate limiting | Backend (pip) |
| `structlog` | Structured logging | Backend (pip) |
| `sentry-sdk[fastapi]` | Error tracking | Backend (pip) |
| `@sentry/react` | Error tracking | Frontend (npm) |
| `locust` | Load testing | Backend (pip, dev only) |
| `ruff` | Python linter | Backend (pip, dev only) |

---

## Post Phase 3 — What's Next?

| Feature | Description |
|---|---|
| Mobile app (React Native) | Native iOS/Android using same API |
| Bulk question import (Excel/CSV) | Teachers upload via spreadsheet |
| Question bank & categories | Reusable pools across exams |
| AI question generation | GPT-based from topics |
| Payment gateway (Razorpay/Stripe) | Students buy coins online |
| Multi-tenant white-label | Branded instances for coaching chains |
| Camera proctoring | Webcam + screen share monitoring |
| Adaptive testing | Dynamic difficulty adjustment |

---

*This completes the 3-phase technical architecture. Reference:*
- *Central PRD: [technical_prd.md](./technical_prd.md)*
- *Phase 1 (MVP): [phase1_plan.md](./phase1_plan.md)*
- *Phase 2 (Growth): [phase2_plan.md](./phase2_plan.md)*
