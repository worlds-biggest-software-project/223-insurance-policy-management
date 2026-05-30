# Insurance Policy Management — Phased Development Plan

> Project: 223-insurance-policy-management · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12+ | Insurance domain is data-heavy with complex business rules. Python's readability, rich ecosystem for financial calculations, and strong ORM/validation libraries (Pydantic, SQLAlchemy) suit the domain. AI/ML integration for risk scoring and claims triage is first-class. |
| API framework | FastAPI 0.115+ | Async-native, auto-generates OpenAPI 3.1 specs (aligning with the OAS 3.2 industry standard), Pydantic v2 integration for request/response validation, dependency injection for tenant isolation, and high performance via uvicorn/ASGI. |
| Database | PostgreSQL 16+ | Industry standard for insurance systems. JSONB support enables the hybrid relational+JSONB data model (Data Model Suggestion 3). Row-level security for multi-tenancy. Full ACID compliance for financial data. GIN indexes for JSONB queries. |
| ORM / Query | SQLAlchemy 2.0+ with Alembic | Type-annotated ORM with async support. Alembic handles schema migrations. Hybrid approach: ORM for CRUD, raw SQL for complex reporting queries. |
| Validation | Pydantic v2 | Request/response validation, settings management, JSON Schema generation for JSONB `details` validation, ACORD data structure mapping. |
| Authentication | OAuth 2.0 + JWT (python-jose) | RFC 6749 / OpenID Connect compliance. JWT tokens for stateless API auth. Role-based access control with scopes matching user roles (underwriter, agent, adjuster, etc.). |
| Task queue | Celery 5 + Redis | Async processing for premium calculations, ACORD XML import/export, batch renewals, AI risk scoring, document generation. Redis as broker and result backend. |
| Cache | Redis 7+ | Session cache, rate limiting, rating table caching, frequently-accessed policy lookups. |
| Payment processing | Stripe SDK (stripe-python) | PCI DSS v4.0.1 compliance via tokenisation — no raw card data touches the platform. Stripe handles card storage, ACH, and refunds. |
| Document storage | S3-compatible (MinIO for self-hosted, AWS S3 for cloud) | Policy documents, claim photos, correspondence. Pre-signed URLs for secure access. |
| Frontend | None (MVP) — API-first | The MVP is a pure API backend. Portal frontends (agent, customer, underwriter) are v1.1 scope. This mirrors how BriteCore and Guidewire expose APIs first, with portals as consumers. |
| Containerisation | Docker + Docker Compose | Self-hosted deployment target. Multi-stage builds for small images. Compose for local dev (API + PostgreSQL + Redis + MinIO). |
| Testing | pytest + pytest-asyncio + httpx | pytest for unit/integration tests. httpx.AsyncClient for API endpoint testing. Factory Boy for test data generation. |
| Code quality | Ruff (linting + formatting) + mypy (type checking) | Ruff replaces flake8/isort/black with a single fast tool. mypy enforces type safety across the codebase. |
| Package manager | uv | Fast, reliable Python package management. Lockfile for reproducible builds. |
| ACORD processing | lxml + xmlschema | ACORD XML import/export via lxml for parsing and xmlschema for XSD validation. |
| AI/ML | scikit-learn (MVP risk scoring) + OpenAI SDK (optional LLM features) | scikit-learn for tabular risk scoring models. OpenAI SDK for optional LLM-powered features (policy language generation, claims triage NLP) in later phases. |

### Project Structure

```
insurance-policy-management/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── alembic/
│   ├── env.py
│   └── versions/
├── src/
│   └── ipm/
│       ├── __init__.py
│       ├── main.py                      # FastAPI app factory
│       ├── config.py                    # Pydantic settings
│       ├── database.py                  # SQLAlchemy engine & session
│       ├── dependencies.py              # FastAPI dependency injection
│       ├── middleware/
│       │   ├── __init__.py
│       │   ├── tenant.py                # Tenant resolution middleware
│       │   ├── auth.py                  # JWT authentication
│       │   └── audit.py                 # Audit logging middleware
│       ├── models/
│       │   ├── __init__.py
│       │   ├── tenant.py
│       │   ├── user.py
│       │   ├── party.py
│       │   ├── product.py
│       │   ├── policy.py
│       │   ├── endorsement.py
│       │   ├── insured_item.py
│       │   ├── claim.py
│       │   ├── billing.py
│       │   ├── reinsurance.py
│       │   ├── document.py
│       │   ├── audit.py
│       │   └── commission.py
│       ├── schemas/
│       │   ├── __init__.py
│       │   ├── tenant.py
│       │   ├── user.py
│       │   ├── party.py
│       │   ├── product.py
│       │   ├── policy.py
│       │   ├── endorsement.py
│       │   ├── claim.py
│       │   ├── billing.py
│       │   └── common.py
│       ├── api/
│       │   ├── __init__.py
│       │   ├── router.py                # Root router aggregation
│       │   ├── tenants.py
│       │   ├── users.py
│       │   ├── parties.py
│       │   ├── products.py
│       │   ├── policies.py
│       │   ├── endorsements.py
│       │   ├── claims.py
│       │   ├── billing.py
│       │   ├── documents.py
│       │   └── health.py
│       ├── services/
│       │   ├── __init__.py
│       │   ├── policy_service.py
│       │   ├── rating_engine.py
│       │   ├── claims_service.py
│       │   ├── billing_service.py
│       │   ├── underwriting_service.py
│       │   ├── endorsement_service.py
│       │   ├── document_service.py
│       │   ├── acord_service.py
│       │   └── commission_service.py
│       ├── acord/
│       │   ├── __init__.py
│       │   ├── xml_parser.py
│       │   ├── xml_builder.py
│       │   ├── schemas/                 # ACORD XSD files
│       │   └── mapping.py              # ACORD ↔ internal model mapping
│       ├── tasks/
│       │   ├── __init__.py
│       │   ├── celery_app.py
│       │   ├── renewal_tasks.py
│       │   ├── billing_tasks.py
│       │   └── document_tasks.py
│       └── utils/
│           ├── __init__.py
│           ├── policy_number.py
│           ├── currency.py
│           └── dates.py
├── tests/
│   ├── conftest.py                      # Fixtures: test DB, client, factories
│   ├── factories/
│   │   ├── __init__.py
│   │   ├── tenant_factory.py
│   │   ├── party_factory.py
│   │   ├── policy_factory.py
│   │   └── claim_factory.py
│   ├── unit/
│   │   ├── test_rating_engine.py
│   │   ├── test_policy_service.py
│   │   ├── test_billing_service.py
│   │   └── test_acord_parser.py
│   ├── integration/
│   │   ├── test_policy_api.py
│   │   ├── test_claims_api.py
│   │   ├── test_billing_api.py
│   │   └── test_endorsement_api.py
│   └── fixtures/
│       ├── acord_samples/
│       │   ├── policy_issuance.xml
│       │   └── claim_fnol.xml
│       └── rating_tables/
│           └── ho3_territory.json
├── scripts/
│   ├── seed_data.py                     # Development seed data
│   └── migrate.sh                       # Migration helper
└── docs/
    └── openapi-extensions.md
```

---

## Phase 1: Foundation & Project Scaffolding

### Purpose
Establish the project skeleton, development toolchain, database connection, configuration management, Docker infrastructure, and health check endpoint. After this phase, a developer can clone the repo, run `docker compose up`, and hit a health check endpoint that confirms the API, database, and Redis are connected.

### Tasks

#### 1.1 — Project Initialisation and Dependency Management

**What**: Create the Python project with uv, configure pyproject.toml, and install core dependencies.

**Design**:

```toml
# pyproject.toml
[project]
name = "insurance-policy-management"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.32.0",
    "sqlalchemy[asyncio]>=2.0.36",
    "asyncpg>=0.30.0",
    "alembic>=1.14.0",
    "pydantic>=2.10.0",
    "pydantic-settings>=2.7.0",
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "redis>=5.2.0",
    "celery[redis]>=5.4.0",
    "httpx>=0.28.0",
    "lxml>=5.3.0",
    "boto3>=1.35.0",
    "python-multipart>=0.0.18",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=6.0.0",
    "httpx>=0.28.0",
    "factory-boy>=3.3.0",
    "ruff>=0.8.0",
    "mypy>=1.13.0",
    "sqlalchemy[mypy]>=2.0.36",
]

[tool.ruff]
target-version = "py312"
line-length = 120

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP", "B", "SIM", "TCH"]

[tool.mypy]
python_version = "3.12"
strict = true
plugins = ["pydantic.mypy", "sqlalchemy.ext.mypy.plugin"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

**Testing**:
- Unit: `uv sync` completes without errors
- Unit: `uv run python -c "import fastapi; import sqlalchemy; import pydantic"` succeeds

#### 1.2 — Configuration Management

**What**: Create a Pydantic-based settings module that reads configuration from environment variables with sensible defaults.

**Design**:

```python
# src/ipm/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Application
    app_name: str = "Insurance Policy Management"
    app_version: str = "0.1.0"
    debug: bool = False
    log_level: str = "INFO"

    # Database
    database_url: str = "postgresql+asyncpg://ipm:ipm@localhost:5432/ipm"
    database_pool_size: int = 20
    database_max_overflow: int = 10
    database_echo: bool = False

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # Auth / JWT
    jwt_secret_key: str = "CHANGE-ME-IN-PRODUCTION"
    jwt_algorithm: str = "HS256"
    jwt_access_token_expire_minutes: int = 60

    # S3 / MinIO
    s3_endpoint_url: str = "http://localhost:9000"
    s3_access_key: str = "minioadmin"
    s3_secret_key: str = "minioadmin"
    s3_bucket_name: str = "ipm-documents"
    s3_region: str = "us-east-1"

    # Celery
    celery_broker_url: str = "redis://localhost:6379/1"
    celery_result_backend: str = "redis://localhost:6379/2"

    # Payment (Stripe)
    stripe_api_key: str = ""
    stripe_webhook_secret: str = ""

    model_config = {"env_prefix": "IPM_", "env_file": ".env"}

def get_settings() -> Settings:
    return Settings()
```

**Testing**:
- Unit: default settings instantiate without errors
- Unit: `IPM_DATABASE_URL` env var overrides default → confirmed in Settings instance
- Unit: `IPM_DEBUG=true` env var → `settings.debug is True`
- Unit: missing optional env vars → defaults applied correctly

#### 1.3 — Database Engine and Session Management

**What**: Configure SQLAlchemy async engine, session factory, and base model class with common audit columns.

**Design**:

```python
# src/ipm/database.py
from datetime import datetime
from uuid import uuid4

from sqlalchemy import MetaData, func
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

from ipm.config import get_settings

NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        server_default=func.now(), nullable=False
    )
    updated_at: Mapped[datetime] = mapped_column(
        server_default=func.now(), onupdate=func.now(), nullable=False
    )

def create_engine():
    settings = get_settings()
    return create_async_engine(
        settings.database_url,
        pool_size=settings.database_pool_size,
        max_overflow=settings.database_max_overflow,
        echo=settings.database_echo,
    )

engine = create_engine()
async_session_factory = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db() -> AsyncSession:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

