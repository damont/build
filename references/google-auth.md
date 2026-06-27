# Reference: Google OAuth + Mandatory Email Verification

Implementation guide for adding Google Sign-In and mandatory email verification to an app that already has the baseline JWT email/password auth from Phase 02. This is the exact pattern used in Toolshed.

**Pull this in when:** you want users to be able to sign in with Google AND/OR you want new email/password registrations to require email verification before they can log in.

The two features share enough plumbing (a shared Client ID, the `email_verified` column, the invite-token passthrough) that they're documented together. You can ship just verification without Google, but if you ship Google you should also ship verification (Google users are auto-verified via the ID token's `email_verified` claim).

---

## High-level design

- **Single Google Client ID** is used on both sides. Backend uses it to verify the audience (`aud`) claim on ID tokens; frontend uses it to render the Google button. Get one from https://console.cloud.google.com/apis/credentials.
- **Frontend sends a Google ID token** (JWT signed by Google) to `POST /api/auth/google`. Backend verifies signature + audience with `google-auth`, then links/creates a user by `sub` (Google's stable user ID), falling back to email match.
- **Email verification** uses a single-use token stored in Mongo (`EmailVerificationToken` collection). Registration creates the user with `email_verified=False` and emails the link. Login blocks unverified users with a 403 whose detail is the string literal `email_not_verified` — the frontend matches on that to show a "Resend verification email" button.
- **Grandfathering**: the `email_verified` field on `User` defaults to `True`, so existing users aren't locked out when you ship this. Only the `/register` endpoint explicitly sets it to `False`.
- **Google users skip verification**: the ID token's `email_verified` claim is trusted, and accounts created/linked via Google are marked verified.
- **Linking**: if a Google sub matches an existing email (because the user previously registered with email/password), the accounts are linked on first Google login (same `User` document gets a `google_sub`).

---

## Dependencies

### Backend (`pyproject.toml`)

```toml
"google-auth[requests]>=2.29.0",
```

**Beanie pin**: if your project was built before Beanie 2.0, keep `"beanie>=1.27,<2.0"`. Beanie 2.0 changed `init_beanie` in a way that breaks the Phase 02 lifespan setup.

### Frontend (`frontend/package.json`)

```json
"@react-oauth/google": "^0.12.1"
```

---

## Backend changes

### 1. Config: add `google_client_id` (`api/config.py`)

Add inside the `Settings` class:

```python
# Google OAuth
google_client_id: Optional[str] = None

# Email verification
email_verification_expire_minutes: int = 60 * 24  # 24 hours
```

The `invite_base_url` scheme validator is worth keeping if you don't already have it — makes Terraform-supplied hostnames Just Work:

```python
from pydantic import field_validator

@field_validator("invite_base_url")
@classmethod
def _ensure_scheme(cls, v: str) -> str:
    if v and not v.startswith(("http://", "https://")):
        return f"https://{v}"
    return v
```

### 2. User model updates (`api/schemas/orm/user.py`)

```python
from datetime import datetime
from typing import Optional

import pymongo
from beanie import Document, Indexed
from pydantic import EmailStr


class User(Document):
    email: Indexed(EmailStr, unique=True)
    username: Indexed(str, unique=True)
    # Optional for Google-only accounts (no password set).
    hashed_password: Optional[str] = None
    display_name: Optional[str] = None
    phone: Optional[str] = None
    # Plain string with a sparse unique index (declared in Settings.indexes)
    # so multiple users without a google_sub don't collide on None.
    google_sub: Optional[str] = None
    # Default True so existing users are grandfathered as verified;
    # the register endpoint explicitly sets False for new signups.
    email_verified: bool = True
    created_at: datetime = datetime.utcnow()
    is_active: bool = True

    class Settings:
        name = "users"
        indexes = [
            pymongo.IndexModel(
                [("google_sub", pymongo.ASCENDING)],
                unique=True,
                sparse=True,
                name="google_sub_unique_sparse",
            ),
        ]
```

Two details that matter:
- `hashed_password` becomes `Optional` because Google-only accounts have no password.
- The sparse unique index on `google_sub` is declared via `pymongo.IndexModel` in `Settings.indexes` — **not** via `Indexed(...)` — because Beanie's `Indexed` helper doesn't support `sparse=True`. Without sparse, every user without a Google account would collide on `None`.

### 3. Email verification token model (`api/schemas/orm/email_verification.py`)

```python
from datetime import datetime
from typing import Optional

from beanie import Document, Indexed
from pydantic import Field


class EmailVerificationToken(Document):
    token: Indexed(str, unique=True)
    user_id: Indexed(str)
    expires_at: datetime
    used_at: Optional[datetime] = None
    created_at: datetime = Field(default_factory=datetime.utcnow)

    class Settings:
        name = "email_verification_tokens"
```

**Register this in `init_beanie`** in `api/main.py`:

```python
from api.schemas.orm.email_verification import EmailVerificationToken
...
document_models=[User, PasswordResetToken, EmailVerificationToken, ...]
```

### 4. Auth DTOs (`api/schemas/dto/auth.py`)

Add these to the existing DTO file:

```python
class RegisterResponse(BaseModel):
    message: str
    email: EmailStr


class GoogleLoginRequest(BaseModel):
    id_token: str
    invite_token: Optional[str] = None


class VerifyEmailRequest(BaseModel):
    token: str


class ResendVerificationRequest(BaseModel):
    email: EmailStr
```

Also update the existing `UserRegister` to accept an optional invite token if your app uses invites:

```python
class UserRegister(BaseModel):
    name: str
    email: EmailStr
    password: str
    phone: Optional[str] = None
    invite_token: Optional[str] = None
```

### 5. Email service: add `send_verification_email` (`api/services/email.py`)

Mirror the existing `send_password_reset_email` pattern. Key bits:

```python
async def send_verification_email(to_email: str, verify_url: str) -> None:
    settings = get_settings()

    hours = max(1, settings.email_verification_expire_minutes // 60)
    expires_wording = f"{hours} hours" if hours != 1 else "1 hour"

    if not settings.smtp_email or not settings.smtp_app_password:
        if "localhost" in settings.invite_base_url:
            logger.warning("SMTP not configured — logging verification link (dev only)")
            logger.info("Email verification link for %s: %s", to_email, verify_url)
            return
        raise RuntimeError("SMTP not configured")

    msg = MIMEMultipart("alternative")
    msg["Subject"] = "<App Name> - Verify Your Email Address"
    msg["From"] = settings.smtp_email
    msg["To"] = to_email

    text = (
        f"Welcome! Activate your account by visiting:\n\n{verify_url}\n\n"
        f"This link expires in {expires_wording}."
    )
    # HTML body: reuse the same table layout as password reset, swap the copy.

    msg.attach(MIMEText(text, "plain", _charset="utf-8"))
    msg.attach(MIMEText(html, "html", _charset="utf-8"))

    def _send():
        with smtplib.SMTP(settings.smtp_host, settings.smtp_port) as server:
            server.starttls()
            server.login(settings.smtp_email, settings.smtp_app_password)
            server.sendmail(settings.smtp_email, to_email, msg.as_string())

    await asyncio.to_thread(_send)
```

The dev fallback (log the link when SMTP is unconfigured AND `invite_base_url` points at localhost) is critical — it means you don't need Gmail credentials to test the flow locally.

### 6. Auth routes (`api/routes/auth.py`)

Update imports:

```python
import secrets
from datetime import datetime, timedelta

from api.schemas.orm.email_verification import EmailVerificationToken
from api.schemas.dto.auth import (
    # ... existing ...
    RegisterResponse,
    GoogleLoginRequest,
    VerifyEmailRequest,
    ResendVerificationRequest,
)
from api.services.email import send_password_reset_email, send_verification_email
```

**Helper** (put near the top of the file, above the routes):

```python
async def _create_and_send_verification(user: User) -> None:
    settings = get_settings()
    token = secrets.token_urlsafe(32)
    verification = EmailVerificationToken(
        token=token,
        user_id=str(user.id),
        expires_at=datetime.utcnow()
        + timedelta(minutes=settings.email_verification_expire_minutes),
    )
    await verification.insert()

    verify_url = f"{settings.invite_base_url.rstrip('/')}/verify-email/{token}"
    try:
        await send_verification_email(user.email, verify_url)
    except Exception:
        logger.exception("Failed to send verification email to %s", user.email)
```

**Modify `/register`** to create users unverified and send the email. Return `RegisterResponse` instead of a token — the user can't log in yet:

```python
@router.post("/register", response_model=RegisterResponse, status_code=status.HTTP_201_CREATED)
async def register(data: UserRegister):
    existing_email = await User.find_one(User.email == data.email)
    if existing_email:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Email already registered",
        )

    user = User(
        email=data.email,
        username=data.email,
        hashed_password=hash_password(data.password),
        display_name=data.name,
        phone=data.phone,
        email_verified=False,
    )
    await user.insert()

    await _create_and_send_verification(user)

    return RegisterResponse(
        message="Account created. Please check your email to verify your address.",
        email=user.email,
    )
```

**Modify `/login`** to reject unverified users with the magic string the frontend matches on:

```python
if user.email_verified is False:
    raise HTTPException(
        status_code=status.HTTP_403_FORBIDDEN,
        detail="email_not_verified",
    )
```

**Add `/verify-email`**:

```python
@router.post("/verify-email", response_model=TokenResponse)
async def verify_email(data: VerifyEmailRequest):
    verification = await EmailVerificationToken.find_one(
        EmailVerificationToken.token == data.token
    )

    if (
        not verification
        or verification.used_at
        or verification.expires_at < datetime.utcnow()
    ):
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Invalid or expired verification token.",
        )

    user = await User.get(verification.user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Invalid or expired verification token.",
        )

    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User account is disabled",
        )

    verification.used_at = datetime.utcnow()
    await verification.save()

    user.email_verified = True
    await user.save()

    access_token = create_access_token(str(user.id))
    return TokenResponse(access_token=access_token)
```

**Add `/resend-verification`**. Always returns the same message whether or not the email exists (prevents enumeration):

```python
@router.post("/resend-verification", response_model=MessageResponse)
async def resend_verification(data: ResendVerificationRequest):
    message = "If that email is registered and unverified, a new verification link has been sent."

    user = await User.find_one(User.email == data.email)
    if not user or user.email_verified:
        return MessageResponse(message=message)

    # Invalidate prior unused tokens
    user_id = str(user.id)
    existing = await EmailVerificationToken.find(
        EmailVerificationToken.user_id == user_id,
        EmailVerificationToken.used_at == None,  # noqa: E711
    ).to_list()
    for t in existing:
        t.used_at = datetime.utcnow()
        await t.save()

    await _create_and_send_verification(user)
    return MessageResponse(message=message)
```

**Add `/google`**:

```python
@router.post("/google", response_model=TokenResponse)
async def google_login(data: GoogleLoginRequest):
    settings = get_settings()
    if not settings.google_client_id:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Google login not configured",
        )

    try:
        from google.oauth2 import id_token as google_id_token
        from google.auth.transport import requests as google_requests
    except ImportError:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Google auth library not installed",
        )

    try:
        idinfo = google_id_token.verify_oauth2_token(
            data.id_token, google_requests.Request(), settings.google_client_id
        )
    except ValueError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid Google token",
        )

    sub = idinfo.get("sub")
    email = idinfo.get("email")
    email_verified = idinfo.get("email_verified", False)
    name = idinfo.get("name") or (email.split("@")[0] if email else None)

    if not sub or not email or not email_verified:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Google token missing required claims",
        )

    user = await User.find_one(User.google_sub == sub)

    if not user:
        # Link Google to existing email/password user if email matches.
        user = await User.find_one(User.email == email)
        if user:
            user.google_sub = sub
            user.email_verified = True  # Google has verified the email
            await user.save()
        else:
            user = User(
                email=email,
                username=email,
                hashed_password=None,
                display_name=name,
                google_sub=sub,
                email_verified=True,
            )
            await user.insert()

    if not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User account is disabled",
        )

    access_token = create_access_token(str(user.id))
    return TokenResponse(access_token=access_token)
```

`verify_oauth2_token` raises `ValueError` on bad signature, wrong audience, or expired token — all collapse to a 401. Do **not** catch this broadly; let the specific `ValueError` surface.

---

## Frontend changes

### 1. Wrap the app with `GoogleOAuthProvider` (`frontend/src/main.tsx`)

The provider is conditional on the env var being set so the app still boots without a Client ID (useful during setup and for CI builds that don't have the secret):

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { GoogleOAuthProvider } from "@react-oauth/google";
import "./index.css";
import App from "./App";

const googleClientId = import.meta.env.VITE_GOOGLE_CLIENT_ID as string | undefined;

const rootNode = <App />;

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    {googleClientId ? (
      <GoogleOAuthProvider clientId={googleClientId}>{rootNode}</GoogleOAuthProvider>
    ) : (
      rootNode
    )}
  </StrictMode>
);
```

### 2. API client methods (`frontend/src/api/client.ts`)

Add to the `ApiClient` class:

```ts
async googleLogin(idToken: string, inviteToken?: string | null) {
  const response = await fetch(`${API_BASE}/auth/google`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ id_token: idToken, invite_token: inviteToken ?? null }),
  });
  if (!response.ok) {
    const error = await response.json().catch(() => ({ detail: "Google login failed" }));
    throw new ApiError(response.status, error.detail);
  }
  return response.json() as Promise<{ access_token: string; token_type: string }>;
}

