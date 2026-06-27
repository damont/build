# Phase 01: Project Setup

Scaffold the project structure, configure dependencies, and make key decisions (deployment path, ports).

---

## Core Principles

1. **API-first** — Backend is a standalone REST API, testable without the frontend
2. **Separate concerns** — Backend and frontend are independent build targets, sharing only an HTTP contract
3. **Mobile-first** — Phone screens first, enhance with Tailwind responsive prefixes
4. **URL-driven navigation** — Every view has its own URL; browser back/forward and deep-linking work
5. **Docker Compose for local dev** — Every app runs locally via `docker-compose.yml`
6. **Keep it simple** — No ORMs with migrations, no Redux, no GraphQL

---

## Tech Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Language (backend) | Python >= 3.12 | Most stable release available |
| Package manager | uv | Fast, lockfile-based, replaces pip/poetry |
| Web framework | FastAPI | Async, auto-generates OpenAPI docs |
| Database | MongoDB | Via async Motor driver |
| ODM | Beanie | Pydantic-based document models |
| Auth | JWT (PyJWT) + Argon2 (argon2-cffi) | See Phase 02 |
| Config | pydantic-settings | Reads from `.env` files |
| Frontend framework | React 19 + TypeScript | Vite for bundling |
| Styling | Tailwind CSS v4 | Utility-first, no custom CSS framework |
| Containerization | Docker + Docker Compose | Multi-stage builds |
| Deployment target | Raspberry Pi, Azure, **or** AWS | See Phase 08 |

---

## Choose Deployment Path

Decide now — it affects project structure and CI/CD:

| | **Path A: Raspberry Pi** | **Path B: Azure** | **Path C: AWS** |
|---|---|---|---|
| **Best for** | Internal/family apps, low traffic | Public-facing apps, external users | Public apps, cheapest managed/idle |
| **Hosting** | Self-hosted Docker + Cloudflare Tunnel | Azure Container Apps (scale-to-zero) | Lambda + API Gateway + CloudFront |
| **Async work** | In-process / host | Always-on Redis + KEDA worker | SNS → per-handler Lambdas (nothing always-on) |
| **Database** | MongoDB on Pi (shared) | MongoDB Atlas | MongoDB Atlas |
| **Cost** | Free (already running) | ~$35–50/mo (always-on Redis) | ~$0–5/mo (pay-per-request) |
| **Examples** | `calendarapp`, `track` | `toolshed` | `recall` |

---

## Project Structure

Create this directory layout:

```
<appname>/
├── api/
│   ├── __init__.py
│   ├── main.py              # FastAPI app, lifespan, router includes
│   ├── config.py            # pydantic-settings, reads .env
│   ├── routes/
│   │   ├── auth.py          # Login/register endpoints
│   │   └── <resource>.py    # One file per resource
│   ├── schemas/
│   │   ├── orm/             # Beanie Document models (database shape)
│   │   │   ├── user.py
│   │   │   ├── password_reset.py
│   │   │   └── <model>.py
│   │   └── dto/             # Request/response Pydantic models
│   │       └── <model>.py
│   ├── services/            # Business logic
│   │   ├── email.py         # SMTP email service (password reset)
│   │   └── <service>.py
│   └── utils/
│       └── auth.py          # JWT creation/validation, password hashing
├── frontend/
│   ├── src/
│   │   ├── main.tsx
│   │   ├── App.tsx
│   │   ├── index.css
│   │   ├── api/
│   │   │   └── client.ts
│   │   ├── hooks/
│   │   │   └── useRouter.ts
│   │   ├── context/
│   │   │   └── AuthContext.tsx
│   │   ├── types/
│   │   │   └── index.ts
│   │   └── components/
│   │       ├── auth/        # Login, Register, ForgotPassword, ResetPassword, LandingPage
│   │       ├── layout/
│   │       └── <feature>/
│   ├── index.html
│   ├── package.json
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── nginx.conf
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml
├── uv.lock
├── .env.example
└── .gitignore
```