**Testing**:
- Integration: `create_engine()` connects to test PostgreSQL database
- Integration: `get_db()` yields a session, commits on success, rolls back on exception
- Unit: `Base` subclass creates table with correct naming convention
- Unit: `TimestampMixin` adds `created_at` and `updated_at` with server defaults

#### 1.4 — FastAPI Application Factory and Health Check

**What**: Create the FastAPI app factory with CORS, lifespan events, and a health check endpoint that verifies database and Redis connectivity.

**Design**:

```python
# src/ipm/main.py
from contextlib import asynccontextmanager

import redis.asyncio as redis
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy import text

from ipm.config import get_settings
from ipm.database import engine

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    settings = get_settings()
    app.state.redis = redis.from_url(settings.redis_url)
    yield
    # Shutdown
    await app.state.redis.aclose()
    await engine.dispose()

def create_app() -> FastAPI:
    settings = get_settings()
    app = FastAPI(
        title=settings.app_name,
        version=settings.app_version,
        lifespan=lifespan,
    )
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["*"],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
    from ipm.api.router import api_router
    app.include_router(api_router, prefix="/api/v1")
    return app

app = create_app()
```

```python
# src/ipm/api/health.py
from fastapi import APIRouter, Request
from sqlalchemy import text

from ipm.database import async_session_factory

router = APIRouter(tags=["health"])

@router.get("/health")
async def health_check(request: Request):
    checks = {}
    # Database
    try:
        async with async_session_factory() as session:
            await session.execute(text("SELECT 1"))
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"error: {e}"
    # Redis
    try:
        await request.app.state.redis.ping()
        checks["redis"] = "ok"
    except Exception as e:
        checks["redis"] = f"error: {e}"
    status = "healthy" if all(v == "ok" for v in checks.values()) else "degraded"
    return {"status": status, "checks": checks, "version": request.app.version}
```

```python
# src/ipm/api/router.py
from fastapi import APIRouter

from ipm.api.health import router as health_router

api_router = APIRouter()
api_router.include_router(health_router)
```

**Testing**:
- Integration: `GET /api/v1/health` returns `{"status": "healthy", ...}` when DB and Redis are up
- Integration: `GET /api/v1/health` returns `{"status": "degraded", ...}` when Redis is down
- Unit: `create_app()` returns a FastAPI instance with CORS middleware configured
- Unit: OpenAPI spec at `/openapi.json` is valid JSON with expected title and version

#### 1.5 — Docker and Docker Compose

**What**: Create Dockerfile (multi-stage) and docker-compose.yml for local development with PostgreSQL, Redis, and MinIO.

**Design**:

```dockerfile
# Dockerfile
FROM python:3.12-slim AS base
WORKDIR /app
RUN pip install uv

FROM base AS deps
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

FROM base AS runtime
COPY --from=deps /app/.venv /app/.venv
COPY src/ src/
COPY alembic/ alembic/
COPY alembic.ini .
ENV PATH="/app/.venv/bin:$PATH"
EXPOSE 8000
CMD ["uvicorn", "ipm.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      IPM_DATABASE_URL: postgresql+asyncpg://ipm:ipm@db:5432/ipm
      IPM_REDIS_URL: redis://redis:6379/0
      IPM_S3_ENDPOINT_URL: http://minio:9000
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ipm
      POSTGRES_PASSWORD: ipm
      POSTGRES_DB: ipm
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ipm"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - miniodata:/data

volumes:
  pgdata:
  miniodata:
```

**Testing**:
- E2E: `docker compose up -d` starts all services without errors
- E2E: `curl http://localhost:8000/api/v1/health` returns `{"status": "healthy"}`
- E2E: `docker compose down` shuts down cleanly

#### 1.6 — Alembic Migration Setup

**What**: Initialise Alembic for async SQLAlchemy migrations.

**Design**:

```python
# alembic/env.py — key configuration
from ipm.database import Base
target_metadata = Base.metadata
# Async migration runner using asyncpg
```

```ini
# alembic.ini
[alembic]
script_location = alembic
sqlalchemy.url = postgresql+asyncpg://ipm:ipm@localhost:5432/ipm
```

**Testing**:
- Integration: `alembic revision --autogenerate -m "initial"` creates a migration file
- Integration: `alembic upgrade head` applies the migration without errors
- Integration: `alembic downgrade -1` reverts cleanly

---

## Phase 2: Core Data Models & Multi-Tenancy

### Purpose
Implement the foundational database tables (tenants, users, parties), authentication/authorization, and tenant isolation. After this phase, the system supports tenant creation, user registration/login with JWT, party (person/organisation) CRUD, and row-level tenant isolation on all queries.

### Tasks

#### 2.1 — Tenant and User Models

**What**: Create SQLAlchemy models for tenants and users, matching the hybrid relational+JSONB schema (Data Model Suggestion 3).

**Design**:

```python
# src/ipm/models/tenant.py
from uuid import uuid4
from sqlalchemy import String, Boolean, CheckConstraint
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import Mapped, mapped_column, relationship
from ipm.database import Base, TimestampMixin

class Tenant(TimestampMixin, Base):
    __tablename__ = "tenants"

    id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    slug: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    tenant_type: Mapped[str] = mapped_column(String(50), nullable=False)
    naic_code: Mapped[str | None] = mapped_column(String(20))
    lei: Mapped[str | None] = mapped_column(String(20))
    jurisdiction_country: Mapped[str] = mapped_column(String(2), nullable=False)  # ISO 3166-1
    regulatory_config: Mapped[dict] = mapped_column(JSONB, default=dict)
    settings: Mapped[dict] = mapped_column(JSONB, default=dict)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)

    users = relationship("User", back_populates="tenant")

    __table_args__ = (
        CheckConstraint(
            "tenant_type IN ('carrier', 'mga', 'broker', 'reinsurer')",
            name="valid_tenant_type",
        ),
    )
```

```python
# src/ipm/models/user.py
class User(TimestampMixin, Base):
    __tablename__ = "users"

    id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    tenant_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("tenants.id"), nullable=False)
    email: Mapped[str] = mapped_column(String(255), nullable=False)
    password_hash: Mapped[str | None] = mapped_column(String(255))
    first_name: Mapped[str] = mapped_column(String(100), nullable=False)
    last_name: Mapped[str] = mapped_column(String(100), nullable=False)
    role: Mapped[str] = mapped_column(String(50), nullable=False)
    permissions: Mapped[list] = mapped_column(JSONB, default=list)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    last_login_at: Mapped[datetime | None] = mapped_column()

    tenant = relationship("Tenant", back_populates="users")

    __table_args__ = (
        UniqueConstraint("tenant_id", "email", name="uq_users_tenant_email"),
        CheckConstraint(
            "role IN ('admin', 'underwriter', 'claims_adjuster', 'agent', 'customer_service', 'billing', 'auditor')",
            name="valid_user_role",
        ),
        Index("ix_users_tenant_role", "tenant_id", "role"),
    )
```

**Testing**:
- Integration: Alembic migration creates `tenants` and `users` tables with all columns and constraints
- Unit: Tenant model instantiation with valid data succeeds
- Unit: Tenant with invalid `tenant_type` raises integrity error on flush
- Unit: User with duplicate email within same tenant raises integrity error
- Unit: User with duplicate email across different tenants succeeds

#### 2.2 — Authentication & JWT

**What**: Implement password hashing, JWT token generation/validation, and authentication dependency.

**Design**:

```python
# src/ipm/services/auth_service.py
from datetime import datetime, timedelta, timezone
from uuid import UUID

from jose import JWTError, jwt
from passlib.context import CryptContext

from ipm.config import get_settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_access_token(user_id: UUID, tenant_id: UUID, role: str) -> str:
    settings = get_settings()
    expire = datetime.now(timezone.utc) + timedelta(minutes=settings.jwt_access_token_expire_minutes)
    payload = {
        "sub": str(user_id),
        "tenant_id": str(tenant_id),
        "role": role,
        "exp": expire,
    }
    return jwt.encode(payload, settings.jwt_secret_key, algorithm=settings.jwt_algorithm)

def decode_access_token(token: str) -> dict:
    settings = get_settings()
    return jwt.decode(token, settings.jwt_secret_key, algorithms=[settings.jwt_algorithm])
```

```python
# src/ipm/schemas/user.py
from pydantic import BaseModel, EmailStr

class UserCreate(BaseModel):
    email: EmailStr
    password: str
    first_name: str
    last_name: str
    role: str

class UserLogin(BaseModel):
    email: EmailStr
    password: str

class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
    expires_in: int

class UserResponse(BaseModel):
    id: UUID
    tenant_id: UUID
    email: str
    first_name: str
    last_name: str
    role: str
    is_active: bool
```

```python
# src/ipm/dependencies.py
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from sqlalchemy.ext.asyncio import AsyncSession

from ipm.database import get_db
from ipm.services.auth_service import decode_access_token

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db),
) -> User:
    try:
        payload = decode_access_token(credentials.credentials)
    except Exception:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")
    user = await db.get(User, UUID(payload["sub"]))
    if not user or not user.is_active:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="User not found or inactive")
    return user

def require_role(*roles: str):
    def checker(user: User = Depends(get_current_user)):
        if user.role not in roles:
            raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="Insufficient permissions")
        return user
    return checker
```

**Testing**:
- Unit: `hash_password("secret")` returns a bcrypt hash that `verify_password` accepts
- Unit: `create_access_token(...)` returns a decodable JWT with correct claims
- Unit: expired token raises `JWTError` on decode
- Integration: `POST /api/v1/auth/login` with valid credentials returns 200 + token
- Integration: `POST /api/v1/auth/login` with wrong password returns 401
- Integration: request with valid Bearer token → `get_current_user` returns the user
- Integration: request with expired token → 401
- Integration: `require_role("admin")` blocks a user with role "agent" → 403

#### 2.3 — Tenant Isolation Middleware

**What**: Middleware that extracts `tenant_id` from the JWT token and injects it into the request state so all database queries are scoped by tenant.

**Design**:

```python
# src/ipm/middleware/tenant.py
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

class TenantMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # tenant_id is extracted from JWT in the auth dependency
        # This middleware ensures it's available on request.state
        response = await call_next(request)
        return response

# Helper for service layer — all queries filter by tenant
def tenant_filter(query, model, tenant_id: UUID):
    """Add tenant_id filter to any SQLAlchemy query."""
    return query.where(model.tenant_id == tenant_id)
```

**Testing**:
- Integration: API request with Tenant A's token cannot read Tenant B's data → empty result
- Integration: API request without token → 401 on protected endpoints
- Unit: `tenant_filter` appends correct WHERE clause to query

#### 2.4 — Party (Person/Organisation) CRUD

**What**: Full CRUD API for parties (policyholders, agents, third parties) using the unified party model with JSONB details and embedded addresses.

**Design**:

```python
# src/ipm/schemas/party.py
from pydantic import BaseModel
from typing import Any
from uuid import UUID

class AddressSchema(BaseModel):
    type: str  # "mailing", "billing", "property", "business"
    line1: str
    line2: str | None = None
    city: str
    state: str | None = None
    zip: str | None = None
    country: str = "US"
    is_primary: bool = False

class PartyCreate(BaseModel):
    party_type: str  # "individual" or "organisation"
    first_name: str | None = None
    last_name: str | None = None
    organisation_name: str | None = None
    email: str | None = None
    phone: str | None = None
    date_of_birth: date | None = None
    tax_id_last4: str | None = None
    details: dict[str, Any] = {}
    addresses: list[AddressSchema] = []

class PartyResponse(BaseModel):
    id: UUID
    tenant_id: UUID
    party_type: str
    display_name: str
    email: str | None
    phone: str | None
    first_name: str | None
    last_name: str | None
    organisation_name: str | None
    details: dict[str, Any]
    addresses: list[AddressSchema]
    created_at: datetime
    updated_at: datetime

class PartyListResponse(BaseModel):
    items: list[PartyResponse]
    total: int
    page: int
    page_size: int
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/parties` | Create a party |
| GET | `/api/v1/parties` | List parties (paginated, filterable by type/name/email) |
| GET | `/api/v1/parties/{id}` | Get party by ID |
| PUT | `/api/v1/parties/{id}` | Update party |
| DELETE | `/api/v1/parties/{id}` | Soft-delete party |

**Testing**:
- Integration: `POST /api/v1/parties` with individual data → 201, party created with computed `display_name`
- Integration: `POST /api/v1/parties` with organisation data → 201, `display_name` = organisation_name
- Integration: `GET /api/v1/parties?party_type=individual` → only individuals returned
- Integration: `GET /api/v1/parties?search=smith` → name search works
- Integration: `PUT /api/v1/parties/{id}` updates address in JSONB array
- Integration: tenant isolation — Tenant A cannot see Tenant B's parties
- Unit: `PartyCreate` validation rejects individual without first_name and last_name
- Unit: `PartyCreate` validation rejects organisation without organisation_name

#### 2.5 — Tenant and User API Endpoints

**What**: API endpoints for tenant setup and user management.

**Design**:

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| POST | `/api/v1/tenants` | Create tenant (system admin only) | System key |
| GET | `/api/v1/tenants/{slug}` | Get tenant info | Authenticated |
| POST | `/api/v1/auth/register` | Register user within a tenant | None (requires tenant slug) |
| POST | `/api/v1/auth/login` | Login → JWT | None |
| GET | `/api/v1/users/me` | Current user profile | Authenticated |
| GET | `/api/v1/users` | List users in tenant | Admin |
| PUT | `/api/v1/users/{id}` | Update user | Admin or self |

**Testing**:
- Integration: `POST /api/v1/tenants` with valid data → tenant created, slug unique
- Integration: `POST /api/v1/auth/register` → user created, password hashed, 201 returned
- Integration: `POST /api/v1/auth/login` → valid token returned
- Integration: `GET /api/v1/users/me` with valid token → user profile returned
- Integration: non-admin user cannot `GET /api/v1/users` → 403
- Integration: admin user can list all users in their tenant

#### 2.6 — Audit Logging

**What**: Create the audit_log table and an automated audit logging mechanism that captures entity changes.

**Design**:

```python
# src/ipm/models/audit.py
class AuditLog(Base):
    __tablename__ = "audit_log"

    id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    tenant_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), nullable=False)
    user_id: Mapped[uuid4 | None] = mapped_column(UUID(as_uuid=True))
    entity_type: Mapped[str] = mapped_column(String(50), nullable=False)
    entity_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), nullable=False)
    action: Mapped[str] = mapped_column(String(30), nullable=False)
    changes: Mapped[dict | None] = mapped_column(JSONB)
    ip_address: Mapped[str | None] = mapped_column(String(45))
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

```python
# src/ipm/services/audit_service.py
async def log_audit(
    db: AsyncSession,
    tenant_id: UUID,
    user_id: UUID | None,
    entity_type: str,
    entity_id: UUID,
    action: str,
    changes: dict | None = None,
    ip_address: str | None = None,
) -> None:
    entry = AuditLog(
        tenant_id=tenant_id, user_id=user_id, entity_type=entity_type,
        entity_id=entity_id, action=action, changes=changes, ip_address=ip_address,
    )
    db.add(entry)
```

**Testing**:
- Integration: creating a party also creates an audit_log entry with action="create"
- Integration: updating a party creates an audit_log entry with old/new field values in `changes` JSONB
- Unit: `log_audit` creates an AuditLog instance with all fields populated
- Integration: audit log entries are tenant-scoped (cannot query across tenants)

---

## Phase 3: Product Configuration & Rating Engine

### Purpose
Implement the insurance product catalogue, coverage definitions, rating tables, and the rating engine that calculates premiums. After this phase, an admin can define insurance products (e.g., HO-3 Homeowners), configure coverages with limits and deductibles, load rating factor tables, and the system can calculate premiums for a given risk profile.

### Tasks

#### 3.1 — Product Model and CRUD

**What**: Create the product model with embedded coverage definitions and underwriting questions in JSONB. CRUD API for product management.

**Design**:

```python
# src/ipm/models/product.py
class Product(TimestampMixin, Base):
    __tablename__ = "products"

    id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    tenant_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("tenants.id"), nullable=False)
    product_code: Mapped[str] = mapped_column(String(50), nullable=False)
    product_name: Mapped[str] = mapped_column(String(255), nullable=False)
    line_of_business: Mapped[str] = mapped_column(String(50), nullable=False)
    iso_class_code: Mapped[str | None] = mapped_column(String(20))
    policy_schema: Mapped[dict] = mapped_column(JSONB, default=dict)       # JSON Schema for policy details
    item_schema: Mapped[dict] = mapped_column(JSONB, default=dict)         # JSON Schema for insured item details
    rating_config: Mapped[dict] = mapped_column(JSONB, default=dict)       # Rating engine configuration
    coverage_definitions: Mapped[list] = mapped_column(JSONB, default=list) # Coverage type definitions
    underwriting_questions: Mapped[list] = mapped_column(JSONB, default=list)
    effective_date: Mapped[date] = mapped_column(nullable=False)
    expiration_date: Mapped[date | None] = mapped_column()
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)

    __table_args__ = (
        UniqueConstraint("tenant_id", "product_code", name="uq_products_tenant_code"),
    )
```

```python
# src/ipm/schemas/product.py
class CoverageDefinition(BaseModel):
    code: str
    name: str
    mandatory: bool = False
    default_limit: float | None = None
    min_limit: float | None = None
    max_limit: float | None = None
    default_deductible: float | None = None
    default_pct_of_a: float | None = None  # For coverages calculated as % of Coverage A

class UnderwritingQuestion(BaseModel):
    code: str
    text: str
    type: str  # "yes_no", "text", "number", "date", "select"
    required: bool = True
    options: list[str] | None = None

class ProductCreate(BaseModel):
    product_code: str
    product_name: str
    line_of_business: str
    iso_class_code: str | None = None
    coverage_definitions: list[CoverageDefinition]
    underwriting_questions: list[UnderwritingQuestion] = []
    rating_config: dict = {}
    effective_date: date
    expiration_date: date | None = None
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/products` | Create product (admin) |
| GET | `/api/v1/products` | List products (filterable by LOB, active status) |
| GET | `/api/v1/products/{id}` | Get product with coverage definitions |
| PUT | `/api/v1/products/{id}` | Update product |
| PATCH | `/api/v1/products/{id}/coverages` | Update coverage definitions |

**Testing**:
- Integration: create HO-3 homeowners product with 4 coverage definitions → 201
- Integration: create commercial auto product with different coverage set → 201
- Integration: GET product returns coverage_definitions as structured array
- Unit: `CoverageDefinition` validation rejects negative limit values
- Unit: `ProductCreate` requires at least one coverage definition
- Integration: products are tenant-isolated

#### 3.2 — Rating Tables and Factor Management

**What**: Create rating tables with JSONB-stored rate data. API for uploading, versioning, and querying rating factors.

**Design**:

```python
# src/ipm/models/product.py (continued)
class RatingTable(TimestampMixin, Base):
    __tablename__ = "rating_tables"

    id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    tenant_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("tenants.id"), nullable=False)
    product_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("products.id"), nullable=False)
    table_name: Mapped[str] = mapped_column(String(255), nullable=False)
    jurisdiction: Mapped[str | None] = mapped_column(String(2))  # NULL = all jurisdictions
    effective_date: Mapped[date] = mapped_column(nullable=False)
    expiration_date: Mapped[date | None] = mapped_column()
    version: Mapped[int] = mapped_column(default=1)
    rate_data: Mapped[dict] = mapped_column(JSONB, nullable=False)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
```

Rate data structures (stored in `rate_data` JSONB):

```python
# Lookup factor (territory, protection class)
{"factor_type": "lookup", "rates": {"001": 0.85, "002": 0.90, "003": 1.00}}

# Banded factor (age of home, credit score)
{"factor_type": "banded", "rates": [
    {"min": 0, "max": 5, "factor": 0.90},
    {"min": 6, "max": 15, "factor": 1.00},
    {"min": 16, "max": 30, "factor": 1.10},
    {"min": 31, "max": null, "factor": 1.25}
]}

# Base rate table
{"factor_type": "base_rate", "rates": {
    "COV_A": {"per_thousand": 2.50},
    "COV_B": {"pct_of_a": 0.10, "per_thousand": 2.50},
    "COV_E": {"flat": 85.00}
}}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/products/{id}/rating-tables` | Upload rating table |
| GET | `/api/v1/products/{id}/rating-tables` | List rating tables for product |
| GET | `/api/v1/rating-tables/{id}` | Get rating table detail |
| PUT | `/api/v1/rating-tables/{id}` | Update (creates new version) |

**Testing**:
- Integration: upload territory factor table → stored in JSONB, version 1
- Integration: update table → new version created, old version deactivated
- Integration: query for effective table on a specific date → correct version returned
- Unit: lookup factor retrieval for "Territory 005" → returns 1.30
- Unit: banded factor retrieval for age 22 → returns 1.10 (falls in 16-30 band)
- Unit: banded factor retrieval for age 0 → returns 0.90

#### 3.3 — Rating Engine

**What**: A service that takes risk characteristics and calculates premium by applying base rates and multiplicative factors from rating tables.

**Design**:

```python
# src/ipm/services/rating_engine.py
from dataclasses import dataclass
from decimal import Decimal
from uuid import UUID

@dataclass
class RatingInput:
    product_id: UUID
    jurisdiction: str
    effective_date: date
    coverage_selections: list[dict]  # [{"code": "COV_A", "limit": 500000, "deductible": 2500}]
    risk_characteristics: dict       # {"territory": "005", "protection_class": "3", "year_built": 1985, ...}

@dataclass
class RatingResult:
    coverage_premiums: list[dict]    # [{"code": "COV_A", "premium": 1800.00, "factors_applied": [...]}]
    total_premium: Decimal
    factors_applied: list[dict]      # [{"factor": "territory", "value": "005", "multiplier": 1.30}]
    rating_date: date