async verifyEmail(token: string) {
  const response = await fetch(`${API_BASE}/auth/verify-email`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ token }),
  });
  if (!response.ok) {
    const error = await response.json().catch(() => ({ detail: "Verification failed" }));
    throw new ApiError(response.status, error.detail);
  }
  return response.json() as Promise<{ access_token: string; token_type: string }>;
}

async resendVerification(email: string) {
  const response = await fetch(`${API_BASE}/auth/resend-verification`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ email }),
  });
  if (!response.ok) {
    const error = await response.json().catch(() => ({ detail: "Request failed" }));
    throw new ApiError(response.status, error.detail);
  }
  return response.json() as Promise<{ message: string }>;
}
```

### 3. AuthContext additions (`frontend/src/context/AuthContext.tsx`)

- `register` now returns `{ verificationRequired: true, email }` instead of setting a token (user must verify first).
- Add `googleLogin(credential)` and `setTokenAndFetch(accessToken)` methods.

```tsx
interface RegisterResult {
  verificationRequired: true;
  email: string;
}

interface AuthContextValue extends AuthState {
  // ...
  register: (data: {...}) => Promise<RegisterResult>;
  googleLogin: (credential: string) => Promise<void>;
  setTokenAndFetch: (accessToken: string) => Promise<void>;
}

