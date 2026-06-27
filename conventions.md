# Build Conventions — Audit Rubric

The checkable rules a codebase should follow, distilled from this skill's phase docs (`${CLAUDE_SKILL_DIR}/phases/*.md`, which are canonical). Used by the `review` and `assess` modes. Each rule has a one-line **expectation** and a **detect** hint (a file to open or a `grep`/`glob` to run). Rate findings per the Severity & Output section of `SKILL.md` (Violation / Gap / Note / Conforms).

Many capabilities are **optional** — see §0. If a capability isn't used, its whole section is **N/A**, not a violation.

---

## 0. Applicability — decide what's in scope first

These are builder decisions; detect which were taken before auditing the rest.

| Capability | In use when… | If used, audit |
|---|---|---|
| **Deployment path** | always one of Pi / Azure / AWS | §6 (the matching path) |
| **Google sign-in / email verification** | `google_sub` in user model, or `GoogleOAuthProvider` in frontend | §7.3 |
| **Background workers** | `worker/` dir, or `events.py` + `publish()` calls | §7.1 |
| **File storage** | `api/services/storage.py` exists | §7.2 |
| **Agent auth / API** | `/api/auth/agent-token` route, or `agentSkill` in frontend | §7.4 |
| **GitHub feedback** | `feedback_github_*` config or `/api/feedback` route | §7.5 |

Baseline (email/password auth, CRUD, tests, Docker) always applies (§1–§6).

---

## 1. Project layout & root files

- **Root has `api/`, `frontend/`, `Dockerfile`, `docker-compose.yml`, `pyproject.toml`, `uv.lock`, `.env.example`, `.gitignore`.** · detect: `ls` the root.
- **`.env.example` is tracked; `.env` is git-ignored.** · detect: `.env.example` present, `.gitignore` lists `.env`, `__pycache__/`, `node_modules/`, `.venv/`.
- **No `.env`, secrets, or `node_modules/`/`.venv/` committed.** · detect: `git ls-files | grep -E '\.env$|node_modules|\.venv'` is empty.
- **(If new app) `database-design.drawio` at root.** · detect: glob `*.drawio`. (Gap, not Violation, if absent.)

---

## 2. Backend — FastAPI / Python

### 2.1 Package structure
- **`api/` is a package with these subpackages, each with `__init__.py`:** `routes/`, `schemas/orm/`, `schemas/dto/`, `services/`, `utils/`. · detect: `find api -name __init__.py`.
- **One route file per resource** in `api/routes/` (`auth.py`, `<resource>.py`). · detect: `ls api/routes`.

### 2.2 Config (`api/config.py`)
- **`Settings(BaseSettings)` using `SettingsConfigDict(env_file=".env", ...)`** — *not* an inner `class Config`. · detect: `grep -n 'SettingsConfigDict\|class Config' api/config.py`.
- **Cached accessor `@lru_cache def get_settings()`.** · detect: `grep get_settings api/config.py`.
- **Env var names are UPPER_SNAKE.** · detect: scan field names / `.env.example`.

### 2.3 App entry (`api/main.py`)
- **Single `app = FastAPI(...)` with an `@asynccontextmanager` lifespan.** · detect: `grep -n 'FastAPI(\|asynccontextmanager' api/main.py`.
- **`init_beanie(document_models=[...])` lists EVERY Beanie Document.** · detect: compare the list to `grep -rl 'Document)' api/schemas/orm`. A missing model is a **Violation** (queries fail silently).
- **`CORSMiddleware` added; routers via `app.include_router(r, prefix="/api/<resource>", tags=[...])`.** · detect: `grep include_router api/main.py`.
- **`GET /api/health` → `{"status": "ok"}`.** · detect: `grep -n health api/main.py`.
- **`logging.basicConfig(...)` configured once here.** · detect: `grep basicConfig api/main.py`.