class RatingEngine:
    def __init__(self, db: AsyncSession):
        self.db = db

    async def calculate_premium(self, input: RatingInput) -> RatingResult:
        """
        Algorithm:
        1. Load product's rating_config to get factor sequence and base rate table name
        2. Load base rate table for the product + jurisdiction + effective date
        3. For each coverage:
           a. Calculate base premium (per_thousand * limit/1000, or flat, or pct_of_a)
           b. Load each rating factor table in sequence
           c. Look up the factor value from risk_characteristics
           d. Apply multiplicative factor
        4. Apply minimum premium floor from rating_config
        5. Round per rating_config.rounding
        6. Sum coverage premiums for total
        """
        ...

    async def _load_effective_table(
        self, product_id: UUID, table_name: str, jurisdiction: str, effective_date: date
    ) -> dict:
        """Load the most recent active rating table for the given parameters."""
        ...

    def _lookup_factor(self, rate_data: dict, value: str | int | float) -> Decimal:
        """Look up a factor from a rating table's rate_data based on factor_type."""
        if rate_data["factor_type"] == "lookup":
            return Decimal(str(rate_data["rates"].get(str(value), 1.0)))
        elif rate_data["factor_type"] == "banded":
            for band in rate_data["rates"]:
                if band["min"] <= value and (band["max"] is None or value <= band["max"]):
                    return Decimal(str(band["factor"]))
            return Decimal("1.0")
        ...
```

**Testing**:
- Unit: `_lookup_factor` with lookup table and valid key → correct factor
- Unit: `_lookup_factor` with banded table and value at band boundary → correct factor
- Unit: `_lookup_factor` with value exceeding all bands → default 1.0
- Integration (mocked DB): `calculate_premium` for HO-3 with territory 005 (1.30), protection class 3 (1.00) → correct premium
- Integration (mocked DB): `calculate_premium` with minimum premium floor → floor applied when calculated premium is below it
- Integration (mocked DB): `calculate_premium` with multiple coverages → each coverage premium calculated independently, total is sum
- Unit: rounding to nearest dollar works correctly for .50 edge case
- Fixture-based: load `tests/fixtures/rating_tables/ho3_territory.json`, run rating, verify output

---

## Phase 4: Policy Lifecycle Management

### Purpose
Implement the complete policy lifecycle: quoting, application, binding, issuance, renewal, and cancellation. This is the core value proposition of the platform. After this phase, agents can create quotes, bind policies, process endorsements, and manage renewals through the API.

### Tasks

#### 4.1 — Policy Model and Schema

**What**: Create the policy SQLAlchemy model with JSONB coverages, details, and interests. Pydantic schemas for all policy lifecycle states.

**Design**:

```python
# src/ipm/models/policy.py
class Policy(TimestampMixin, Base):
    __tablename__ = "policies"

    id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    tenant_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("tenants.id"), nullable=False)
    policy_number: Mapped[str] = mapped_column(String(50), nullable=False)
    product_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("products.id"), nullable=False)
    policyholder_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("parties.id"), nullable=False)
    agent_id: Mapped[uuid4 | None] = mapped_column(UUID(as_uuid=True), ForeignKey("parties.id"))
    status: Mapped[str] = mapped_column(String(30), nullable=False)
    effective_date: Mapped[date] = mapped_column(nullable=False)
    expiration_date: Mapped[date] = mapped_column(nullable=False)
    original_inception_date: Mapped[date] = mapped_column(nullable=False)
    term_months: Mapped[int] = mapped_column(default=12)
    jurisdiction: Mapped[str] = mapped_column(String(2), nullable=False)
    total_premium: Mapped[Decimal | None] = mapped_column(Numeric(15, 2))
    currency: Mapped[str] = mapped_column(String(3), default="USD")
    prior_policy_id: Mapped[uuid4 | None] = mapped_column(UUID(as_uuid=True), ForeignKey("policies.id"))
    underwriter_id: Mapped[uuid4 | None] = mapped_column(UUID(as_uuid=True), ForeignKey("users.id"))
    coverages: Mapped[list] = mapped_column(JSONB, default=list)
    details: Mapped[dict] = mapped_column(JSONB, default=dict)
    interests: Mapped[list] = mapped_column(JSONB, default=list)
    bound_at: Mapped[datetime | None] = mapped_column()
    issued_at: Mapped[datetime | None] = mapped_column()
    cancellation_date: Mapped[date | None] = mapped_column()
    cancellation_reason: Mapped[str | None] = mapped_column(String(100))

    __table_args__ = (
        UniqueConstraint("tenant_id", "policy_number", name="uq_policies_tenant_number"),
        CheckConstraint(
            "status IN ('quote','application','bound','issued','active','pending_renewal','renewed','cancelled','expired','non_renewed')",
            name="valid_policy_status",
        ),
    )
```

Policy status state machine:
```
quote → application → bound → issued → active → pending_renewal → renewed
                                    ↘ cancelled
                                    ↘ expired
                                    ↘ non_renewed
```

**Testing**:
- Integration: migration creates policies table with all indexes and constraints
- Unit: Policy with invalid status value raises integrity error
- Unit: policy_number uniqueness enforced within tenant

#### 4.2 — Policy Number Generation

**What**: Generate unique, formatted policy numbers per product line and tenant.

**Design**:

```python
# src/ipm/utils/policy_number.py
import datetime
from uuid import UUID

async def generate_policy_number(
    db: AsyncSession, tenant_id: UUID, product_code: str
) -> str:
    """
    Format: {product_code}-{year}-{sequence:06d}
    Example: HO3-2026-000001

    Uses a database sequence per tenant+product to guarantee uniqueness
    under concurrent access.
    """
    year = datetime.date.today().year
    # Atomic sequence increment using PostgreSQL advisory lock
    result = await db.execute(
        text("""
            SELECT nextval(
                pg_catalog.format('policy_seq_%s_%s_%s', :tenant_id, :product_code, :year)
            )
        """),
        {"tenant_id": str(tenant_id).replace("-", ""), "product_code": product_code, "year": year},
    )
    seq = result.scalar()
    return f"{product_code}-{year}-{seq:06d}"
```

**Testing**:
- Integration: two concurrent calls generate different sequence numbers
- Unit: format matches expected pattern "HO3-2026-000001"
- Integration: sequence resets per year (2026 vs 2027)

#### 4.3 — Quoting Service

**What**: Create a quote by selecting a product, entering risk characteristics, selecting coverages, and running the rating engine.

**Design**:

```python
# src/ipm/schemas/policy.py
class QuoteRequest(BaseModel):
    product_id: UUID
    policyholder_id: UUID
    agent_id: UUID | None = None
    effective_date: date
    term_months: int = 12
    jurisdiction: str  # ISO 3166-2 state code
    coverage_selections: list[CoverageSelection]
    risk_characteristics: dict
    details: dict = {}
    underwriting_answers: dict = {}

class CoverageSelection(BaseModel):
    code: str
    limit: float
    deductible: float = 0

class QuoteResponse(BaseModel):
    id: UUID
    policy_number: str
    status: str  # "quote"
    total_premium: float
    coverages: list[dict]
    effective_date: date
    expiration_date: date
```

```python
# src/ipm/services/policy_service.py
class PolicyService:
    async def create_quote(self, db: AsyncSession, tenant_id: UUID, request: QuoteRequest) -> Policy:
        """
        1. Validate product exists and is active
        2. Validate policyholder exists
        3. Validate coverage selections against product's coverage_definitions
        4. Run rating engine to calculate premiums
        5. Generate policy number
        6. Create policy with status='quote'
        7. Create audit log entry
        8. Return policy with calculated premiums
        """
        ...
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/policies/quote` | Create quote |
| GET | `/api/v1/policies` | List policies (filterable by status, policyholder, agent, date range) |
| GET | `/api/v1/policies/{id}` | Get policy detail |
| GET | `/api/v1/policies/number/{policy_number}` | Get policy by number |

**Testing**:
- Integration: create quote with valid data → 201, status="quote", premium calculated
- Integration: create quote with invalid product → 404
- Integration: create quote with coverage limit above product max → 400
- Integration: create quote for inactive product → 400
- Integration: list policies with status filter → correct results
- Unit: coverage selection validation against product definitions
- Fixture-based: create quote with fixture rating tables, verify premium matches expected value

#### 4.4 — Binding and Issuance

**What**: Transition a quote through application → bound → issued → active. Each transition has business rules and audit logging.

**Design**:

```python
# src/ipm/services/policy_service.py (continued)
class PolicyService:
    VALID_TRANSITIONS = {
        "quote": ["application", "bound"],      # Small risks can skip application
        "application": ["bound", "quote"],       # Can return to quote if changes needed
        "bound": ["issued"],
        "issued": ["active"],
        "active": ["pending_renewal", "cancelled", "expired"],
        "pending_renewal": ["renewed", "non_renewed", "cancelled"],
    }

    async def bind_policy(self, db: AsyncSession, policy_id: UUID, user: User) -> Policy:
        """
        1. Validate current status allows transition to 'bound'
        2. Validate underwriting approval (if required by product config)
        3. Set status='bound', bound_at=now()
        4. Audit log with action='approve'
        """
        ...

    async def issue_policy(self, db: AsyncSession, policy_id: UUID, user: User) -> Policy:
        """
        1. Validate status is 'bound'
        2. Set status='issued', issued_at=now()
        3. Generate declaration page document (async task)
        4. Create billing account and installment schedule
        5. Audit log
        """
        ...

    async def activate_policy(self, db: AsyncSession, policy_id: UUID) -> Policy:
        """
        1. Validate status is 'issued' and effective_date <= today
        2. Set status='active'
        3. Audit log
        """
        ...

    async def cancel_policy(
        self, db: AsyncSession, policy_id: UUID, cancellation_date: date,
        reason: str, user: User
    ) -> Policy:
        """
        1. Validate policy is active or pending_renewal
        2. Calculate earned/unearned premium split
        3. Set status='cancelled', cancellation_date, cancellation_reason
        4. Create refund invoice if unearned premium > 0
        5. Audit log
        """
        ...
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/policies/{id}/bind` | Bind a quoted/applied policy |
| POST | `/api/v1/policies/{id}/issue` | Issue a bound policy |
| POST | `/api/v1/policies/{id}/activate` | Activate an issued policy |
| POST | `/api/v1/policies/{id}/cancel` | Cancel a policy |

**Testing**:
- Integration: quote → bind → issue → activate lifecycle completes successfully
- Integration: attempt to bind an already-issued policy → 400
- Integration: bind without underwriting approval (when required) → 400
- Integration: cancel with pro-rata refund → correct unearned premium calculated
- Integration: each transition creates an audit log entry
- Unit: `VALID_TRANSITIONS` map rejects invalid status transitions
- Unit: earned premium calculation for mid-term cancellation (180 days into 365-day term)

#### 4.5 — Insured Items

**What**: CRUD for insured items (vehicles, properties, equipment) attached to policies, with type-specific details in JSONB.

**Design**:

```python
# src/ipm/models/insured_item.py
class InsuredItem(TimestampMixin, Base):
    __tablename__ = "insured_items"

    id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    policy_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("policies.id"), nullable=False)
    item_type: Mapped[str] = mapped_column(String(30), nullable=False)
    item_number: Mapped[int] = mapped_column(nullable=False)
    description: Mapped[str] = mapped_column(Text, nullable=False)
    coverages: Mapped[list] = mapped_column(JSONB, default=list)
    details: Mapped[dict] = mapped_column(JSONB, default=dict)

class Driver(TimestampMixin, Base):
    __tablename__ = "drivers"
    # ... (per Data Model Suggestion 3)