const setTokenAndFetch = async (accessToken: string) => {
  localStorage.setItem(TOKEN_KEY, accessToken);
  api.setToken(accessToken);
  const user = await api.get<User>("/auth/me");
  setState({ user, loading: false, error: null });
};

const register = async (data): Promise<RegisterResult> => {
  const resp = await api.post<{ message: string; email: string }>("/auth/register", data);
  return { verificationRequired: true, email: resp.email };
};

const googleLogin = async (credential: string) => {
  const inviteToken = localStorage.getItem("pending_invite_token");
  const { access_token } = await api.googleLogin(credential, inviteToken);
  await setTokenAndFetch(access_token);
};
```

### 4. Login component (`frontend/src/components/auth/Login.tsx`)

Key additions:

```tsx
import { GoogleLogin } from "@react-oauth/google";
const googleClientId = import.meta.env.VITE_GOOGLE_CLIENT_ID as string | undefined;

// In the component:
const { login, googleLogin } = useAuth();
const [unverified, setUnverified] = useState(false);

const handleSubmit = async (e) => {
  // ...
  try {
    await login(email, password);
  } catch (err) {
    if (err instanceof ApiError && err.status === 403 && err.message === "email_not_verified") {
      setUnverified(true);
      setLocalError("Your email isn't verified yet.");
    } else {
      setLocalError(err instanceof ApiError ? err.message : "Login failed");
    }
  }
};