### 2.4 Database (Beanie + MongoDB)
- **ORM models in `api/schemas/orm/`, each `class X(Document)` with inner `Settings` `name = "<collection>"`.** · detect: `grep -rn 'class.*Document\|name = ' api/schemas/orm`.
- **Indexed unique fields use `Indexed(...)`; sparse/compound via `pymongo.IndexModel`.** · detect: `grep -rn 'Indexed\|IndexModel' api/schemas/orm`.
- **Timestamps `created_at` / `updated_at` on documents.** · detect: `grep -rn created_at api/schemas/orm`.
- **Nested data modeled as embedded `BaseModel`, not extra collections** (keep it simple). · review models.

### 2.5 Auth (`api/utils/auth.py`, `api/routes/auth.py`)
- **Argon2 (`argon2-cffi`) for passwords — not bcrypt; PyJWT for tokens.** · detect: `grep -n 'argon2\|bcrypt\|jwt' api/utils/auth.py`.
- **Exports `hash_password`, `verify_password`, `create_access_token(subject, expires_delta=None)`, `get_current_user` (HTTPBearer dependency).** · detect: `grep -n 'def ' api/utils/auth.py`.
- **User model has `email: Indexed(str, unique=True)`, `hashed_password`, a display name.** · detect: `api/schemas/orm/user.py`.
- **Endpoints: `POST /register`, `POST /login`, `GET /me` (protected), `POST /forgot-password`, `POST /reset-password`.** · detect: `grep -n '@router' api/routes/auth.py`.
- **`PasswordResetToken` model (`token`, `user_id`, `expires_at`, `used_at`) + `api/services/email.py::send_password_reset_email`.** · detect: those files. Dev logs the link; prod sends SMTP; HTML email is table/inline-style (Apple-Mail-safe).

### 2.6 DTOs & routes (per resource)
- **DTOs in `api/schemas/dto/<resource>.py`: `<R>Create`, `<R>Update` (all Optional), `<R>Response` (with `id: str`).** · detect: `ls api/schemas/dto`, scan a file.
- **Route module has `router = APIRouter()` and standard CRUD** (`GET /{id}`, `GET /` with `skip`/`limit`, `POST /`→201, `PUT /{id}`, `DELETE /{id}`→204). · detect: `grep -n '@router' api/routes/<r>.py`.
- **Protected endpoints use `Depends(get_current_user)` and verify ownership (user can't touch another user's rows).** · detect: `grep -n 'get_current_user\|user_id' api/routes/*.py`. Missing ownership check = **Violation**.
- **Routes return DTOs (via a `<thing>_to_response` helper), never raw Beanie docs.** · detect: `grep -n 'response_model\|_to_response' api/routes/*.py`.

### 2.7 Logging
- **Module-level `logger = logging.getLogger(__name__)` in modules that log; INFO level.** · detect: `grep -rn 'getLogger(__name__)' api`.
- **Logs state changes & auth events; does NOT log bodies/passwords/PII or every read.** · review log calls.

---

## 3. Backend testing (pytest)

- **`tests/` has `conftest.py` + `test_<resource>.py` (roughly one per route file).** · detect: `ls tests`.
- **Test DB name carries a `_test` suffix and is dropped per test (autouse fixture).** · detect: `grep -n '_test\|drop' tests/conftest.py`. Hitting the real DB name = **Violation**.
- **Fixtures: `anyio_backend`→`"asyncio"`, `client` (`AsyncClient` + `ASGITransport(app=app)`), `authenticated_client` (real user + `create_access_token` + Bearer header).** · detect: `grep -n '@pytest.fixture\|ASGITransport\|authenticated_client' tests/conftest.py`.
- **`[tool.pytest.ini_options] asyncio_mode = "auto"` in `pyproject.toml`.** · detect: `grep -n asyncio_mode pyproject.toml`.
- **Covers auth flow, CRUD per resource, and cross-user authorization.** · review test names. Skips pure-validation/library tests.

---

## 4. Frontend — React + TS + Vite