```

**Testing**:
- Integration: add vehicle to auto policy with VIN, make, model in details → 201
- Integration: add property to homeowners policy with construction type, year_built in details → 201
- Integration: validate item details against product's `item_schema` JSON Schema → 400 on invalid
- Integration: list insured items for a policy → returns items with details
- Unit: item_number auto-increments per policy

#### 4.6 — Policy Renewal

**What**: Renewal workflow that creates a new policy linked to the expiring one, with updated premiums.

**Design**:

```python
# src/ipm/services/policy_service.py (continued)
class PolicyService:
    async def create_renewal(self, db: AsyncSession, policy_id: UUID, user: User) -> Policy:
        """
        1. Load expiring policy and validate status is 'active' or 'pending_renewal'
        2. Copy policy data to new policy with:
           - New policy_number (same product code, new sequence)
           - effective_date = expiring policy's expiration_date
           - expiration_date = effective_date + term_months
           - prior_policy_id = expiring policy's id
           - status = 'quote' (renewal quote)
           - original_inception_date preserved from original
        3. Re-rate with current rating tables
        4. Copy insured items and drivers
        5. Set expiring policy status to 'pending_renewal'
        6. Audit log on both policies
        """
        ...
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/policies/{id}/renew` | Create renewal quote for an expiring policy |

**Testing**:
- Integration: renew an active policy → new quote created with correct dates and prior_policy_id linkage
- Integration: renewal premium may differ from original (re-rated with current tables)
- Integration: cannot renew a cancelled policy → 400
- Integration: renewal chain preserves original_inception_date
- Unit: expiration_date of new policy = effective_date + term_months

---

## Phase 5: Endorsements & Underwriting

### Purpose
Implement mid-term policy modifications (endorsements) with approval workflows, and the underwriting submission and review process. After this phase, users can request policy changes, route them for underwriting approval, and track the complete endorsement audit trail.

### Tasks

#### 5.1 — Endorsement Model and Service

**What**: Create endorsements that capture policy changes as JSONB diffs with an approval workflow.

**Design**:

```python
# src/ipm/models/endorsement.py
class Endorsement(TimestampMixin, Base):
    __tablename__ = "endorsements"

    id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    policy_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("policies.id"), nullable=False)
    endorsement_number: Mapped[int] = mapped_column(nullable=False)
    endorsement_type: Mapped[str] = mapped_column(String(50), nullable=False)
    description: Mapped[str] = mapped_column(Text, nullable=False)
    effective_date: Mapped[date] = mapped_column(nullable=False)
    premium_change: Mapped[Decimal] = mapped_column(Numeric(15, 2), default=0)
    status: Mapped[str] = mapped_column(String(20), nullable=False)
    changes: Mapped[dict] = mapped_column(JSONB, nullable=False)
    approved_by: Mapped[uuid4 | None] = mapped_column(UUID(as_uuid=True), ForeignKey("users.id"))
    approved_at: Mapped[datetime | None] = mapped_column()

    __table_args__ = (
        UniqueConstraint("policy_id", "endorsement_number", name="uq_endorsements_policy_number"),
    )
```

Endorsement status state machine:
```
draft → pending_approval → approved → issued
                         ↘ rejected
```

```python
# src/ipm/services/endorsement_service.py
class EndorsementService:
    async def create_endorsement(
        self, db: AsyncSession, policy_id: UUID, request: EndorsementRequest, user: User
    ) -> Endorsement:
        """
        1. Validate policy is active
        2. Auto-increment endorsement_number for this policy
        3. Validate proposed changes (coverage limits within bounds, etc.)
        4. Calculate premium_change by re-rating with new values vs current
        5. Store changes as JSONB diff: {"coverages": {"COV_A": {"limit": {"old": 500000, "new": 600000}}}}
        6. Set status = 'draft' or 'pending_approval' based on premium_change threshold
        """
        ...

    async def approve_endorsement(self, db: AsyncSession, endorsement_id: UUID, user: User) -> Endorsement:
        """
        1. Validate user has underwriter role
        2. Set status='approved', approved_by, approved_at
        """
        ...

    async def issue_endorsement(self, db: AsyncSession, endorsement_id: UUID, user: User) -> Endorsement:
        """
        1. Validate status is 'approved'
        2. Apply changes to the policy's coverages/details JSONB
        3. Update policy.total_premium
        4. Set endorsement status='issued'
        5. Create billing adjustment if premium changed
        6. Audit log
        """
        ...
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/policies/{id}/endorsements` | Create endorsement |
| GET | `/api/v1/policies/{id}/endorsements` | List endorsements for policy |
| GET | `/api/v1/endorsements/{id}` | Get endorsement detail |
| POST | `/api/v1/endorsements/{id}/approve` | Approve endorsement |
| POST | `/api/v1/endorsements/{id}/reject` | Reject endorsement |
| POST | `/api/v1/endorsements/{id}/issue` | Issue approved endorsement |

**Testing**:
- Integration: create endorsement increasing Coverage A limit → premium_change calculated
- Integration: endorsement with premium_change > $100 → auto-set to pending_approval
- Integration: underwriter approves → status transitions correctly
- Integration: issue endorsement → policy.coverages updated, total_premium recalculated
- Integration: endorsement changes JSONB shows old/new values
- Integration: non-underwriter cannot approve → 403
- Integration: cannot endorse a cancelled policy → 400
- Unit: endorsement_number increments correctly (1, 2, 3...)
- Unit: premium change calculation matches expected value

#### 5.2 — Underwriting Submissions

**What**: Underwriting review workflow for new business and referrals, with risk scoring and assignment.

**Design**:

```python
# src/ipm/services/underwriting_service.py
class UnderwritingService:
    async def create_submission(
        self, db: AsyncSession, policy_id: UUID, submission_type: str
    ) -> dict:
        """
        1. Create submission record with status='submitted'
        2. Auto-assign to underwriter based on product/jurisdiction rules
        3. Run initial risk score (simple rule-based for MVP, AI in Phase 9)
        4. If risk_score < 50: auto-approve (straight-through processing)
        5. If risk_score >= 50 and < 80: assign for review
        6. If risk_score >= 80: auto-refer to senior underwriter
        """
        ...

    async def review_submission(
        self, db: AsyncSession, submission_id: UUID, decision: str, notes: str, user: User
    ) -> dict:
        """
        Decision: 'approved', 'declined', 'referred', 'info_requested'
        If approved: bind the policy
        If declined: set policy status back to 'quote'
        """
        ...
```

The underwriting submission is stored as a JSONB field on the policy's `details` to avoid adding another table in the MVP:

```python
# Policy details JSONB includes:
{
    "underwriting": {
        "submission_type": "new_business",
        "status": "approved",
        "assigned_to": "uuid-...",
        "risk_score": 35.5,
        "risk_tier": "preferred",
        "notes": "Clean loss history, protection class 3",
        "submitted_at": "2026-07-01T10:00:00Z",
        "decided_at": "2026-07-01T10:05:00Z",
        "decided_by": "uuid-..."
    }
}
```

**Testing**:
- Integration: submit new business with low risk → auto-approved, policy bound
- Integration: submit with moderate risk → assigned to underwriter, status='in_review'
- Integration: underwriter approves → policy status transitions to 'bound'
- Integration: underwriter declines → policy status returns to 'quote'
- Integration: underwriter requests info → status='info_requested'
- Unit: risk scoring rules produce expected tier for given inputs

---

## Phase 6: Claims Management

### Purpose
Implement First Notice of Loss (FNOL), claims lifecycle management, reserve and payment tracking, and adjuster assignment. After this phase, claims can be filed against active policies, assigned to adjusters, reserved, paid, and closed.

### Tasks

#### 6.1 — Claims Model and FNOL

**What**: Create the claims model with loss-type-specific details in JSONB. FNOL intake endpoint.

**Design**:

```python
# src/ipm/models/claim.py
class Claim(TimestampMixin, Base):
    __tablename__ = "claims"

    id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    tenant_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("tenants.id"), nullable=False)
    claim_number: Mapped[str] = mapped_column(String(50), nullable=False)
    policy_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("policies.id"), nullable=False)
    claimant_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("parties.id"), nullable=False)
    status: Mapped[str] = mapped_column(String(30), nullable=False)
    loss_date: Mapped[date] = mapped_column(nullable=False)
    reported_date: Mapped[date] = mapped_column(nullable=False)
    loss_type: Mapped[str] = mapped_column(String(50), nullable=False)
    loss_description: Mapped[str | None] = mapped_column(Text)
    assigned_adjuster_id: Mapped[uuid4 | None] = mapped_column(UUID(as_uuid=True), ForeignKey("users.id"))
    total_incurred: Mapped[Decimal] = mapped_column(Numeric(15, 2), default=0)
    total_paid: Mapped[Decimal] = mapped_column(Numeric(15, 2), default=0)
    total_reserved: Mapped[Decimal] = mapped_column(Numeric(15, 2), default=0)
    fraud_score: Mapped[Decimal | None] = mapped_column(Numeric(5, 2))
    coverage_breakdown: Mapped[list] = mapped_column(JSONB, default=list)
    details: Mapped[dict] = mapped_column(JSONB, default=dict)
    closed_at: Mapped[datetime | None] = mapped_column()

    __table_args__ = (
        UniqueConstraint("tenant_id", "claim_number", name="uq_claims_tenant_number"),
    )
```

Claims status state machine:
```
fnol → open → under_investigation → reserved → approved → closed
                                              ↘ denied → closed
                                   → subrogation → closed
                                   → reopened → open
```

```python
# src/ipm/services/claims_service.py
class ClaimsService:
    async def file_claim(self, db: AsyncSession, tenant_id: UUID, request: FNOLRequest) -> Claim:
        """
        1. Validate policy is active and loss_date is within policy period
        2. Validate claimant is a party in the system
        3. Generate claim number: CLM-{year}-{sequence:06d}
        4. Match loss to applicable coverages from policy.coverages
        5. Create claim with status='fnol'
        6. Auto-assign adjuster based on loss_type and jurisdiction
        7. Create initial claim activity (FNOL received)
        8. Audit log
        """
        ...
```

```python
# src/ipm/schemas/claim.py
class FNOLRequest(BaseModel):
    policy_id: UUID
    claimant_id: UUID
    loss_date: date
    loss_type: str
    loss_description: str
    details: dict = {}