const handleResend = async () => {
  await api.resendVerification(email);
  // show confirmation...
};

const handleGoogleSuccess = async (credential?: string) => {
  if (!credential) return;
  await googleLogin(credential);
};

// In JSX, above the email/password form:
{googleClientId && (
  <>
    <GoogleLogin
      onSuccess={(resp) => handleGoogleSuccess(resp.credential)}
      onError={() => setLocalError("Google login failed")}
    />
    {/* "or" divider */}
  </>
)}

// In the error block, when unverified is true, render a "Resend verification email" button.
```

The `err.message === "email_not_verified"` check relies on the backend returning that exact string as `detail`.

### 5. Register component (`frontend/src/components/auth/Register.tsx`)

- Same Google button pattern as Login.
- On successful `register()`, don't redirect — swap to a "Check your inbox" screen showing the pending email with a Resend button:

```tsx
const [pendingEmail, setPendingEmail] = useState<string | null>(null);

const handleSubmit = async (e) => {
  // ...
  const result = await register({ name, email, password, invite_token: inviteToken });
  setPendingEmail(result.email);
};

if (pendingEmail) {
  return (
    <div>
      <h2>Check your inbox</h2>
      <p>We sent a verification link to <strong>{pendingEmail}</strong>.</p>
      <button onClick={() => api.resendVerification(pendingEmail)}>
        Resend verification email
      </button>
    </div>
  );
}
```

### 6. VerifyEmail component (`frontend/src/components/auth/VerifyEmail.tsx`) — NEW

Landing page for the link in the email. Auto-verifies on mount, logs the user in on success, shows a resend form on error. Use a `useRef` guard so React StrictMode's double-invocation doesn't burn the token twice:

```tsx
import { useEffect, useRef, useState, type FormEvent } from "react";
import { api, ApiError } from "../../api/client";
import { useAuth } from "../../context/AuthContext";

