# Phase 02: Backend Foundation

Build the FastAPI app skeleton with authentication and a health check. By the end of this phase, you can register a user, log in, and call authenticated endpoints.

**Prerequisite:** Phase 01 complete (project scaffolded, dependencies installed, config in place).

---

## FastAPI App Entry Point

Create `api/main.py`:

```python
import logging
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from motor.motor_asyncio import AsyncIOMotorClient
from beanie import init_beanie

from api.config import get_settings
from api.schemas.orm.user import User
from api.schemas.orm.password_reset import PasswordResetToken
from api.routes import auth

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
logger = logging.getLogger(__name__)

@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = get_settings()
    client = AsyncIOMotorClient(settings.mongodb_url)
    await init_beanie(
        database=client[settings.mongodb_db_name],
        document_models=[User, PasswordResetToken],  # Add all Beanie documents here as they're created
    )
    logger.info("Connected to MongoDB database: %s", settings.mongodb_db_name)
    yield
    client.close()
    logger.info("Disconnected from MongoDB")

app = FastAPI(lifespan=lifespan)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(auth.router, prefix="/api/auth", tags=["auth"])

@app.get("/api/health")
async def health_check():
    return {"status": "ok"}
```

Key points:
- Logging is configured once here, before the app is created
- Beanie is initialized during the lifespan — register every Document model in `document_models`
- Add new routers with `app.include_router()` as resources are created in Phase 03

---

## User Model

Create `api/schemas/orm/user.py`:

```python
from datetime import datetime, timezone
from typing import Optional
from beanie import Document, Indexed
from pydantic import Field

class User(Document):
    email: Indexed(str, unique=True)
    display_name: str
    hashed_password: str
    is_active: bool = True
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    updated_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))

    class Settings:
        name = "users"
```

Ensure `api/schemas/__init__.py` and `api/schemas/orm/__init__.py` exist (empty files).

---

## Auth Utilities

Create `api/utils/auth.py`:

```python
from datetime import datetime, timedelta, timezone
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError
from api.config import get_settings
from api.schemas.orm.user import User

ph = PasswordHasher()
security = HTTPBearer()

def hash_password(password: str) -> str:
    return ph.hash(password)

def verify_password(plain_password: str, hashed_password: str) -> bool:
    try:
        ph.verify(hashed_password, plain_password)
        return True
    except VerifyMismatchError:
        return False

def create_access_token(subject: str) -> str:
    settings = get_settings()
    expire = datetime.now(timezone.utc) + timedelta(minutes=settings.jwt_expire_minutes)
    payload = {"sub": subject, "exp": expire}
    return jwt.encode(payload, settings.jwt_secret, algorithm=settings.jwt_algorithm)

async def get_current_user(credentials: HTTPAuthorizationCredentials = Depends(security)):
    settings = get_settings()
    try:
        payload = jwt.decode(
            credentials.credentials, settings.jwt_secret,
            algorithms=[settings.jwt_algorithm]
        )
        user_id = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=401, detail="Invalid token")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")
    user = await User.get(user_id)
    if user is None:
        raise HTTPException(status_code=401, detail="User not found")
    return user
```

Key points:
- **Argon2** for password hashing (not bcrypt/passlib — bcrypt 5.x has compatibility issues)
- `get_current_user` is a FastAPI dependency used on every protected route

Ensure `api/utils/__init__.py` exists (empty file).

---

## Auth DTOs

Create `api/schemas/dto/auth.py`:

```python
from pydantic import BaseModel, EmailStr, Field

class RegisterRequest(BaseModel):
    name: str
    email: EmailStr
    password: str

class LoginRequest(BaseModel):
    email: EmailStr
    password: str

class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"

class UserResponse(BaseModel):
    id: str
    name: str
    email: str

class PasswordResetRequest(BaseModel):
    email: EmailStr

class PasswordResetConfirm(BaseModel):
    token: str
    new_password: str = Field(min_length=6)

class MessageResponse(BaseModel):
    message: str
```

Ensure `api/schemas/dto/__init__.py` exists (empty file).

---

## Auth Routes

Create `api/routes/auth.py`:

```python
import logging
import secrets
from datetime import datetime, timedelta, timezone

from fastapi import APIRouter, Depends, HTTPException
from api.config import get_settings
from api.schemas.orm.user import User
from api.schemas.orm.password_reset import PasswordResetToken
from api.schemas.dto.auth import (
    RegisterRequest, LoginRequest, TokenResponse, UserResponse,
    PasswordResetRequest, PasswordResetConfirm, MessageResponse,
)
from api.utils.auth import hash_password, verify_password, create_access_token, get_current_user
from api.services.email import send_password_reset_email

logger = logging.getLogger(__name__)
router = APIRouter()

@router.post("/register", response_model=UserResponse, status_code=201)
async def register(data: RegisterRequest):
    existing_email = await User.find_one(User.email == data.email)
    if existing_email:
        raise HTTPException(status_code=409, detail="Email already registered")
    user = User(
        email=data.email,
        display_name=data.name,
        hashed_password=hash_password(data.password),
    )
    await user.insert()
    return UserResponse(id=str(user.id), name=user.display_name, email=user.email)

@router.post("/login", response_model=TokenResponse)
async def login(data: LoginRequest):
    user = await User.find_one(User.email == data.email)
    if not user or not verify_password(data.password, user.hashed_password):
        raise HTTPException(status_code=401, detail="Invalid email or password")
    token = create_access_token(str(user.id))
    return TokenResponse(access_token=token)

@router.post("/forgot-password", response_model=MessageResponse)
async def forgot_password(data: PasswordResetRequest):
    settings = get_settings()
    message = "If that email is registered, a reset link has been sent."

    user = await User.find_one(User.email == data.email)
    if not user:
        return MessageResponse(message=message)

    # Invalidate any existing unused tokens for this user
    user_id = str(user.id)
    existing = await PasswordResetToken.find(
        PasswordResetToken.user_id == user_id,
        PasswordResetToken.used_at == None,  # noqa: E711
    ).to_list()
    for t in existing:
        t.used_at = datetime.now(timezone.utc)
        await t.save()

    # Generate new token
    token = secrets.token_urlsafe(16)
    reset_token = PasswordResetToken(
        token=token,
        user_id=user_id,
        expires_at=datetime.now(timezone.utc) + timedelta(minutes=settings.password_reset_expire_minutes),
    )
    await reset_token.insert()

    reset_url = f"{settings.frontend_base_url}/reset-password/{token}"
    try:
        await send_password_reset_email(data.email, reset_url)
    except Exception:
        logger.exception("Failed to send password reset email to %s", data.email)

    return MessageResponse(message=message)

@router.post("/reset-password", response_model=MessageResponse)
async def reset_password(data: PasswordResetConfirm):
    reset_token = await PasswordResetToken.find_one(PasswordResetToken.token == data.token)

    if not reset_token or reset_token.used_at or reset_token.expires_at < datetime.now(timezone.utc):
        raise HTTPException(status_code=400, detail="Invalid or expired reset token.")

    user = await User.get(reset_token.user_id)
    if not user:
        raise HTTPException(status_code=400, detail="Invalid or expired reset token.")

    # Mark token as used first
    reset_token.used_at = datetime.now(timezone.utc)
    await reset_token.save()

    # Update password
    user.hashed_password = hash_password(data.new_password)
    await user.save()

    return MessageResponse(message="Password has been reset. You can now sign in.")

@router.get("/me", response_model=UserResponse)
async def get_me(user=Depends(get_current_user)):
    return UserResponse(id=str(user.id), name=user.display_name, email=user.email)
```

Ensure `api/routes/__init__.py` exists (empty file).

---

## Password Reset Token Model

Create `api/schemas/orm/password_reset.py`:

```python
from datetime import datetime, timezone
from typing import Optional
from beanie import Document, Indexed
from pydantic import Field

class PasswordResetToken(Document):
    token: Indexed(str, unique=True)
    user_id: Indexed(str)
    expires_at: datetime
    used_at: Optional[datetime] = None
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))

    class Settings:
        name = "password_reset_tokens"
```

Register this model in `api/main.py` alongside the User model:

```python
document_models=[User, PasswordResetToken],  # Add all Beanie documents here
```

---

## Email Service

Create `api/services/email.py`:

```python
import asyncio
import logging
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText

from api.config import get_settings

logger = logging.getLogger(__name__)

async def send_password_reset_email(to_email: str, reset_url: str) -> None:
    settings = get_settings()

    if not settings.smtp_email or not settings.smtp_app_password:
        if "localhost" in settings.frontend_base_url:
            logger.warning("SMTP not configured — logging reset link (dev only)")
            logger.info("Password reset link for %s: %s", to_email, reset_url)
            return
        raise RuntimeError("SMTP not configured")

    msg = MIMEMultipart("alternative")
    msg["Subject"] = "Reset Your Password"
    msg["From"] = settings.smtp_email
    msg["To"] = to_email

    text = f"Reset your password by visiting:\n\n{reset_url}\n\nThis link expires in 1 hour."
    html = f"""\
<!DOCTYPE html>
<html lang="en">
<head><meta charset="utf-8"><meta name="viewport" content="width=device-width, initial-scale=1.0"><meta name="x-apple-disable-message-reformatting"></head>
<body style="margin:0; padding:20px; font-family:Arial, sans-serif; background-color:#f5f5f5;">
<table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="max-width:480px; margin:0 auto; background:#ffffff; border-radius:8px; padding:32px;">
<tr><td>
<h2 style="margin:0 0 16px 0; color:#333;">Reset Your Password</h2>
<p style="color:#555; line-height:1.5;">Click the button below to reset your password:</p>
<p style="text-align:center; margin:24px 0;">
<a href="{reset_url}" target="_blank" style="display:inline-block; padding:12px 24px; background-color:#4f46e5; color:#ffffff; text-decoration:none; border-radius:6px; font-weight:bold;">Reset Password</a>
</p>
<p style="color:#888; font-size:13px; line-height:1.5;">If the button doesn't work, copy and paste this link into your browser:</p>
<p style="font-size:13px; word-break:break-all;"><a href="{reset_url}" target="_blank" style="color:#4f46e5; text-decoration:underline;">{reset_url}</a></p>
<p style="color:#888; font-size:13px; margin-top:24px;">This link expires in 1 hour. If you didn't request this, you can ignore this email.</p>
</td></tr>
</table>
</body>
</html>"""

    msg.attach(MIMEText(text, "plain", _charset="utf-8"))
    msg.attach(MIMEText(html, "html", _charset="utf-8"))

    def _send():
        with smtplib.SMTP(settings.smtp_host, settings.smtp_port) as server:
            server.starttls()
            server.login(settings.smtp_email, settings.smtp_app_password)
            server.sendmail(settings.smtp_email, to_email, msg.as_string())

    await asyncio.to_thread(_send)
```