class ClaimResponse(BaseModel):
    id: UUID
    claim_number: str
    policy_id: UUID
    status: str
    loss_date: date
    loss_type: str
    total_incurred: float
    total_paid: float
    total_reserved: float
    coverage_breakdown: list[dict]
    details: dict
    assigned_adjuster_id: UUID | None
    created_at: datetime
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/claims` | File FNOL |
| GET | `/api/v1/claims` | List claims (filterable by status, adjuster, policy, date range) |
| GET | `/api/v1/claims/{id}` | Get claim detail |
| GET | `/api/v1/policies/{id}/claims` | List claims for a policy |

**Testing**:
- Integration: file FNOL against active policy → claim created with status='fnol'
- Integration: file FNOL with loss_date outside policy period → 400
- Integration: file FNOL against cancelled policy → 400
- Integration: adjuster auto-assignment works based on loss_type
- Integration: coverage_breakdown populated from policy coverages
- Unit: claim number format CLM-2026-000001

#### 6.2 — Claims Lifecycle Transitions

**What**: Status transitions, adjuster reassignment, and claim closure.

**Design**:

```python
# src/ipm/services/claims_service.py (continued)
class ClaimsService:
    VALID_TRANSITIONS = {
        "fnol": ["open"],
        "open": ["under_investigation", "reserved", "closed", "denied"],
        "under_investigation": ["reserved", "open", "denied", "closed"],
        "reserved": ["approved", "denied", "subrogation", "closed"],
        "approved": ["closed"],
        "denied": ["closed", "reopened"],
        "subrogation": ["closed"],
        "reopened": ["open"],
        "closed": ["reopened"],
    }

    async def transition_claim(
        self, db: AsyncSession, claim_id: UUID, new_status: str, user: User, notes: str = ""
    ) -> Claim:
        ...

    async def assign_adjuster(
        self, db: AsyncSession, claim_id: UUID, adjuster_id: UUID, user: User
    ) -> Claim:
        ...

    async def close_claim(self, db: AsyncSession, claim_id: UUID, user: User) -> Claim:
        """Validate all reserves are zero or paid. Set status='closed', closed_at=now()."""
        ...
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/claims/{id}/transition` | Transition claim status |
| POST | `/api/v1/claims/{id}/assign` | Assign/reassign adjuster |
| POST | `/api/v1/claims/{id}/close` | Close claim |
| POST | `/api/v1/claims/{id}/reopen` | Reopen a closed claim |

**Testing**:
- Integration: fnol → open → reserved → approved → closed lifecycle
- Integration: invalid transition (fnol → closed directly) → 400
- Integration: close claim with outstanding reserves → 400
- Integration: reopen a closed claim → status returns to 'open'
- Integration: reassign adjuster creates audit log entry

#### 6.3 — Claim Reserves and Payments

**What**: Set reserves on claim coverages, process claim payments (indemnity, expense), and track totals.

**Design**:

```python
class ClaimsService:
    async def set_reserve(
        self, db: AsyncSession, claim_id: UUID, coverage_code: str, amount: Decimal, user: User
    ) -> Claim:
        """
        1. Find coverage in coverage_breakdown JSONB array
        2. Update reserve amount
        3. Recalculate total_reserved and total_incurred
        4. Create claim_activity with type='reserve_change'
        """
        ...

    async def process_payment(
        self, db: AsyncSession, claim_id: UUID, request: ClaimPaymentRequest, user: User
    ) -> Claim:
        """
        1. Validate payment does not exceed reserve for the coverage
        2. Add payment to coverage_breakdown
        3. Update total_paid, recalculate total_incurred
        4. Create claim_activity with type='payment'
        """
        ...
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/claims/{id}/reserves` | Set/update reserve for a coverage |
| POST | `/api/v1/claims/{id}/payments` | Process a claim payment |
| GET | `/api/v1/claims/{id}/activities` | Get claim activity history |

**Testing**:
- Integration: set reserve on COV_A for $50,000 → total_reserved updated
- Integration: process $15,000 payment → total_paid updated, reserve reduced
- Integration: payment exceeding reserve → 400
- Integration: claim activities show reserve changes and payments in order
- Unit: total_incurred = total_reserved + total_paid

#### 6.4 — Claim Activities (Diary)

**What**: Activity log for claims: notes, phone calls, inspections, status changes. Every claim action creates an activity.

**Design**:

```python
# src/ipm/models/claim.py (continued)
class ClaimActivity(Base):
    __tablename__ = "claim_activities"

    id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    claim_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("claims.id"), nullable=False)
    activity_type: Mapped[str] = mapped_column(String(50), nullable=False)
    description: Mapped[str] = mapped_column(Text, nullable=False)
    details: Mapped[dict] = mapped_column(JSONB, default=dict)
    performed_by: Mapped[uuid4 | None] = mapped_column(UUID(as_uuid=True), ForeignKey("users.id"))
    performed_at: Mapped[datetime] = mapped_column(server_default=func.now())
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/claims/{id}/activities` | Add manual activity (note, phone call) |
| GET | `/api/v1/claims/{id}/activities` | List activities (paginated, chronological) |

**Testing**:
- Integration: add manual note → activity created with type='note'
- Integration: status change auto-creates activity with old/new status
- Integration: activities are ordered by performed_at descending
- Integration: activities are tenant-scoped through claim ownership

---

## Phase 7: Billing & Payment Processing

### Purpose
Implement premium billing, invoice generation, payment collection (via Stripe), and billing account management. After this phase, issuing a policy creates a billing account with an installment schedule, invoices are generated, and payments can be collected.

### Tasks

#### 7.1 — Billing Account and Invoice Models

**What**: Create billing accounts linked to policyholders, with configurable billing plans and invoice generation.

**Design**:

```python
# src/ipm/models/billing.py
class BillingAccount(TimestampMixin, Base):
    __tablename__ = "billing_accounts"

    id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    tenant_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("tenants.id"), nullable=False)
    account_number: Mapped[str] = mapped_column(String(50), nullable=False)
    policyholder_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("parties.id"), nullable=False)
    billing_plan: Mapped[str] = mapped_column(String(30), nullable=False)
    balance_due: Mapped[Decimal] = mapped_column(Numeric(15, 2), default=0)
    next_due_date: Mapped[date | None] = mapped_column()
    status: Mapped[str] = mapped_column(String(20), default="active")
    payment_methods: Mapped[list] = mapped_column(JSONB, default=list)

    __table_args__ = (
        UniqueConstraint("tenant_id", "account_number", name="uq_billing_tenant_number"),
    )

class Invoice(TimestampMixin, Base):
    __tablename__ = "invoices"

    id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    billing_account_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("billing_accounts.id"))
    policy_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("policies.id"), nullable=False)
    invoice_number: Mapped[str] = mapped_column(String(50), nullable=False)
    invoice_date: Mapped[date] = mapped_column(nullable=False)
    due_date: Mapped[date] = mapped_column(nullable=False)
    amount_due: Mapped[Decimal] = mapped_column(Numeric(15, 2), nullable=False)
    amount_paid: Mapped[Decimal] = mapped_column(Numeric(15, 2), default=0)
    status: Mapped[str] = mapped_column(String(20), nullable=False)
    line_items: Mapped[list] = mapped_column(JSONB, default=list)

class PaymentTransaction(Base):
    __tablename__ = "payment_transactions"

    id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid4)
    tenant_id: Mapped[uuid4] = mapped_column(UUID(as_uuid=True), ForeignKey("tenants.id"), nullable=False)
    invoice_id: Mapped[uuid4 | None] = mapped_column(UUID(as_uuid=True), ForeignKey("invoices.id"))
    transaction_type: Mapped[str] = mapped_column(String(20), nullable=False)
    amount: Mapped[Decimal] = mapped_column(Numeric(15, 2), nullable=False)
    currency: Mapped[str] = mapped_column(String(3), default="USD")
    payment_method: Mapped[str | None] = mapped_column(String(20))
    processor_reference: Mapped[str | None] = mapped_column(String(255))
    iso20022_msg_id: Mapped[str | None] = mapped_column(String(35))
    status: Mapped[str] = mapped_column(String(20), nullable=False)
    processed_at: Mapped[datetime | None] = mapped_column()
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
```

**Testing**:
- Integration: migration creates billing tables with all constraints
- Unit: invoice status constraints enforced

#### 7.2 — Billing Service and Installment Generation

**What**: Service that creates a billing account when a policy is issued and generates an installment schedule based on the billing plan.

**Design**:

```python
# src/ipm/services/billing_service.py
class BillingService:
    async def create_billing_account(
        self, db: AsyncSession, policy: Policy, billing_plan: str
    ) -> BillingAccount:
        """Create billing account and generate invoice schedule."""
        ...

    def generate_installment_schedule(
        self, total_premium: Decimal, billing_plan: str, effective_date: date, term_months: int
    ) -> list[dict]:
        """
        Returns list of installments:
        - annual: 1 installment, due at effective_date
        - semi_annual: 2 installments (50/50 split)
        - quarterly: 4 installments (25% each, due every 3 months)
        - monthly: 12 installments (first = deposit amount, rest equal)
        """
        ...

    async def generate_invoices(
        self, db: AsyncSession, billing_account: BillingAccount, schedule: list[dict], policy: Policy
    ) -> list[Invoice]:
        """Create Invoice records from installment schedule."""
        ...

    async def apply_endorsement_adjustment(
        self, db: AsyncSession, policy_id: UUID, premium_change: Decimal
    ) -> Invoice:
        """
        When an endorsement changes the premium:
        - If increase: create new invoice for the additional amount
        - If decrease: create credit memo / reduce future installments
        """
        ...
```

**Testing**:
- Unit: monthly plan for $2,400 annual premium → 12 installments of $200
- Unit: quarterly plan for $2,400 → 4 installments of $600
- Unit: semi-annual plan for $2,400 → 2 installments of $1,200
- Integration: issue policy → billing account created, invoices generated
- Integration: endorsement with $300 increase → adjustment invoice created
- Unit: installment due dates calculated correctly (monthly intervals)

#### 7.3 — Payment Processing (Stripe Integration)

**What**: Process payments through Stripe, update invoice and billing account balances, handle refunds.

**Design**:

```python
# src/ipm/services/billing_service.py (continued)
class BillingService:
    async def process_payment(
        self, db: AsyncSession, invoice_id: UUID, payment_method_token: str
    ) -> PaymentTransaction:
        """
        1. Load invoice and billing account
        2. Call Stripe Charges API with token and amount
        3. On success: create PaymentTransaction(status='completed')
        4. Update invoice.amount_paid, set status='paid' if fully paid
        5. Update billing_account.balance_due
        6. On failure: create PaymentTransaction(status='failed')
        """
        ...

    async def process_refund(
        self, db: AsyncSession, transaction_id: UUID, amount: Decimal
    ) -> PaymentTransaction:
        """
        1. Load original transaction
        2. Call Stripe Refunds API
        3. Create refund PaymentTransaction
        4. Update invoice and billing account
        """
        ...

    async def add_payment_method(
        self, db: AsyncSession, billing_account_id: UUID, token: str, type: str
    ) -> dict:
        """
        1. Call Stripe to create payment method from token
        2. Store tokenised reference in billing_account.payment_methods JSONB
        3. Never store raw card data (PCI DSS compliance)
        """
        ...
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/billing-accounts` | List billing accounts for tenant |
| GET | `/api/v1/billing-accounts/{id}` | Get billing account detail |
| GET | `/api/v1/billing-accounts/{id}/invoices` | List invoices |
| POST | `/api/v1/invoices/{id}/pay` | Process payment for invoice |
| POST | `/api/v1/invoices/{id}/refund` | Process refund |
| POST | `/api/v1/billing-accounts/{id}/payment-methods` | Add payment method |
| GET | `/api/v1/policies/{id}/invoices` | List invoices for a policy |

**Testing**:
- Integration (mocked Stripe): pay invoice → transaction created, invoice status='paid', balance updated
- Integration (mocked Stripe): payment failure → transaction status='failed', invoice unchanged
- Integration (mocked Stripe): refund → refund transaction created, balance adjusted
- Integration: add payment method → token stored in JSONB, no raw card data
- Integration: payment for amount > invoice balance → 400
- Unit: balance_due calculation after partial payment

---

## Phase 8: ACORD Standards Integration

### Purpose
Implement ACORD XML import and export for policy and claims data, enabling EDI interchange with agents, reinsurers, and regulatory systems. This is a mandatory integration point for real-world insurance operations.

### Tasks

#### 8.1 — ACORD XML Parser (Import)

**What**: Parse ACORD P&C XML messages and map them to internal data structures.

**Design**:

```python
# src/ipm/acord/xml_parser.py
from lxml import etree