interface VerifyEmailProps {
  token: string;
  onSwitchToLogin: () => void;
  onSuccess: () => void;
}

type Status = "verifying" | "success" | "error";

export default function VerifyEmail({ token, onSwitchToLogin, onSuccess }: VerifyEmailProps) {
  const { setTokenAndFetch } = useAuth();
  const [status, setStatus] = useState<Status>("verifying");
  const [error, setError] = useState("");
  const [resendEmail, setResendEmail] = useState("");
  const ran = useRef(false);

  useEffect(() => {
    if (ran.current) return;
    ran.current = true;
    (async () => {
      try {
        const { access_token } = await api.verifyEmail(token);
        await setTokenAndFetch(access_token);
        setStatus("success");
        onSuccess();
      } catch (err) {
        setStatus("error");
        setError(err instanceof ApiError ? err.message : "Invalid or expired verification link.");
      }
    })();
  }, [token, setTokenAndFetch, onSuccess]);

  // Render: "Verifying your email...", success redirect, or error + resend form.
}
```

### 7. Route wiring (`frontend/src/App.tsx`)

Add a route for `/verify-email/:token` that renders `<VerifyEmail>`. The component handles its own state; on success route the user into the authed shell (`/browse`, `/dashboard`, whatever).

---

## Config & secrets plumbing

### `.env.example` (root)

```
# Email / SMTP (Gmail)
SMTP_EMAIL=your-email@gmail.com
SMTP_APP_PASSWORD=your-app-password
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
PASSWORD_RESET_EXPIRE_MINUTES=60
EMAIL_VERIFICATION_EXPIRE_MINUTES=1440