### 4.1 Structure (`frontend/`)
- **`src/` has `main.tsx`, `App.tsx`, `index.css`, `api/client.ts`, `hooks/useRouter.ts`, `context/AuthContext.tsx`, `types/index.ts`, `components/`.** · detect: `ls frontend/src`.
- **Root has `index.html`, `package.json`, `vite.config.ts`, `tsconfig.json`, `nginx.conf`.** · detect: `ls frontend`.

### 4.2 Config
- **`vite.config.ts`: `react()` + `tailwindcss()` plugins, dev port **8095**, `/api` proxy → `http://localhost:8020` (`changeOrigin`), and a `test` block (`jsdom`, `globals`, `setupFiles`).** · detect: `grep -n '8095\|8020\|proxy\|jsdom' frontend/vite.config.ts`.
- **`tsconfig.json`: `strict: true`, `jsx: "react-jsx"`, `target ES2020`, `noUnusedLocals/Parameters`.** · detect: open it.

### 4.3 Styling
- **`index.css` does `@import "tailwindcss";` and defines `:root` CSS variables (dark theme).** · detect: `grep -n 'tailwindcss\|:root' frontend/src/index.css`.
- **No per-component CSS files; styling via Tailwind classes + `var(--…)`.** · detect: `find frontend/src -name '*.css'` should be ~just `index.css`.

### 4.4 API client (`src/api/client.ts`)
- **Single client with `get/post/put/delete<T>()`; token in `localStorage` (`getToken/setToken/clearToken`); `Authorization: Bearer` on requests; on 401 → `clearToken()` + throw (no `window.location.reload()`).** · detect: `grep -n 'localStorage\|Bearer\|401\|reload' frontend/src/api/client.ts`.

### 4.5 Auth context (`src/context/AuthContext.tsx`)
- **Exports `AuthProvider` + `useAuth()`; provides `isAuthenticated/isLoading/user/login/logout`; restores session via `/api/auth/me` on mount; auto-logout on 401.** · detect: `grep -n 'AuthProvider\|useAuth\|/api/auth/me' frontend/src/context/AuthContext.tsx`.

### 4.6 Routing (`src/hooks/useRouter.ts`, `App.tsx`)
- **`useRouter()` returns `{ path, navigate }`, uses `history.pushState`/`replaceState`, listens to `popstate`; `matchPath(pattern, path)` extracts params.** · detect: `grep -n 'pushState\|popstate\|matchPath' frontend/src/hooks/useRouter.ts`.
- **`App.tsx`: URL is the source of truth for the current view (derive from `path`, not `useState`); redirect unauth'd users off protected routes; loading state during session check.** · detect: `grep -n 'useRouter\|useAuth\|isLoading' frontend/src/App.tsx`. Selected-entity IDs held in `useState` instead of the URL = **Note/Gap**.

### 4.7 Entry & components
- **`main.tsx` wraps `<StrictMode>` + `<AuthProvider>` (and `<GoogleOAuthProvider>` only if `VITE_GOOGLE_CLIENT_ID` set); imports `index.css`.** · detect: `grep -n 'StrictMode\|AuthProvider\|GoogleOAuthProvider' frontend/src/main.tsx`.
- **Components grouped by feature (`components/auth/`, `components/<feature>/`), one default export + typed `Props` interface per file.** · detect: `ls frontend/src/components` (dirs, not loose `.tsx`).
- **Mobile-first Tailwind: base = phone, `sm:`/`md:`/`lg:` enhance; interactive elements ≥44px touch target.** · detect: `grep -rn 'sm:\|min-h-\[44px\]\|flex-col sm:flex-row' frontend/src/components | head`.
- **Nav uses `<a href>` + `e.preventDefault(); navigate(...)` (not bare `<button>`)** so open-in-new-tab works. · detect: `grep -rn '<a href' frontend/src/components | head`.

---

## 5. Frontend testing (Vitest)