---

## Python Dependencies

Create `pyproject.toml`:

```toml
[project]
name = "myapp"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.109.0",
    "uvicorn[standard]>=0.27.0",
    "beanie>=1.25.0",
    "motor>=3.3.0",
    "pydantic>=2.5.0",
    "pydantic-settings>=2.1.0",
    "PyJWT>=2.8.0",
    "argon2-cffi>=23.1.0",
    "python-multipart>=0.0.6",
    "httpx>=0.26.0",
]

[tool.uv]
dev-dependencies = [
    "pytest>=7.4.0",
    "anyio[trio]>=4.0.0",
    "pytest-asyncio>=0.23.0",
    "httpx>=0.26.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["api"]
```

Install and lock:

```bash
uv sync
```

---

## Configuration

Create `api/config.py`:

```python
from typing import Optional
from functools import lru_cache
from pydantic import field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8")

    mongodb_url: str = "mongodb://localhost:27017"
    mongodb_db_name: str = "myapp"
    jwt_secret: str = "change-me"
    jwt_algorithm: str = "HS256"
    jwt_expire_minutes: int = 10080  # 7 days

    # Email / SMTP (Gmail — for password reset)
    smtp_email: Optional[str] = None
    smtp_app_password: Optional[str] = None
    smtp_host: str = "smtp.gmail.com"
    smtp_port: int = 587
    password_reset_expire_minutes: int = 60

    # Frontend base URL (used in email links)
    frontend_base_url: str = "https://localhost:8095"

    @field_validator("frontend_base_url")
    @classmethod
    def _ensure_scheme(cls, v: str) -> str:
        """Auto-prepend https:// if no scheme — Apple Mail won't recognize scheme-less URLs as clickable."""
        if v and not v.startswith(("http://", "https://")):
            return f"https://{v}"
        return v

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

**Note:** Use `SettingsConfigDict` instead of inner `class Config` — pydantic-settings v2 deprecates the inner class approach.

---

## Environment Variables

Create `.env.example` (checked into git):

```
MONGODB_URL=mongodb://localhost:27017
MONGODB_DB_NAME=myapp
JWT_SECRET=change-me-in-production
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=10080

# Email / SMTP (Gmail — for password reset emails)
SMTP_EMAIL=your-email@gmail.com
SMTP_APP_PASSWORD=your-app-password
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
PASSWORD_RESET_EXPIRE_MINUTES=60

# Frontend base URL (used in password reset email links)
FRONTEND_BASE_URL=https://localhost:8095
```

Copy to `.env` and fill in real values. Add `.env` to `.gitignore`.

**Gmail App Password setup:** Go to Google Account > Security > 2-Step Verification > App Passwords, generate one for your app. For deployment, add `SMTP_EMAIL` and `SMTP_APP_PASSWORD` as GitHub Secrets.

---

## Database Design Diagram

Before writing any Beanie models, create a draw.io (`.drawio`) diagram in the app root:

- Each MongoDB document (Beanie `Document` subclass) as a box with its fields and types
- Nested models (`BaseModel` subclasses) shown as contained boxes or with composition arrows
- Relationships between documents (e.g., `user_id` references) shown as association arrows
- Collection names labeled on each document box

Save as `<appname>/database-design.drawio` and keep it updated as the schema evolves.

---

## Checklist

- [ ] Directory structure created
- [ ] `pyproject.toml` created with all dependencies
- [ ] `uv sync` runs successfully
- [ ] `api/config.py` created
- [ ] `.env.example` created
- [ ] `.env` created (from example, with local values)
- [ ] `.gitignore` includes `.env`, `__pycache__/`, `node_modules/`, `.venv/`
- [ ] Database design diagram created (`database-design.drawio`)
- [ ] `api/__init__.py` exists (empty file)