Key points:
- SMTP runs in `asyncio.to_thread()` to avoid blocking the event loop
- In dev (localhost), logs the reset link to console if SMTP isn't configured
- In production, raises an error if SMTP isn't configured (caught by the route's try/except)
- Always pass `_charset="utf-8"` to `MIMEText()` — without it, some clients misrender special characters

**Email HTML compatibility (critical — these were discovered through production bugs):**
- **Table-based layout only** — Gmail's mobile app strips `<div>` and flex/grid styling but respects `<table>` with `role="presentation"`
- **Inline styles only** — Gmail strips `<style>` blocks and CSS classes entirely; every element needs a `style` attribute
- **`<meta name="x-apple-disable-message-reformatting">`** — prevents Apple Mail from aggressively reformatting the email layout
- **`target="_blank"` on ALL links** — Apple iOS Mail won't recognize links as tappable without this attribute
- **Fallback URL must be a real `<a>` tag** — plain text URLs are not clickable in Apple Mail; always wrap in `<a href="..." target="_blank">` with explicit styling
- **URL scheme must be present** — Apple Mail doesn't recognize URLs without `https://` as clickable; use a Pydantic field_validator on `frontend_base_url` to auto-prepend the scheme (see config below)
- Use `word-break:break-all` on fallback URL paragraphs so long tokens don't overflow on mobile

Ensure `api/services/__init__.py` exists (empty file).

---

## Verify

Start the dev server:

```bash
uv run uvicorn api.main:app --reload --port 8020
```

Test the endpoints:

```bash
# Health check
curl http://localhost:8020/api/health

# Register
curl -X POST http://localhost:8020/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Test User","email":"test@example.com","password":"testpass123"}'

# Login
curl -X POST http://localhost:8020/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"testpass123"}'

# Get current user (use the token from login response)
curl http://localhost:8020/api/auth/me \
  -H "Authorization: Bearer <token>"

# Forgot password (check server console for the reset link in dev)
curl -X POST http://localhost:8020/api/auth/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com"}'

# Reset password (use the token from the reset link)
curl -X POST http://localhost:8020/api/auth/reset-password \
  -H "Content-Type: application/json" \
  -d '{"token":"<token-from-reset-link>","new_password":"newpass123"}'
```

---

## Checklist

- [ ] `api/main.py` created with lifespan, CORS, health check
- [ ] `api/schemas/orm/user.py` — User document model (email, display_name, hashed_password)
- [ ] `api/schemas/orm/password_reset.py` — PasswordResetToken document model
- [ ] `api/utils/auth.py` — password hashing, JWT, get_current_user
- [ ] `api/schemas/dto/auth.py` — request/response models (including password reset DTOs)
- [ ] `api/routes/auth.py` — register, login, forgot-password, reset-password, /me
- [ ] `api/services/email.py` — send_password_reset_email (Gmail SMTP)
- [ ] All `__init__.py` files in place
- [ ] `curl /api/health` returns `{"status": "ok"}`
- [ ] Can register a user, login, and call `/api/auth/me` with the token
- [ ] Forgot password sends reset link (logged to console in dev, emailed in production)

---

## Optional: Google Sign-In + mandatory email verification

If the project requires Google Sign-In and/or email verification on registration, read `${CLAUDE_SKILL_DIR}/references/google-auth.md` before running this phase. It patches the `User` model, `Settings`, DTOs, and `auth.py` routes with the exact Toolshed implementation — do the baseline phase first, then apply the reference's diffs on top.
- [ ] Reset password with valid token updates the password