- **`src/test/setup.ts` imports `@testing-library/jest-dom`; `package.json` has `"test": "vitest run"`.** · detect: those files.
- **Tests co-located (`Component.test.tsx` beside `Component.tsx`, `client.test.ts` beside `client.ts`).** · detect: `find frontend/src -name '*.test.ts*'`.
- **Covers API client (mock fetch, headers, 401), logic-bearing components, context flows; skips pure presentation.** · review test names.

---

## 6. Docker & deployment

### 6.1 Common (all paths)
- **Root `Dockerfile` is multi-stage: `backend` (`python:3.12-slim`, `uv sync --frozen --no-dev`, uvicorn :8020), `frontend-build` (`node:20-slim`, `npm ci` + `npm run build`), `frontend` (`nginx:alpine`, serves `dist/`). No node in the final image.** · detect: `grep -n 'AS backend\|AS frontend-build\|AS frontend\|npm ci' Dockerfile`.
- **`docker-compose.yml`: `api` (target `backend`, :8020, `extra_hosts host.docker.internal`, `restart: unless-stopped`) + `frontend` (target `frontend`, :8095, `depends_on: [api]`).** · detect: open it.
- **`frontend/nginx.conf`: SPA fallback `try_files $uri $uri/ /index.html`, `/api/` → `proxy_pass http://api:8020`, 1-year cache on static assets.** · detect: `grep -n 'try_files\|proxy_pass\|expires' frontend/nginx.conf`.

### 6.2 Path A — Raspberry Pi
- Standard `Dockerfile` + compose suffice (no extra files). Each app uses a distinct `MONGODB_DB_NAME` against the shared on-Pi Mongo. Infra (nginx vhost, Cloudflare tunnel/DNS, auto-deploy script) lives outside the repo.

### 6.3 Path B — Azure Container Apps
- **Extra files: `Dockerfile.api`, `Dockerfile.frontend`, `frontend/default.conf.template` (nginx `${API_HOST}` envsubst, `NGINX_ENVSUBST_FILTER=API_HOST`).** · detect: `ls Dockerfile.api Dockerfile.frontend frontend/default.conf.template`.
- **`infra/` Terraform (azurerm): env-scoped names `*-${environment}`, internal Redis app (always-on), frontend(external)/api(internal)/worker(no ingress, KEDA `redis-streams`) apps, all `min_replicas = 0` except Redis.** · detect: `grep -rn 'min_replicas\|redis-streams' infra`.
- **Worker handles blocking-read `TimeoutError` + `socket_keepalive=True`.** · detect: `grep -rn 'socket_keepalive\|TimeoutError' worker`.

### 6.4 Path C — AWS Lambda
- **Extra files: `docker/Dockerfile.api` (Lambda Web Adapter copied to `/opt/extensions`, `AWS_LWA_*` env, uvicorn :8000) + `docker/Dockerfile.worker` (`awslambdaric` ENTRYPOINT, handler CMD overridable).** · detect: `grep -n 'lambda-adapter\|awslambdaric' docker/Dockerfile.*`.
- **`infra/` Terraform (aws): API Lambda `package_type = "Image"` behind an **API Gateway HTTP API** (not a Function URL), processor Lambdas via `for_each`, `sns.tf` topic + subscriptions, `iam.tf` shared role with `sns:Publish`, us-east-1 `acm.tf`, `frontend.tf` S3(+OAC)+CloudFront(2 origins, SPA-router function), `infra/bootstrap/` (state bucket, ECR, OIDC role).** · detect: `ls infra`; `grep -n 'package_type\|apigatewayv2\|sns:Publish' infra/*.tf`.
- **SNS handlers (`handlers/*.py`) unwrap the SNS envelope and call shared `core/` logic; `EVENTS_BACKEND` switches `local` (dev/test) vs `sns`.** · detect: `grep -rn 'Sns\|EVENTS_BACKEND' handlers api`.
- **`deploy-dev.yml` (PR→main) + `deploy-prod.yml` (push→main) via GitHub OIDC: build+push to ECR, `terraform apply`, `s3 sync`, CloudFront invalidation; per-env state key `…/<env>.tfstate`.** · detect: `ls .github/workflows`; `grep -n 'oidc\|id-token\|tfstate' .github/workflows/*`.
- **`ANTHROPIC_MODEL` configurable (not a hardcoded model id).** · detect: `grep -rn 'claude-' api core handlers` for hardcoded ids → **Violation**.