class ACORDParser:
    def __init__(self, xsd_path: str | None = None):
        self.schema = etree.XMLSchema(etree.parse(xsd_path)) if xsd_path else None

    def parse_policy_message(self, xml_bytes: bytes) -> dict:
        """
        Parse an ACORD XML policy message into internal format.
        Maps ACORD elements to policy schema:
        - ACORD/InsuranceSvcRq/HomePolicyQuoteInqRq → QuoteRequest
        - PersPolicy → Policy fields
        - PersPolicy/HomeLineBusiness/Dwell → InsuredItem (property)
        - PersPolicy/HomeLineBusiness/Dwell/Coverage → CoverageSelection
        - Producer → Agent
        - InsuredOrPrincipal → Policyholder
        """
        root = etree.fromstring(xml_bytes)
        if self.schema and not self.schema.validate(root):
            errors = [str(e) for e in self.schema.error_log]
            raise ValueError(f"ACORD XML validation failed: {errors}")
        # Extract and map elements
        ...

    def parse_claim_message(self, xml_bytes: bytes) -> dict:
        """Parse ACORD ClaimNotice message into FNOLRequest format."""
        ...
```

```python
# src/ipm/acord/mapping.py
# Mapping tables between ACORD element paths and internal field names
POLICY_MAPPING = {
    "PersPolicy/@PolicyNumber": "policy_number",
    "PersPolicy/ContractTerm/EffectiveDt": "effective_date",
    "PersPolicy/ContractTerm/ExpirationDt": "expiration_date",
    "PersPolicy/PolicyStatusCd": "status",
    "PersPolicy/CurrentTermAmt/Amt": "total_premium",
    "PersPolicy/CurrentTermAmt/CurCd": "currency",
}

COVERAGE_MAPPING = {
    "Coverage/CoverageCd": "code",
    "Coverage/CoverageDesc": "name",
    "Coverage/Limit/FormatCurrencyAmt/Amt": "limit",
    "Coverage/Deductible/FormatCurrencyAmt/Amt": "deductible",
    "Coverage/CurrentTermAmt/Amt": "premium",
}
```

**Testing**:
- Unit: parse valid ACORD policy XML → correct QuoteRequest fields extracted
- Unit: parse ACORD XML with missing required element → clear error message
- Unit: XSD validation catches malformed XML → ValueError with details
- Fixture-based: parse `tests/fixtures/acord_samples/policy_issuance.xml` → all fields mapped correctly
- Unit: ACORD date formats ("2026-07-01") parsed to Python date objects
- Unit: ACORD currency amounts parsed to Decimal (not float)

#### 8.2 — ACORD XML Builder (Export)

**What**: Generate ACORD P&C XML messages from internal data structures for EDI export.

**Design**:

```python
# src/ipm/acord/xml_builder.py
from lxml import etree

class ACORDBuilder:
    ACORD_NAMESPACE = "http://www.ACORD.org/standards/PC_Surety/ACORD1/xml/"

    def build_policy_message(self, policy: dict, message_type: str = "PolicyIssuanceNotif") -> bytes:
        """
        Build an ACORD XML message from internal policy data.
        message_type options:
        - PolicyIssuanceNotif: new policy issuance notification
        - PolicyRenewalNotif: renewal notification
        - PolicyCancelNotif: cancellation notification
        - PolicyEndorsementNotif: endorsement notification
        """
        root = etree.Element("ACORD", nsmap={None: self.ACORD_NAMESPACE})
        # Build XML tree from policy data using mapping
        ...
        return etree.tostring(root, pretty_print=True, xml_declaration=True, encoding="UTF-8")

    def build_claim_message(self, claim: dict) -> bytes:
        """Build ACORD ClaimNotice XML from internal claim data."""
        ...
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/acord/import` | Import ACORD XML message |
| GET | `/api/v1/policies/{id}/acord-xml` | Export policy as ACORD XML |
| GET | `/api/v1/claims/{id}/acord-xml` | Export claim as ACORD XML |

**Testing**:
- Unit: build policy XML → valid ACORD structure with correct namespace
- Unit: build XML from fixture policy data → XSD-validates successfully
- Integration: import ACORD XML → policy created in system
- Integration: export policy as XML → parse back to verify round-trip fidelity
- Unit: round-trip test: build → parse → compare → all fields preserved
- Unit: cancellation notice includes cancellation_date and reason

#### 8.3 — ACORD JSON (Next-Gen Digital Standards) Support

**What**: JSON-based ACORD data format support for modern API-to-API integrations.

**Design**:

The internal API already uses JSON natively. This task defines ACORD-compliant JSON response shapes as alternative serialisation for API endpoints.

```python
# src/ipm/schemas/acord_json.py
class ACORDPolicyResponse(BaseModel):
    """ACORD Next-Gen Digital Standards JSON format for policy data."""
    resourceType: str = "InsurancePolicy"
    policyNumber: str
    policyStatus: str
    contractTerm: dict  # {"effectiveDate": "...", "expirationDate": "..."}
    insuredParty: dict
    coverages: list[dict]
    premium: dict  # {"amount": 2450.00, "currency": "USD"}
```

API endpoints accept `Accept: application/vnd.acord+json` header to return ACORD JSON format.

**Testing**:
- Integration: `GET /api/v1/policies/{id}` with `Accept: application/vnd.acord+json` → ACORD JSON response
- Integration: default `Accept: application/json` → standard response format
- Unit: ACORD JSON response validates against ACORD JSON Schema (if available)

---

## Phase 9: Document Management & Async Tasks

### Purpose
Implement document storage (policy declarations, claim photos, correspondence), document generation, and Celery-based async task processing for long-running operations. After this phase, the system generates PDF declaration pages, stores documents in S3, and processes background tasks like batch renewals.

### Tasks

#### 9.1 — Document Storage Service

**What**: Upload, download, and manage documents linked to policies, claims, and endorsements via S3-compatible storage.

**Design**:

```python
# src/ipm/services/document_service.py
import boto3
from uuid import UUID

class DocumentService:
    def __init__(self, settings):
        self.s3 = boto3.client(
            "s3",
            endpoint_url=settings.s3_endpoint_url,
            aws_access_key_id=settings.s3_access_key,
            aws_secret_access_key=settings.s3_secret_key,
            region_name=settings.s3_region,
        )
        self.bucket = settings.s3_bucket_name

    async def upload_document(
        self, db: AsyncSession, tenant_id: UUID, entity_type: str, entity_id: UUID,
        document_type: str, file_name: str, file_content: bytes, mime_type: str,
        uploaded_by: UUID,
    ) -> Document:
        """
        1. Generate storage path: {tenant_id}/{entity_type}/{entity_id}/{uuid}_{file_name}
        2. Upload to S3
        3. Create Document record in database
        4. Return document with pre-signed download URL
        """
        ...

    async def get_download_url(self, document_id: UUID, expires_in: int = 3600) -> str:
        """Generate pre-signed S3 URL for secure download."""
        ...

    async def delete_document(self, db: AsyncSession, document_id: UUID) -> None:
        """Delete from S3 and mark database record as deleted."""
        ...
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/documents/upload` | Upload document (multipart form) |
| GET | `/api/v1/documents/{id}` | Get document metadata + download URL |
| GET | `/api/v1/documents?entity_type=policy&entity_id={id}` | List documents for entity |
| DELETE | `/api/v1/documents/{id}` | Delete document |

**Testing**:
- Integration (mocked S3): upload file → document record created, S3 put_object called
- Integration (mocked S3): get download URL → valid pre-signed URL returned
- Integration: list documents for a policy → returns only that policy's documents
- Integration: tenant isolation — cannot access other tenant's documents

#### 9.2 — Celery Task Infrastructure

**What**: Set up Celery with Redis broker for background task processing.

**Design**:

```python
# src/ipm/tasks/celery_app.py
from celery import Celery
from ipm.config import get_settings

settings = get_settings()

celery_app = Celery(
    "ipm",
    broker=settings.celery_broker_url,
    backend=settings.celery_result_backend,
)

celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="UTC",
    enable_utc=True,
    task_track_started=True,
    task_acks_late=True,
    worker_prefetch_multiplier=1,
)
```

Add Celery worker to docker-compose:

```yaml
  celery-worker:
    build: .
    command: celery -A ipm.tasks.celery_app worker --loglevel=info
    environment: *api-env
    depends_on:
      - redis
      - db
```

**Testing**:
- Integration: Celery worker connects to Redis broker
- Integration: enqueue task → task executes and result stored in Redis
- Unit: Celery configuration matches expected settings

#### 9.3 — Background Task Definitions

**What**: Implement specific async tasks: batch renewal processing, document generation, overdue invoice notifications.

**Design**:

```python
# src/ipm/tasks/renewal_tasks.py
from ipm.tasks.celery_app import celery_app

@celery_app.task
def process_batch_renewals(tenant_id: str, days_before_expiration: int = 60):
    """
    Find all active policies expiring within N days that don't have a renewal quote.
    For each, create a renewal quote with current rating tables.
    """
    ...

# src/ipm/tasks/billing_tasks.py
@celery_app.task
def check_overdue_invoices(tenant_id: str):
    """
    Find invoices past due_date with status='sent'.
    Update status to 'overdue'.
    Create communication record for overdue notice.
    """
    ...

@celery_app.task
def generate_declaration_page(policy_id: str):
    """
    Generate PDF declaration page for issued policy.
    Upload to S3 via DocumentService.
    """
    ...
```

**Testing**:
- Integration (mocked): batch renewal finds 3 expiring policies → 3 renewal quotes created
- Integration: overdue invoice check updates status correctly
- Unit: declaration page generation produces valid PDF bytes

---

## Phase 10: Reinsurance & Commissions

### Purpose
Implement reinsurance treaty management with cession calculations, and agent commission tracking. These are essential operational modules for carriers and MGAs.

### Tasks

#### 10.1 — Reinsurance Treaty Management

**What**: CRUD for reinsurance treaties with JSONB terms, and automatic cession calculation when policies are issued.

**Design**:

```python
# src/ipm/models/reinsurance.py
class ReinsuranceTreaty(TimestampMixin, Base):
    __tablename__ = "reinsurance_treaties"
    # ... (per Data Model Suggestion 3)

class ReinsuranceCession(Base):
    __tablename__ = "reinsurance_cessions"
    # ... (per Data Model Suggestion 3)