# Google OAuth (same Client ID for backend verification and frontend button).
# Get one from https://console.cloud.google.com/apis/credentials
# Backend uses it to verify the ID token's audience claim; frontend uses it
# to render the Google Sign-In button. The frontend var is baked in at
# `docker compose build` time via the VITE_GOOGLE_CLIENT_ID build arg, so
# change this value and rebuild the frontend image to roll it out.
GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
VITE_GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
```

Both vars hold the **same value**. The duplication exists because Vite rewrites `import.meta.env.VITE_*` at build time, so the frontend needs its own `VITE_`-prefixed copy.

### `Dockerfile` (frontend build stage)

```dockerfile
ARG VITE_GOOGLE_CLIENT_ID
ENV VITE_GOOGLE_CLIENT_ID=$VITE_GOOGLE_CLIENT_ID
RUN npm run build
```

The `ARG` must appear **after** `FROM` (or re-declared per stage) and **before** `RUN npm run build`. Vite resolves `import.meta.env.VITE_GOOGLE_CLIENT_ID` at build time, so the value is baked into the static bundle — rolling it out requires rebuilding the frontend image.

### `docker-compose.yml`

```yaml
frontend:
  build:
    context: .
    dockerfile: Dockerfile.frontend
    args:
      VITE_GOOGLE_CLIENT_ID: ${VITE_GOOGLE_CLIENT_ID:-}
```

Backend service just reads `GOOGLE_CLIENT_ID` from its `env_file: .env` — no special wiring needed.

### GitHub Actions workflows

In the frontend image build step:

```yaml
docker build -f Dockerfile.frontend \
  --build-arg VITE_GOOGLE_CLIENT_ID=${{ secrets.GOOGLE_CLIENT_ID }} \
  -t $ACR_LOGIN_SERVER/app-frontend:${{ github.sha }} .
```

In the Terraform apply step:

```yaml
env:
  TF_VAR_google_client_id: ${{ secrets.GOOGLE_CLIENT_ID }}
```

Add `GOOGLE_CLIENT_ID` as a repo secret — one value, used by both the frontend build arg and the Terraform var.

### Terraform (`infra/variables.tf`)

```hcl
variable "google_client_id" {
  type      = string
  sensitive = true
  default   = ""
}
```

### Terraform (`infra/main.tf`) — API container env

```hcl
secure_environment_variables = {
  # ... existing ...
  GOOGLE_CLIENT_ID = var.google_client_id
}
```

---

## Google Cloud Console setup

These apps reuse a **single shared Google Cloud project** — `toolshed-493402` —
for OAuth across every app, rather than creating a new project per app. Open it
here:

**https://console.cloud.google.com/welcome?project=toolshed-493402**

You have two choices within that project, both fine:

- **Reuse the existing Web-application OAuth client** (simplest — one Client ID
  for all apps). Just add the new app's origins to it (step 3 below). Every app
  then shares the same `GOOGLE_CLIENT_ID`.
- **Create a new OAuth client** inside the project (one Client ID per app) if you
  want them isolated.

Steps:

1. In project `toolshed-493402`, go to **APIs & Services → Credentials**
   (https://console.cloud.google.com/apis/credentials?project=toolshed-493402).
   If this is the project's first OAuth client, you'll be prompted to configure
   the **OAuth consent screen** once (External, app name, support email) before
   you can create a credential.
2. Either open the existing **OAuth 2.0 Client ID** (type *Web application*) or
   create one (**+ Create Credentials → OAuth client ID → Web application**).
3. **Authorized JavaScript origins** — add every host this app's frontend is
   served from (scheme + host + port, no path, no trailing slash):
   - `http://localhost:<dev-port>` (the app's frontend port — e.g. `http://localhost:8102`)
   - `https://dev.yourapp.com`
   - `https://yourapp.com`
   Adding a new app to the shared client just means appending its origins here.
4. **Authorized redirect URIs** — not needed for `@react-oauth/google`'s
   `GoogleLogin` button; it uses the implicit/ID-token flow. Leave blank unless
   you add a server-side redirect flow later.
5. Copy the Client ID (ends in `.apps.googleusercontent.com`) into both
   `GOOGLE_CLIENT_ID` and `VITE_GOOGLE_CLIENT_ID` (same value) in `.env`, then
   rebuild the frontend image so Vite bakes the `VITE_` copy into the bundle.

