# Project Template Guidelines
## HTMX + TailwindCSS + SQLite/SQLAlchemy + FastAPI

Distilled from NEESH production experience. Copy-paste these patterns for new projects.

---

## Table of Contents

1. [Stack](#stack)
2. [Project Structure](#project-structure)
3. [Portable Path Anchoring](#portable-path-anchoring)
4. [Virtual Environment](#virtual-environment)
5. [pyproject.toml](#pyprojecttoml)
6. [Configuration & Environment Files](#configuration--environment-files)
7. [Windows Dev Scripts (.bat)](#windows-dev-scripts-bat)
7. [TailwindCSS Build](#tailwindcss-build)
8. [Database — SQLAlchemy + Alembic](#database--sqlalchemy--alembic)
9. [FastAPI App Structure](#fastapi-app-structure)
10. [HTMX Patterns](#htmx-patterns)
11. [Testing — pytest unit tests](#testing--pytest-unit-tests)
12. [Testing — Playwright E2E](#testing--playwright-e2e)
13. [Linux Deployment](#linux-deployment)
14. [systemd Service File](#systemd-service-file)
15. [Deploy Script](#deploy-script)
16. [CLAUDE.md Template](#claudemd-template)

---

## Stack

| Layer | Choice | Why |
|-------|--------|-----|
| Web framework | FastAPI + uvicorn | Async, typed, fast |
| Templates | Jinja2 | Server-side HTML, works with HTMX |
| Interactivity | HTMX | Partial page updates without a JS framework |
| Styles | TailwindCSS v3 | Utility-first, pre-built on dev, no Node on servers |
| ORM | SQLAlchemy 2 async | Async queries, Alembic migrations |
| Database | SQLite (aiosqlite) | Zero-infra, perfect for Pi/home servers |
| Config | pydantic-settings | Type-safe env var loading |
| Testing | pytest + pytest-asyncio + Playwright | Unit + E2E coverage |
| Background jobs | APScheduler | Simple cron-style jobs |

---

## Project Structure

```
myapp/
├── app/
│   ├── main.py              # FastAPI app creation, router registration
│   ├── config.py            # Settings (pydantic-settings), Jinja2 templates
│   ├── database.py          # Engine, session factory, init_db()
│   ├── dependencies.py      # FastAPI dependency functions (get_db, get_current_user)
│   ├── error_handlers.py    # 404/500 error pages
│   ├── logging_config.py    # Structured logging setup
│   ├── models/
│   │   ├── __init__.py      # Re-export all models (required for Alembic)
│   │   ├── base.py          # DeclarativeBase, TimestampMixin
│   │   └── user.py          # Example model
│   ├── schemas/             # Pydantic schemas for request/response validation
│   ├── repositories/        # Database CRUD (raw SQLAlchemy queries)
│   ├── services/            # Business logic (calls repositories)
│   ├── routes/
│   │   ├── auth.py          # Full-page routes
│   │   ├── dashboard.py
│   │   └── api_fragments.py # HTMX partial HTML routes (/api/fragments/*)
│   ├── templates/
│   │   ├── base.html        # <!DOCTYPE html>, HTMX script, TailwindCSS link
│   │   ├── components/
│   │   │   └── nav.html
│   │   ├── dashboard/
│   │   │   ├── index.html
│   │   │   └── fragments/   # HTMX partial templates
│   │   │       └── some_card.html
│   │   └── auth/
│   └── jobs/
│       └── scheduler.py     # APScheduler setup
├── static/
│   ├── css/
│   │   ├── input.css        # Tailwind @tailwind directives + custom CSS
│   │   └── output.css       # BUILT — committed to git
│   └── js/
│       └── htmx.min.js      # Vendored HTMX (no CDN dependency)
├── migrations/              # Alembic migrations
│   ├── env.py
│   └── versions/
├── tests/
│   ├── conftest.py          # Shared fixtures: test_db, test_client, mock_user
│   ├── unit/
│   │   └── test_*.py
│   ├── e2e/
│   │   └── test_*.py        # Playwright tests
│   └── fixtures/            # Sample files for upload/import tests
├── scripts/
│   ├── setup_pi.sh          # One-time Pi/Linux setup
│   ├── deploy.sh            # Safe deploy (pull → migrate → restart)
│   └── setup_services.sh    # Generate systemd service files
├── docs/
├── data/                    # SQLite databases (gitignored)
├── logs/                    # Log files (gitignored)
├── backups/                 # DB backups (gitignored)
├── .tmp/                    # Scratch scripts (gitignored)
├── pyproject.toml
├── package.json             # Only for Tailwind build — devDependency only
├── tailwind.config.js
├── alembic.ini
├── .env.example             # Template — commit this
├── .env.prod.example        # Prod template — commit this
├── .gitignore
├── setup.bat                # Windows: create venv, install deps, init env files
├── start_dev.bat            # Windows: activate venv, start Tailwind watcher, uvicorn
├── start_user.bat           # Windows: production data server
├── run_e2e_tests.bat        # Windows: clean DB, run playwright, copy to dev DB
└── clean_test_db.bat        # Windows: delete test DB
```

---

## Portable Path Anchoring

**The rule:** Never hardcode absolute paths. Every script, config file, and service
definition must derive its root directory from its own location on disk. This makes
the project freely movable (`mv ~/myapp /srv/apps/myapp`, cloned to a different user's
home, or checked out on a new machine) without touching a single path string.

### Python — app/config.py

Use `__file__` to anchor to the source file's location, then navigate relative to it.

```python
from pathlib import Path

# config.py lives at: <project_root>/app/config.py
# .parent      → app/
# .parent.parent → <project_root>/
PROJECT_ROOT = Path(__file__).parent.parent

class Settings(BaseSettings):
    DATABASE_PATH: str = "./data/myapp.db"

    @property
    def DATABASE_URL(self) -> str:
        db_path = Path(self.DATABASE_PATH)
        # Make relative paths absolute, anchored to project root
        if not db_path.is_absolute():
            db_path = PROJECT_ROOT / db_path
        db_path.parent.mkdir(parents=True, exist_ok=True)
        return f"sqlite+aiosqlite:///{db_path}"

    @property
    def LOG_PATH(self) -> Path:
        log_dir = Path(self.LOG_DIR)
        if not log_dir.is_absolute():
            log_dir = PROJECT_ROOT / log_dir
        log_dir.mkdir(parents=True, exist_ok=True)
        return log_dir / "myapp.log"
```

`.env.*` files can then use short relative paths (`DATABASE_PATH=./data/myapp.db`)
that get resolved correctly wherever the project lives.

### Bash scripts

```bash
#!/bin/bash
# Use BASH_SOURCE[0] — works even when the script is sourced, called via symlink,
# or run from a different working directory.

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
APP_DIR="$(dirname "$SCRIPT_DIR")"   # scripts/ lives one level inside project root

# Now reference everything via $APP_DIR — never assume cwd
VENV="$APP_DIR/venv"
DB_PATH="$APP_DIR/data/myapp.db"
ENV_FILE_PATH="$APP_DIR/.env.prod"

source "$VENV/bin/activate"
cd "$APP_DIR"
ENV_FILE="$ENV_FILE_PATH" alembic upgrade head
```

### Windows .bat scripts

```bat
@echo off
REM %~dp0 expands to the drive+path of the .bat file itself (always has trailing backslash)
REM Works correctly regardless of where you launched the script from.

set APP_DIR=%~dp0
REM Strip trailing backslash
set APP_DIR=%APP_DIR:~0,-1%

set VENV=%APP_DIR%\venv
set ENV_DEV=%APP_DIR%\.env.dev

call "%VENV%\Scripts\activate.bat"
set ENV_FILE=%ENV_DEV%
python -m uvicorn app.main:app --reload --port 8000
```

### systemd service — use auto-detection, not hardcoded paths

Don't commit a service file with `/home/pi/myapp` baked in — the path breaks the
moment you move the project. Instead, generate it from the repo:

```bash
# scripts/setup_services.sh — run once on the server
CURRENT_USER=$(whoami)
APP_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

cat > "$APP_DIR/deploy/myapp.service" << EOF
[Service]
User=$CURRENT_USER
WorkingDirectory=$APP_DIR
Environment="PATH=$APP_DIR/venv/bin:/usr/bin"
ExecStart=$APP_DIR/venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8000
ReadWritePaths=$APP_DIR/data $APP_DIR/logs $APP_DIR/backups
EOF
```

Run `./scripts/setup_services.sh` after cloning or moving the project, then install
the generated `deploy/myapp.service`. The service file in the repo root (`myapp.service`)
is a human-readable template with comments; the generated one in `deploy/` has real paths.

### alembic.ini

Don't set `sqlalchemy.url` here — it differs per environment. Set it at runtime:

```python
# migrations/env.py
from app.config import settings   # settings reads ENV_FILE → resolves DATABASE_URL
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)
```

```ini
# alembic.ini — no URL here
[alembic]
script_location = migrations
sqlalchemy.url =           # left blank — set in env.py
```

### Summary checklist

| Context | Pattern |
|---------|---------|
| Python config | `Path(__file__).parent.parent` |
| Bash scripts | `$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)` |
| .bat scripts | `%~dp0` (strip trailing `\`) |
| systemd service | Generated by `setup_services.sh`, not committed with hardcoded paths |
| alembic.ini | No URL; resolved in `migrations/env.py` from `settings.DATABASE_URL` |
| .env files | Use relative paths (`./data/myapp.db`); `config.py` makes them absolute |

---

## Virtual Environment

### Standard (recommended)
```bat
python -m venv venv
```

### With system-site-packages (use on ARM/Pi for pre-compiled numpy/pandas)
```bash
# On Raspberry Pi where apt packages are pre-compiled for ARM:
python3 -m venv --system-site-packages venv
# This lets venv inherit system-installed numpy, pandas, scipy
# without recompiling from source (which is very slow on Pi 3/4)
```

**Rule:** Use `--system-site-packages` on deployment targets where:
- You're on ARM and `pip install pandas` would compile from source
- The system already has the heavy packages installed via `apt`

Always use `venv/Scripts/python` (Windows) or `venv/bin/python` (Linux) explicitly in scripts — never bare `python` after activation to avoid confusion.

---

## pyproject.toml

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "myapp"
version = "1.0.0"
description = "My App"
dependencies = [
    # Web
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.34.0",
    "jinja2>=3.1.0",
    "python-multipart>=0.0.18",

    # Database
    "sqlalchemy[asyncio]>=2.0.0",
    "aiosqlite>=0.20.0",
    "alembic>=1.14.0",

    # Config
    "pydantic-settings>=2.7.0",

    # Auth
    "PyJWT>=2.10.0",

    # Background jobs
    "APScheduler>=3.10.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=6.0.0",
    "pytest-playwright>=0.5.0",
    "httpx>=0.28.0",
]

[tool.setuptools.packages.find]
where = ["."]
include = ["app*"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
```

Install with `pip install -e ".[dev]"` — the `-e` flag means edits to `app/` are
reflected immediately without reinstalling.

---

## Configuration & Environment Files

### Three-env pattern

| File | Database | Port | Purpose |
|------|----------|------|---------|
| `.env.dev` | `data/myapp_dev.db` | 8000 | Review test data |
| `.env.test` | `test_e2e.db` | 8765 | E2E test isolation |
| `.env.prod` | `data/myapp.db` | 8000 | Real data |

Select env file via `ENV_FILE` environment variable before launching uvicorn:
```bat
set ENV_FILE=.env.dev
python -m uvicorn app.main:app --reload
```

### .env.example (commit this)
```ini
# ── Application ──────────────────────────────────────────────────────
APP_NAME=MyApp
DEBUG=true
DEV_MODE=true
PORT=8000

# ── Database ─────────────────────────────────────────────────────────
DATABASE_PATH=./data/myapp.db

# ── Auth ─────────────────────────────────────────────────────────────
# Generate: python -c "import secrets; print(secrets.token_urlsafe(32))"
SECRET_KEY=CHANGE-THIS-IN-PRODUCTION
JWT_SECRET=CHANGE-THIS-IN-PRODUCTION

# ── Dev credentials (DEV_MODE only, ignored in prod) ─────────────────
DEV_TEST_PHONE=9999999999
DEV_TEST_PASSWORD=dev123

# ── Logging ──────────────────────────────────────────────────────────
LOG_LEVEL=DEBUG
LOG_DIR=./logs
```

### app/config.py skeleton

```python
import os
from pathlib import Path
from pydantic_settings import BaseSettings, SettingsConfigDict
from fastapi.templating import Jinja2Templates


def get_env_file() -> str:
    project_root = Path(__file__).parent.parent
    env_file = os.environ.get("ENV_FILE", ".env")
    path = project_root / env_file
    if path.exists():
        return str(path)
    fallback = project_root / ".env"
    return str(fallback) if fallback.exists() else str(path)


class Settings(BaseSettings):
    APP_NAME: str = "MyApp"
    DEBUG: bool = True
    DEV_MODE: bool = True
    PORT: int = 8000
    DATABASE_PATH: str = "./data/myapp.db"
    SECRET_KEY: str = "CHANGE-ME"
    JWT_SECRET: str = "CHANGE-ME"
    LOG_LEVEL: str = "DEBUG"

    @property
    def DATABASE_URL(self) -> str:
        db_path = Path(self.DATABASE_PATH)
        if not db_path.is_absolute():
            db_path = Path(__file__).parent.parent / db_path
        db_path.parent.mkdir(parents=True, exist_ok=True)
        return f"sqlite+aiosqlite:///{db_path}"

    model_config = SettingsConfigDict(
        env_file=get_env_file(),
        env_file_encoding="utf-8",
        case_sensitive=True,
        extra="ignore",
    )


settings = Settings()
templates = Jinja2Templates(directory="app/templates")
```

---

## Windows Dev Scripts (.bat)

### setup.bat

```bat
@echo off
REM One-time setup after clone

echo Checking Python...
python --version 2>nul || (echo ERROR: Python not found && exit /b 1)

REM Create venv
if not exist venv (
    echo Creating virtual environment...
    python -m venv venv
) else (
    echo Virtual environment already exists.
)

call venv\Scripts\activate.bat
python -m pip install --upgrade pip
pip install -e ".[dev]"
playwright install chromium

REM Init env files
if not exist .env.dev  copy .env.example .env.dev  >nul
if not exist .env.test copy .env.example .env.test >nul
if not exist .env.prod copy .env.example .env.prod >nul

REM Patch DATABASE_PATH per env
python -c "import re; f=open('.env.dev','r').read(); open('.env.dev','w').write(re.sub(r'DATABASE_PATH=.*','DATABASE_PATH=./data/myapp_dev.db',f))"
python -c "import re; f=open('.env.test','r').read(); open('.env.test','w').write(re.sub(r'DATABASE_PATH=.*','DATABASE_PATH=./test_e2e.db',re.sub(r'PORT=8000','PORT=8765',f)))"

if not exist data mkdir data

echo.
echo Setup complete! Edit .env.dev then run: .\start_dev.bat
```

### start_dev.bat

```bat
@echo off
REM Dev server: test data, Tailwind watcher

if exist venv\Scripts\activate.bat (call venv\Scripts\activate.bat) else (echo WARNING: no venv)
if not exist .env.dev (echo ERROR: .env.dev missing && exit /b 1)

REM Start Tailwind watcher in a background window
if exist package.json (
    if not exist node_modules (call npm install --silent)
    start "Tailwind Watcher" /MIN cmd /c "npm run watch:css"
)

set ENV_FILE=.env.dev
python -m uvicorn app.main:app --reload --host localhost --port 8000
```

### start_user.bat (production data)

```bat
@echo off
if exist venv\Scripts\activate.bat (call venv\Scripts\activate.bat)
if not exist .env.prod (echo ERROR: .env.prod missing && exit /b 1)

set ENV_FILE=.env.prod
python -m uvicorn app.main:app --host localhost --port 8000
```

### run_e2e_tests.bat

```bat
@echo off
if exist venv\Scripts\activate.bat (call venv\Scripts\activate.bat)
if not exist .env.test (echo ERROR: .env.test missing && exit /b 1)

netstat -an | findstr ":8765" >nul && (echo ERROR: Port 8765 in use && exit /b 1)

call clean_test_db.bat

set ENV_FILE=.env.test
pytest tests/e2e/ -v %*

if not exist data mkdir data
if exist test_e2e.db (
    copy /Y test_e2e.db data\myapp_dev.db >nul
    echo Copied test DB → data\myapp_dev.db. Run .\start_dev.bat to review.
)
```

### clean_test_db.bat

```bat
@echo off
if exist test_e2e.db del test_e2e.db && echo Deleted test_e2e.db
```

---

## TailwindCSS Build

### package.json

```json
{
  "scripts": {
    "build:css": "tailwindcss -i ./static/css/input.css -o ./static/css/output.css --minify",
    "watch:css": "tailwindcss -i ./static/css/input.css -o ./static/css/output.css --watch"
  },
  "devDependencies": {
    "tailwindcss": "^3.4.0"
  }
}
```

### tailwind.config.js

```js
module.exports = {
  content: ["./app/templates/**/*.html", "./static/js/**/*.js"],
  darkMode: "class",
  theme: { extend: {} },
  plugins: [],
}
```

### static/css/input.css

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Custom utilities go here */
.htmx-indicator { display: none; }
.htmx-request .htmx-indicator { display: inline; }
```

### Key rule: CSS is pre-built and committed

- Build CSS on your **dev machine**: `npm run build:css`
- Commit `static/css/output.css` to git
- Deployment targets (Pi, BB) do **not** need Node.js
- deploy.sh checks for `output.css` and warns if missing

---

## Database — SQLAlchemy + Alembic

### app/models/base.py

```python
from datetime import datetime, timezone
from sqlalchemy import DateTime
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc),
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc),
    )
```

**SQLAlchemy reserved names:** Never name a column `metadata`. Use `action_metadata` (mapped to `"metadata"` column via `mapped_column("metadata")`).

### app/database.py

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from app.config import settings
from app.models.base import Base

engine = create_async_engine(settings.DATABASE_URL, echo=False)
async_session_factory = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def init_db():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

async def get_db_session():
    async with async_session_factory() as session:
        yield session
```

### app/dependencies.py

```python
from app.database import async_session_factory
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

async def get_db() -> AsyncSession:
    async with async_session_factory() as db:
        yield db
```

### alembic.ini (key line)

```ini
[alembic]
script_location = migrations
# URL is set dynamically in migrations/env.py from settings
sqlalchemy.url =
```

### migrations/env.py (key pattern)

```python
import os
from app.config import settings
from app.models import Base   # imports ALL models

config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)
target_metadata = Base.metadata
```

### Workflow

```bash
# Create a migration after model change
ENV_FILE=.env.dev alembic revision --autogenerate -m "add_widget_table"

# Apply migrations
ENV_FILE=.env.dev alembic upgrade head

# In deploy.sh on server
ENV_FILE=.env.prod alembic upgrade head
```

---

## FastAPI App Structure

### app/main.py skeleton

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from app.config import settings
from app.database import init_db
from app.error_handlers import register_error_handlers

@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_db()
    # start background jobs here
    yield
    # cleanup here

app = FastAPI(
    title=settings.APP_NAME,
    debug=settings.DEBUG,
    lifespan=lifespan,
    docs_url="/api/docs" if settings.DEBUG else None,
)

register_error_handlers(app)
app.mount("/static", StaticFiles(directory="static"), name="static")

# Register routers
from app.routes.auth import router as auth_router
from app.routes.dashboard import router as dashboard_router
from app.routes.api_fragments import router as fragments_router

app.include_router(auth_router, tags=["Auth"])
app.include_router(dashboard_router, tags=["Dashboard"])
app.include_router(fragments_router, tags=["Fragments"])
```

### Route file pattern

```python
from fastapi import APIRouter, Depends, Request
from fastapi.responses import HTMLResponse
from sqlalchemy.ext.asyncio import AsyncSession
from app.config import templates
from app.dependencies import get_db, get_current_user

router = APIRouter(prefix="/dashboard")

@router.get("/", response_class=HTMLResponse)
async def dashboard(
    request: Request,
    db: AsyncSession = Depends(get_db),
    current_user = Depends(get_current_user),
):
    service = DashboardService(db)
    data = await service.get_data(current_user.id)
    return templates.TemplateResponse(
        "dashboard/index.html",
        {"request": request, "data": data, "user": current_user},
    )
```

---

## HTMX Patterns

### base.html (minimum required)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}App{% endblock %}</title>
    <link rel="stylesheet" href="/static/css/output.css">
    <script src="/static/js/htmx.min.js"></script>
</head>
<body>
    {% include 'components/nav.html' %}
    {% block content %}{% endblock %}
</body>
</html>
```

### Loading indicator (global)

Add to `input.css`:
```css
.htmx-indicator { display: none; }
.htmx-request .htmx-indicator { display: inline; }
```

Use in templates:
```html
<div hx-get="/api/fragments/card"
     hx-trigger="load"
     hx-swap="outerHTML">
    <span class="htmx-indicator">Loading...</span>
</div>
```

### Fragment route (server side)

```python
# app/routes/api_fragments.py
router = APIRouter(prefix="/api")

@router.get("/fragments/some-card", response_class=HTMLResponse)
async def some_card(request: Request, db = Depends(get_db), user = Depends(get_current_user)):
    data = await SomeService(db).get_data(user.id)
    return templates.TemplateResponse(
        "dashboard/fragments/some_card.html",
        {"request": request, "data": data},
    )
```

### Fragment template

```html
{# dashboard/fragments/some_card.html — returned by HTMX, no extends needed #}
<div class="card-panel p-4">
    <h2 class="text-lg font-semibold">{{ data.title }}</h2>
    <p class="text-2xl font-bold">{{ data.value | inr }}</p>
</div>
```

### Common HTMX patterns

```html
<!-- Lazy load on page ready -->
<div hx-get="/api/fragments/card" hx-trigger="load" hx-swap="outerHTML">
    <div class="animate-pulse h-24 bg-gray-200 rounded"></div>
</div>

<!-- Button that refreshes a section -->
<button hx-post="/api/fragments/refresh"
        hx-target="#results-table"
        hx-swap="innerHTML"
        hx-indicator="#spinner">
    Refresh
    <span id="spinner" class="htmx-indicator">...</span>
</button>

<!-- Form submit without page reload -->
<form hx-post="/items/add"
      hx-target="#items-list"
      hx-swap="beforeend"
      hx-on::after-request="this.reset()">
    ...
</form>

<!-- Delete with confirmation + row removal -->
<button hx-delete="/items/{{ item.id }}"
        hx-target="closest tr"
        hx-swap="outerHTML swap:200ms"
        hx-confirm="Delete this item?">
    Delete
</button>
```

---

## Testing — pytest unit tests

### tests/conftest.py

```python
import pytest
import asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from app.models.base import Base
from app.dependencies import get_db, get_current_user
from app.main import app


@pytest.fixture(scope="session")
def event_loop():
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()


@pytest.fixture
async def test_db():
    """Fresh in-memory SQLite per test — fast and isolated."""
    engine = create_async_engine("sqlite+aiosqlite:///:memory:", echo=False)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    session_factory = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
    async with session_factory() as session:
        yield session
    await engine.dispose()


@pytest.fixture
async def test_client(test_db):
    async def override_get_db():
        yield test_db

    app.dependency_overrides[get_db] = override_get_db
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        yield client
    app.dependency_overrides.clear()


@pytest.fixture
def mock_user():
    from app.models.user import User
    return User(id="test-123", email="test@example.com", role="user")


@pytest.fixture
async def authenticated_test_client(test_db, mock_user):
    async def override_get_db():
        yield test_db
    async def override_get_current_user():
        return mock_user

    app.dependency_overrides[get_db] = override_get_db
    app.dependency_overrides[get_current_user] = override_get_current_user
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        yield client
    app.dependency_overrides.clear()
```

### Unit test pattern

```python
# tests/unit/test_item_service.py
import pytest
from app.services.item_service import ItemService
from app.schemas.item_schema import ItemCreate

@pytest.mark.asyncio
async def test_create_item(test_db):
    service = ItemService(test_db)
    item = await service.create(ItemCreate(name="Widget", price=9.99), user_id="test-123")
    assert item.name == "Widget"
    assert item.id is not None

@pytest.mark.asyncio
async def test_get_items_empty(test_db):
    service = ItemService(test_db)
    items = await service.get_all("test-123")
    assert items == []
```

**Patching rule:** Patch at the definition module, not the import site.
```python
# CORRECT: patch where SomeClass is defined
with patch("app.services.item_service.ExternalAPI") as mock:
    ...

# WRONG: patch where it's used (often doesn't intercept)
with patch("app.routes.items.ExternalAPI") as mock:
    ...
```

---

## Testing — Playwright E2E

### tests/e2e/conftest.py

```python
import pytest
import subprocess
import time
import os
from playwright.sync_api import sync_playwright

TEST_BASE_URL = "http://localhost:8765"

@pytest.fixture(scope="session", autouse=True)
def test_server():
    """Start test server for the duration of the E2E session."""
    env = {**os.environ, "ENV_FILE": ".env.test"}
    proc = subprocess.Popen(
        ["venv/bin/python", "-m", "uvicorn", "app.main:app", "--port", "8765"],
        env=env,
    )
    time.sleep(2)  # Wait for startup
    yield proc
    proc.terminate()
    proc.wait()

@pytest.fixture(scope="session")
def browser():
    with sync_playwright() as p:
        b = p.chromium.launch(headless=True)
        yield b
        b.close()

@pytest.fixture
def page(browser):
    page = browser.new_page()
    yield page
    page.close()

@pytest.fixture
def authenticated_page(page):
    """Returns a Playwright page already logged in as test user."""
    page.goto(f"{TEST_BASE_URL}/auth/login")
    page.fill("[name=phone]", "9999999999")
    page.fill("[name=password]", "dev123")
    page.click("[type=submit]")
    page.wait_for_url(f"{TEST_BASE_URL}/dashboard")
    return page
```

### E2E test pattern

```python
# tests/e2e/test_items.py
def test_create_item(authenticated_page):
    page = authenticated_page
    page.goto(f"{TEST_BASE_URL}/items/add")
    page.fill("[name=name]", "Test Widget")
    page.fill("[name=price]", "9.99")
    page.click("[type=submit]")

    # Assert redirect and success message
    page.wait_for_url(f"{TEST_BASE_URL}/items")
    assert page.locator("text=Test Widget").is_visible()

def test_delete_item(authenticated_page):
    page = authenticated_page
    # ... setup item, then delete
    page.click("[data-testid=delete-btn]")
    page.click("text=Confirm")
    page.wait_for_selector("text=Test Widget", state="detached")
```

### pytest.ini or pyproject.toml

```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
# Run E2E tests separately: pytest tests/e2e/ -v
# Unit tests: pytest tests/unit/ -v
```

---

## Linux Deployment

### scripts/setup_pi.sh

```bash
#!/bin/bash
# One-time setup on Raspberry Pi / BeagleBone / Ubuntu server
# Usage: MYAPP_USER=$(whoami) curl -sSL https://.../setup_pi.sh | bash

set -e

MYAPP_USER="${MYAPP_USER:-pi}"
APP_DIR="/home/${MYAPP_USER}/myapp"
REPO_URL="https://github.com/yourname/myapp.git"

# 1. System packages
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-pip python3-venv python3-dev \
    git sqlite3 libffi-dev libssl-dev build-essential curl

# 2. Clone
[ -d "$APP_DIR" ] && (cd "$APP_DIR" && git pull) || git clone "$REPO_URL" "$APP_DIR"
cd "$APP_DIR"

# 3. Virtual environment
# Use --system-site-packages on Pi to inherit compiled numpy/pandas from apt
python3 -m venv --system-site-packages venv
source venv/bin/activate

# 4. Dependencies
pip install --upgrade pip
pip install -r requirements.txt
# OR: pip install -e "." for editable installs

# 5. Directories
mkdir -p data logs backups

# 6. Env file
if [ ! -f ".env.prod" ]; then
    cp .env.prod.example .env.prod
    echo "IMPORTANT: Edit $APP_DIR/.env.prod — set SECRET_KEY, JWT_SECRET, etc."
    echo "  python3 -c \"import secrets; print(secrets.token_urlsafe(32))\""
fi

# 7. DB migrations
ENV_FILE=.env.prod alembic upgrade head

# 8. Systemd service
./scripts/setup_services.sh
# Then follow the printed install instructions

echo "Setup complete! Start with: sudo systemctl start myapp"
```

### requirements.txt vs pyproject.toml on Pi

For Pi deployment, generate a flat `requirements.txt` from pyproject.toml:
```bash
# On dev machine:
pip install -e "." --dry-run 2>&1 | grep "Would install" > requirements.txt
# OR simply:
pip freeze > requirements.txt
```
Pi's `pip install -r requirements.txt` is simpler than `pip install -e "."`.

---

## systemd Service File

### myapp.service (commit to repo root)

```ini
# ─── QUICK SETUP ────────────────────────────────────────────────────
# sed -i 's|/home/pi/myapp|/home/YOUR_USER/YOUR_FOLDER|g' myapp.service
# sed -i 's|User=pi|User=YOUR_USER|g' myapp.service
#
# INSTALL:
#   sudo cp myapp.service /etc/systemd/system/
#   sudo systemctl daemon-reload && sudo systemctl enable myapp && sudo systemctl start myapp
# ────────────────────────────────────────────────────────────────────

[Unit]
Description=MyApp Web Server
After=network.target

[Service]
Type=simple
User=pi
Group=pi

WorkingDirectory=/home/pi/myapp
Environment="PATH=/home/pi/myapp/venv/bin:/usr/bin"
Environment="ENV_FILE=.env.prod"

ExecStart=/home/pi/myapp/venv/bin/uvicorn \
    app.main:app \
    --host 127.0.0.1 \
    --port 8000 \
    --log-level info

Restart=on-failure
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=3

# Security hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/home/pi/myapp/data /home/pi/myapp/logs /home/pi/myapp/backups
PrivateTmp=true

# Resource limits (prevents runaway memory on Pi)
MemoryMax=512M
MemoryHigh=400M
CPUQuota=80%

[Install]
WantedBy=multi-user.target
```

### scripts/setup_services.sh (auto-configures for current user)

```bash
#!/bin/bash
# Run from repo root: ./scripts/setup_services.sh
set -e

CURRENT_USER=$(whoami)
APP_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
mkdir -p "$APP_DIR/deploy"

cat > "$APP_DIR/deploy/myapp.service" << EOF
[Unit]
Description=MyApp Web Server
After=network.target

[Service]
Type=simple
User=$CURRENT_USER
Group=$CURRENT_USER
WorkingDirectory=$APP_DIR
Environment="PATH=$APP_DIR/venv/bin:/usr/bin"
Environment="ENV_FILE=.env.prod"
ExecStart=$APP_DIR/venv/bin/uvicorn app.main:app --host 127.0.0.1 --port 8000 --log-level info
Restart=on-failure
RestartSec=5
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=$APP_DIR/data $APP_DIR/logs $APP_DIR/backups
PrivateTmp=true
MemoryMax=512M
CPUQuota=80%

[Install]
WantedBy=multi-user.target
EOF

echo "Created deploy/myapp.service for user: $CURRENT_USER"
echo ""
echo "Install with:"
echo "  sudo cp $APP_DIR/deploy/myapp.service /etc/systemd/system/"
echo "  sudo systemctl daemon-reload && sudo systemctl enable myapp && sudo systemctl start myapp"
```

---

## Deploy Script

### scripts/deploy.sh

```bash
#!/bin/bash
# Safe deploy: pull → dependencies → migrate → restart
# Run from the app directory: ./scripts/deploy.sh
set -e

GREEN='\033[0;32m'; YELLOW='\033[1;33m'; RED='\033[0;31m'; NC='\033[0m'
ok()   { echo -e "${GREEN}  ✓ $*${NC}"; }
warn() { echo -e "${YELLOW}  ⚠ $*${NC}"; }
err()  { echo -e "${RED}  ✗ $*${NC}"; }

[ -d ".git" ] || (err "Run from the app directory" && exit 1)

APP_DIR="$(pwd)"
SERVICE_NAME="myapp"

echo "══════════════════════════════════"
echo "  Deploy: $SERVICE_NAME"
echo "══════════════════════════════════"

# [1] Pull code
echo "[1/5] Pulling code..."
git pull
chmod +x scripts/*.sh
ok "Code updated."

# [2] Stop service (graceful then force)
echo "[2/5] Stopping service..."
if systemctl is-active --quiet "$SERVICE_NAME"; then
    sudo systemctl stop "$SERVICE_NAME"
    for i in $(seq 1 15); do
        systemctl is-active --quiet "$SERVICE_NAME" || { ok "Stopped after ${i}s."; break; }
        sleep 1
    done
    systemctl is-active --quiet "$SERVICE_NAME" && \
        sudo systemctl kill --signal=SIGKILL "$SERVICE_NAME" && sleep 2
fi
ok "Service stopped."

# [3] Update dependencies
echo "[3/5] Updating dependencies..."
source venv/bin/activate
pip install -r requirements.txt --quiet
ok "Dependencies updated."

# [4] CSS check (pre-built on dev machine)
[ -f "static/css/output.css" ] && ok "CSS OK." || \
    warn "output.css missing — run 'npm run build:css' on dev machine and commit."

# [5] Migrate database
echo "[4/5] Running migrations..."
ENV_FILE=.env.prod alembic upgrade head
ok "Migrations applied."

# [6] Start service
echo "[5/5] Starting service..."
sudo systemctl start "$SERVICE_NAME"
sleep 3
systemctl is-active --quiet "$SERVICE_NAME" && ok "$SERVICE_NAME started." || \
    (err "Service failed to start!" && journalctl -u "$SERVICE_NAME" -n 20 --no-pager && exit 1)

echo ""
echo "══════════════════════════════════"
echo "  DEPLOY SUCCESSFUL"
echo "══════════════════════════════════"
echo "  Logs: journalctl -u $SERVICE_NAME -f"
```

---

## CLAUDE.md Template

Copy this into your new project's `CLAUDE.md`:

```markdown
# MyApp

[One-line project description]

## Quick Start

```powershell
.\setup.bat          # First time setup
.\start_dev.bat      # Dev server (port 8000)
.\run_e2e_tests.bat  # E2E tests (port 8765)
```

## Environment Files

| File | Database | Purpose |
|------|----------|---------|
| `.env.dev` | `data/myapp_dev.db` | Review test data |
| `.env.test` | `test_e2e.db` | E2E test isolation |
| `.env.prod` | `data/myapp.db` | Real data |

## Dev Login
Email/Phone: `9999999999`, Password: `dev123`

## Code Architecture

```
app/
├── models/         # SQLAlchemy models
├── schemas/        # Pydantic schemas
├── repositories/   # Database CRUD
├── services/       # Business logic
├── routes/         # FastAPI route handlers
│   └── api_fragments.py   # HTMX partial HTML endpoints
└── templates/
    ├── base.html
    ├── components/
    └── {feature}/
        ├── index.html
        └── fragments/     # HTMX partial templates

tests/
├── conftest.py     # test_db, test_client, mock_user, authenticated_test_client
├── unit/
└── e2e/
```

## Key Patterns

- **DB session:** `async with get_db_session() as db:` or `Depends(get_db)`
- **HTMX fragments:** Routes in `api_fragments.py`, templates in `*/fragments/`
- **CSS:** Pre-built `static/css/output.css` committed to git — run `npm run build:css` after template changes
- **Migrations:** `ENV_FILE=.env.dev alembic revision --autogenerate -m "description"`
- **Patching in tests:** Patch at definition module `app.services.foo`, not import site

## Deployment (Pi/Linux)

```bash
# First time
./scripts/setup_pi.sh

# After code push
./scripts/deploy.sh
```

## Development Principles

- Read existing code before modifying
- Fix underlying logic, not just make tests pass
- Prefer editing existing files over creating new ones
- Add comments only where logic is non-obvious
- Use `.tmp/` for throwaway scripts (gitignored)
```

---

## .gitignore

```gitignore
# Python
venv/
__pycache__/
*.py[cod]
*.egg-info/
dist/
.pytest_cache/
.coverage
htmlcov/

# Databases
data/
*.db
*.db-shm
*.db-wal

# Logs & backups
logs/
backups/
*.log

# Environment (never commit secrets)
.env
.env.dev
.env.test
.env.prod

# Node (only package-lock if you want reproducible builds)
node_modules/

# Scratch
.tmp/

# Playwright
test-results/
playwright-report/

# OS
.DS_Store
Thumbs.db
```

**Commit:** `.env.example`, `.env.prod.example`, `static/css/output.css`  
**Never commit:** `.env.*` (except examples), `data/`, `logs/`, `backups/`
```