```

```python
# src/ipm/services/reinsurance_service.py
class ReinsuranceService:
    async def calculate_cessions(
        self, db: AsyncSession, policy: Policy
    ) -> list[ReinsuranceCession]:
        """
        For each active treaty applicable to this policy's LOB and jurisdiction:
        - Quota share: ceded_premium = total_premium * cession_percentage
        - Excess of loss: calculate attachment point and cession
        - Apply ceding commission: commission = ceded_premium * commission_rate
        """
        ...
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/reinsurance/treaties` | Create treaty |
| GET | `/api/v1/reinsurance/treaties` | List treaties |
| GET | `/api/v1/reinsurance/treaties/{id}` | Get treaty detail |
| GET | `/api/v1/reinsurance/treaties/{id}/cessions` | List cessions for treaty |
| GET | `/api/v1/policies/{id}/cessions` | List cessions for a policy |

**Testing**:
- Integration: create quota share treaty → stored with terms in JSONB
- Integration: issue policy in covered LOB → cession auto-calculated
- Unit: quota share cession: $2,400 premium * 30% = $720 ceded, $180 commission at 25%
- Unit: excess of loss: premium below retention → no cession
- Integration: treaty with excluded perils filters correctly

#### 10.2 — Commission Management

**What**: Commission calculation for agents based on commission schedules, with statement generation.

**Design**:

```python
# src/ipm/models/commission.py
class CommissionEntry(Base):
    __tablename__ = "commission_entries"
    # ... (per Data Model Suggestion 3)
```

```python
# src/ipm/services/commission_service.py
class CommissionService:
    async def calculate_commission(
        self, db: AsyncSession, policy: Policy, agent_id: UUID, commission_type: str
    ) -> CommissionEntry:
        """
        1. Load commission schedule for agent/product
        2. Apply rate (new_business_rate or renewal_rate) to premium
        3. Create commission_entry record
        """
        ...

    async def generate_statement(
        self, db: AsyncSession, agent_id: UUID, period_start: date, period_end: date
    ) -> dict:
        """Aggregate commission entries for an agent over a period."""
        ...
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/commissions` | List commission entries (filterable by agent, period) |
| GET | `/api/v1/agents/{id}/commission-statement` | Generate commission statement |

**Testing**:
- Integration: issue policy → commission entry created for agent
- Unit: new business commission: $2,400 * 15% = $360
- Unit: renewal commission: $2,400 * 10% = $240
- Integration: commission statement aggregates entries for the period

---

## Phase 11: Reporting & Search

### Purpose
Implement reporting endpoints for key insurance metrics and full-text search across policies, claims, and parties. After this phase, users can query loss ratios, premium summaries, and search the system efficiently.

### Tasks

#### 11.1 — Reporting Endpoints

**What**: Aggregate reporting queries for policy, claims, and billing analytics.

**Design**:

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/reports/policy-summary` | Policy count/premium by status, LOB, jurisdiction |
| GET | `/api/v1/reports/claims-summary` | Claims count/incurred by status, loss_type, period |
| GET | `/api/v1/reports/loss-ratio` | Loss ratio (incurred losses / earned premium) by LOB |
| GET | `/api/v1/reports/billing-aging` | Outstanding invoices by aging bucket (current, 30, 60, 90+ days) |
| GET | `/api/v1/reports/renewal-pipeline` | Policies expiring in next 30/60/90 days with renewal status |

Reports use raw SQL for performance on aggregate queries, not ORM:

```python
# Example: loss ratio query
async def get_loss_ratio(db: AsyncSession, tenant_id: UUID, period_start: date, period_end: date):
    result = await db.execute(text("""
        SELECT
            pr.line_of_business,
            SUM(p.total_premium) AS earned_premium,
            SUM(c.total_incurred) AS incurred_losses,
            CASE WHEN SUM(p.total_premium) > 0
                 THEN ROUND(SUM(c.total_incurred) / SUM(p.total_premium) * 100, 2)
                 ELSE 0 END AS loss_ratio_pct
        FROM policies p
        JOIN products pr ON p.product_id = pr.id
        LEFT JOIN claims c ON c.policy_id = p.id AND c.loss_date BETWEEN :start AND :end
        WHERE p.tenant_id = :tenant_id
          AND p.status IN ('active', 'expired', 'cancelled', 'renewed')
          AND p.effective_date <= :end AND p.expiration_date >= :start
        GROUP BY pr.line_of_business
        ORDER BY earned_premium DESC
    """), {"tenant_id": tenant_id, "start": period_start, "end": period_end})
    return result.mappings().all()
```

**Testing**:
- Integration: policy summary with 10 test policies → correct aggregation by status
- Integration: loss ratio with claims → mathematically correct ratio
- Integration: billing aging with overdue invoices → correct bucket assignment
- Integration: renewal pipeline shows policies expiring within window

#### 11.2 — Search

**What**: Search across policies, claims, and parties using PostgreSQL full-text search and JSONB queries.

**Design**:

```python
# src/ipm/api/search.py
@router.get("/api/v1/search")
async def search(
    q: str,
    entity_types: list[str] = Query(default=["policy", "claim", "party"]),
    db: AsyncSession = Depends(get_db),
    user: User = Depends(get_current_user),
):
    """
    Search across entities:
    - Policies: by policy_number, policyholder name
    - Claims: by claim_number, loss_description
    - Parties: by name, email, phone
    Uses PostgreSQL ts_vector for full-text search + ILIKE for exact matches.
    """
    ...
```

**Testing**:
- Integration: search "Smith" → returns matching parties and their policies
- Integration: search "CLM-2026" → returns matching claims
- Integration: search "HO3-2026-000001" → returns exact policy match
- Integration: search results are tenant-scoped

---

## Phase 12: AI-Assisted Features (v1.1 Foundation)

### Purpose
Lay the foundation for AI-native capabilities: ML-based risk scoring for underwriting, rule-based claims triage, and the integration points for future LLM-powered features. This phase delivers the first AI differentiators.

### Tasks

#### 12.1 — ML Risk Scoring for Underwriting

**What**: Train and deploy a scikit-learn model that generates underwriting risk scores from policy and risk characteristics.

**Design**:

```python
# src/ipm/services/risk_scoring.py
from sklearn.ensemble import GradientBoostingClassifier
import joblib

class RiskScoringService:
    def __init__(self, model_path: str | None = None):
        self.model = joblib.load(model_path) if model_path else None

    def score_risk(self, features: dict) -> dict:
        """
        Input features:
        - protection_class (int, 1-10)
        - territory_code (str → encoded)
        - year_built (int → age)
        - prior_claims_count (int)
        - credit_tier (str → encoded)
        - coverage_a_limit (float)
        - loss_history_5yr (int, count)

        Output:
        - risk_score (float, 0-100)
        - risk_tier ('preferred', 'standard', 'substandard')
        - confidence (float, 0-1)
        - factors: list of contributing factor explanations
        """
        if self.model is None:
            return self._rule_based_score(features)
        # ML model scoring
        ...

    def _rule_based_score(self, features: dict) -> dict:
        """
        Fallback rule-based scoring when no ML model is available:
        - Start at 50
        - +10 for protection_class > 7
        - +15 for prior_claims_count > 2
        - -10 for credit_tier == 'excellent'
        - +20 for age_of_home > 40
        etc.
        """
        score = 50.0
        factors = []
        if features.get("prior_claims_count", 0) > 2:
            score += 15
            factors.append("Multiple prior claims (+15)")
        # ... additional rules
        tier = "preferred" if score < 40 else "standard" if score < 70 else "substandard"
        return {"risk_score": score, "risk_tier": tier, "confidence": 0.6, "factors": factors}
```

```python
# Training script: scripts/train_risk_model.py
# Trains on historical policy + claims data
# Exports model to models/risk_model.joblib
```

**Testing**:
- Unit: rule-based scoring with clean profile → score < 40, tier='preferred'
- Unit: rule-based scoring with multiple prior claims → score increases
- Unit: rule-based scoring with high protection class + old home → tier='substandard'
- Integration: underwriting submission with risk scoring → risk_score stored in policy details
- Unit: feature engineering transforms raw inputs correctly (year_built → age)

#### 12.2 — Rule-Based Claims Triage

**What**: Automated routing of new claims based on severity, loss type, and coverage amounts.

**Design**:

```python
# src/ipm/services/claims_triage.py
class ClaimsTriageService:
    TRIAGE_RULES = {
        "auto_approve": {
            "max_reserve": 5000,
            "loss_types": ["glass_breakage", "minor_water"],
            "max_loss_age_days": 30,
        },
        "priority_high": {
            "min_reserve": 100000,
            "loss_types": ["fire", "total_loss", "liability_injury"],
        },
        "fraud_flags": {
            "recent_policy_days": 90,       # Claim within 90 days of inception
            "prior_claims_threshold": 3,     # 3+ claims in 12 months
            "late_reporting_days": 30,       # Reported > 30 days after loss
        },
    }

    async def triage_claim(self, claim: Claim, policy: Policy) -> dict:
        """
        Returns:
        - routing: 'auto_approve' | 'standard' | 'priority_high' | 'siu_referral'
        - fraud_flags: list of triggered fraud indicators
        - assigned_team: suggested adjuster team based on loss_type
        """
        ...
```

**Testing**:
- Unit: small glass claim → auto_approve routing
- Unit: large fire claim → priority_high routing
- Unit: claim filed 10 days after policy inception → fraud flag triggered
- Unit: 4th claim in 12 months → fraud flag triggered
- Integration: FNOL with triage → routing stored in claim details, adjuster assigned by team

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                     ─── required by everything
    │
Phase 2: Core Data Models              ─── requires Phase 1
    │
Phase 3: Products & Rating             ─── requires Phase 2
    │
Phase 4: Policy Lifecycle              ─── requires Phase 3
    │
    ├── Phase 5: Endorsements & UW      ─── requires Phase 4
    ├── Phase 6: Claims Management      ─── requires Phase 4
    └── Phase 7: Billing & Payments     ─── requires Phase 4
         │
         ├── Phase 8: ACORD Integration ─── requires Phases 4+6 (can parallel with 9, 10)
         ├── Phase 9: Documents & Tasks ─── requires Phases 4+6+7 (can parallel with 8, 10)
         └── Phase 10: Reinsurance & Commissions ─── requires Phases 4+7 (can parallel with 8, 9)
              │
              Phase 11: Reporting & Search       ─── requires Phases 4+6+7
              │
              Phase 12: AI Features              ─── requires Phases 5+6
```

### Parallelism Opportunities

- **Phases 5, 6, and 7** can all be developed concurrently after Phase 4 is complete. They are independent modules that connect to policies but not to each other.
- **Phases 8, 9, and 10** can all be developed concurrently after their respective dependencies are met.
- **Phase 11** depends on multiple data modules being in place but could start after Phase 7 with stub data for claims.
- **Phase 12** can begin as soon as Phases 5 and 6 are complete (underwriting and claims provide the integration points).

---

## Definition of Done (per phase)

1. All tasks implemented with production-quality code.
2. All unit tests pass (`pytest tests/unit/`).
3. All integration tests pass (`pytest tests/integration/`).
4. Ruff linting passes with zero warnings (`ruff check src/ tests/`).
5. Ruff formatting applied (`ruff format src/ tests/`).
6. mypy type checking passes (`mypy src/`).
7. Docker build succeeds (`docker build -t ipm .`).
8. `docker compose up` starts all services and health check returns "healthy".
9. New API endpoints appear in auto-generated OpenAPI spec at `/docs`.
10. Database migrations created and apply cleanly (`alembic upgrade head`).
11. Database migrations downgrade cleanly (`alembic downgrade -1`).
12. Audit logging covers all create/update/delete operations in the phase.
13. Tenant isolation verified — cross-tenant data access returns 403 or empty results.
14. Test coverage for the phase's code is above 80%.