**No client secret is used.** The backend only verifies the ID token's signature
and audience — verification is public-key-based.

**Gotcha — origins are exact-match.** `http://localhost:8102` and
`http://localhost:8025` are different origins; the JS origin must be the host the
**frontend** is served from, not the API. A missing/incorrect origin makes the
Google button render but fail on click with an `idpiframe`/origin error. Changes
to authorized origins can take a few minutes to propagate.

---

## Checklist

- [ ] `google-auth[requests]>=2.29.0` in `pyproject.toml`, `@react-oauth/google` in `frontend/package.json`
- [ ] `google_client_id` and `email_verification_expire_minutes` in `Settings`
- [ ] `User` has `google_sub: Optional[str]`, `email_verified: bool = True`, `hashed_password: Optional[str]`, and the sparse unique index on `google_sub`
- [ ] `EmailVerificationToken` document registered in `init_beanie`
- [ ] `send_verification_email` in `api/services/email.py`
- [ ] `_create_and_send_verification` helper in `api/routes/auth.py`
- [ ] `/register` creates users with `email_verified=False` and returns `RegisterResponse` (no token)
- [ ] `/login` rejects unverified users with 403 + detail `email_not_verified`
- [ ] `/verify-email`, `/resend-verification`, `/google` endpoints
- [ ] Frontend `main.tsx` wraps in `GoogleOAuthProvider` when env var is set
- [ ] `googleLogin`, `verifyEmail`, `resendVerification` methods on API client
- [ ] `AuthContext` exposes `googleLogin` + `setTokenAndFetch`; `register` returns `{ verificationRequired, email }`
- [ ] `Login` shows Google button + unverified resend path
- [ ] `Register` shows Google button + "check your inbox" screen
- [ ] `VerifyEmail` component + `/verify-email/:token` route
- [ ] `GOOGLE_CLIENT_ID` / `VITE_GOOGLE_CLIENT_ID` in `.env.example`
- [ ] Dockerfile has `ARG VITE_GOOGLE_CLIENT_ID` before `npm run build`
- [ ] `docker-compose.yml` passes `VITE_GOOGLE_CLIENT_ID` as a build arg
- [ ] GitHub Actions build step passes `--build-arg VITE_GOOGLE_CLIENT_ID=${{ secrets.GOOGLE_CLIENT_ID }}`
- [ ] GitHub Actions TF step sets `TF_VAR_google_client_id`
- [ ] Terraform `variable "google_client_id"` + API container env `GOOGLE_CLIENT_ID = var.google_client_id`
- [ ] Google Cloud Console: OAuth client created, all frontend origins added

---

## Common gotchas

- **`email_not_verified` string is a protocol** between backend and frontend. Don't rename it without updating both sides.
- **Vite env vars bake in at build time.** Changing `VITE_GOOGLE_CLIENT_ID` in prod means rebuilding the frontend image — not just restarting the container.
- **The sparse index on `google_sub` must be created via `pymongo.IndexModel`**, not via Beanie's `Indexed(...)` helper. Beanie's helper doesn't expose `sparse=True`, so without this you'll get duplicate-key errors on the second user without a Google account.
- **Beanie 2.0 breaks `init_beanie`** in ways that aren't obvious until runtime. Pin `beanie>=1.27,<2.0` until you've done the 2.0 migration intentionally.
- **Dev without SMTP works** because `send_verification_email` logs the link when SMTP is unconfigured and `invite_base_url` contains `localhost`. In prod, unconfigured SMTP raises `RuntimeError` — which is caught by `_create_and_send_verification` so registration still succeeds, but the user won't get an email. Monitor logs for "Failed to send verification email".
- **`invite_base_url` must be a full URL with scheme** for verification links to work. The `_ensure_scheme` validator prepends `https://` if missing — matters when Terraform passes bare hostnames.
- **Google button only renders when `VITE_GOOGLE_CLIENT_ID` is set.** If the button doesn't appear in a deployed build, the secret wasn't passed as a build arg or the `ARG` was below `RUN npm run build` in the Dockerfile.