---

## 7. Optional capabilities (audit only if in use — see §0)

### 7.1 Background workers
- `worker/main.py` event loop + `@handles("<topic>")` handlers in `worker/handlers/`; events are facts (`x_created`), not commands; one worker, all topics. `events.py::publish()` is fire-and-forget; API returns immediately. `redis_url` in config; `redis` + `worker` services in compose; `redis[hiredis]` dep added **only** when used. · detect: `ls worker`; `grep -rn '@handles\|publish(' worker api`.

### 7.2 File storage
- `api/services/storage.py` with abstract `StorageBackend` + `LocalStorageBackend` / `AzureBlobStorageBackend`, chosen by `STORAGE_BACKEND`; backend picked in lifespan; structured keys `{project}/{resource}/{file}`, never hardcoded paths in routes; `azure-storage-blob[aio]` dep **only** for Azure. · detect: `grep -n 'StorageBackend\|STORAGE_BACKEND' api/services/storage.py api/config.py`.

### 7.3 Google sign-in / email verification
- User model gains `google_sub: Optional[str]` (sparse unique), `email_verified: bool = True`, `hashed_password: Optional[str]`; `EmailVerificationToken` model; endpoints `POST /api/auth/google`, `/verify-email`, `/resend-verification`; `/login` 403s unverified (`email_not_verified`). Frontend: conditional `GoogleOAuthProvider`, `GoogleLogin` button, `VerifyEmail` route `/verify-email/:token`. `VITE_GOOGLE_CLIENT_ID` is a frontend **build arg** (in Dockerfile). · detect: `grep -rn 'google_sub\|/google\|GoogleOAuthProvider\|VITE_GOOGLE_CLIENT_ID' api frontend Dockerfile*`.

### 7.4 Agent auth & API
- `POST /api/auth/agent-token` (protected, `expires_in_days` 1–365) using `create_access_token(expires_delta=…)`; OpenAPI exposed (`docs_url="/api/agent"`, `openapi_url="/api/openapi.json"`, `GET /api/schema`); Profile page generates/downloads a token + setup skill via `frontend/src/lib/agentSkill.ts` (`agentSkill()` uses `<base>` placeholders, embeds no paths/token, includes a 401/403/404/409/422/5xx failure table; covered by `agentSkill.test.ts`). · detect: `grep -rn 'agent-token\|openapi_url\|agentSkill' api frontend`.

### 7.5 GitHub feedback (Phase 09)
- `feedback_github_token` / `_repo_owner` / `_repo_name` config, `POST /api/feedback` route → GitHub Issues, and a feedback modal component. · detect: `grep -rn 'feedback_github\|/api/feedback' api frontend`.

---

## 8. Cross-cutting

- **`.env.example` (tracked) lists all env vars in UPPER_SNAKE; backend reads them via pydantic-settings; frontend build-time vars are `VITE_*`.** · detect: open `.env.example`; `grep env_file api/config.py`.
- **HTTP status codes used correctly** (201/204/400/401/403/404/409/422/5xx) via `HTTPException`; frontend try/catches API calls and auto-logs-out on 401. · review.
- **Core principles upheld:** API-first, separate backend/frontend, mobile-first, URL-driven nav, Docker-compose local dev, "keep it simple" (no migration-heavy ORM, no Redux, no GraphQL). · holistic check — flag clear violations (e.g. Redux present = **Note/Gap** against the stated principle).
