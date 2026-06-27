# Phase 06: Frontend Features

Build feature components, wire them into the URL-based routing system, and apply mobile-first responsive design.

**Prerequisite:** Phase 05 complete (React app shell with auth and routing working).

---

## Component Organization

Group components by feature, not by type:

```
components/
├── auth/           # Login.tsx, Register.tsx, ForgotPassword.tsx, ResetPassword.tsx, LandingPage.tsx
├── layout/         # AppLayout.tsx, Header.tsx, Sidebar.tsx
├── tasks/          # TaskList.tsx, TaskItem.tsx, TaskDetail.tsx
└── categories/     # CategoryFilter.tsx, CategoryForm.tsx
```

Each component should:
- Be a single default export function component
- Accept props via a typed interface
- Use Tailwind utility classes directly — no separate CSS files
- Keep state local unless it needs to be shared (then lift to context)

---

## Auth Components

Build the auth pages referenced by `App.tsx`:

### Login.tsx

```typescript
// frontend/src/components/auth/Login.tsx
import { useState } from 'react'
import { useAuth } from '../../context/AuthContext'
import { useRouter } from '../../hooks/useRouter'

export default function Login() {
  const { login } = useAuth()
  const { navigate } = useRouter()
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const [error, setError] = useState('')

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setError('')
    try {
      await login(email, password)
      navigate('/items', true)
    } catch {
      setError('Invalid credentials')
    }
  }

  return (
    <div className="min-h-screen flex items-center justify-center bg-[var(--bg-main)] px-4">
      <form onSubmit={handleSubmit} className="w-full max-w-sm space-y-4">
        <h1 className="text-2xl font-bold text-[var(--text-primary)] text-center">Login</h1>
        {error && <p className="text-[var(--danger)] text-sm text-center">{error}</p>}
        <input
          type="email"
          placeholder="Email"
          value={email}
          onChange={e => setEmail(e.target.value)}
          className="w-full px-3 py-2 bg-[var(--bg-surface)] border border-[var(--border-color)] rounded text-[var(--text-primary)] min-h-[44px] sm:min-h-0"
        />
        <input
          type="password"
          placeholder="Password"
          value={password}
          onChange={e => setPassword(e.target.value)}
          className="w-full px-3 py-2 bg-[var(--bg-surface)] border border-[var(--border-color)] rounded text-[var(--text-primary)] min-h-[44px] sm:min-h-0"
        />
        <button
          type="submit"
          className="w-full py-2 bg-[var(--accent)] hover:bg-[var(--accent-hover)] text-white rounded font-medium min-h-[44px] sm:min-h-0"
        >
          Sign in
        </button>
        <p className="text-center text-sm text-[var(--text-secondary)]">
          <a href="/forgot-password" onClick={e => { e.preventDefault(); navigate('/forgot-password') }} className="text-[var(--accent)]">
            Forgot password?
          </a>
        </p>
        <p className="text-center text-sm text-[var(--text-secondary)]">
          Don't have an account?{' '}
          <a href="/register" onClick={e => { e.preventDefault(); navigate('/register') }} className="text-[var(--accent)]">
            Register
          </a>
        </p>
      </form>
    </div>
  )
}
```

Follow the same pattern for `Register.tsx` (with name, email, password fields) and `LandingPage.tsx`.

### ForgotPassword.tsx

```typescript
// frontend/src/components/auth/ForgotPassword.tsx
import { useState } from 'react'
import { useRouter } from '../../hooks/useRouter'
import { api } from '../../api/client'

export default function ForgotPassword() {
  const { navigate } = useRouter()
  const [email, setEmail] = useState('')
  const [loading, setLoading] = useState(false)
  const [submitted, setSubmitted] = useState(false)
  const [error, setError] = useState('')

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setLoading(true)
    setError('')
    try {
      await api.post('/api/auth/forgot-password', { email })
      setSubmitted(true)
    } catch {
      setError('Something went wrong. Please try again.')
    } finally {
      setLoading(false)
    }
  }

  return (
    <div className="min-h-screen flex items-center justify-center bg-[var(--bg-main)] px-4">
      <div className="w-full max-w-sm space-y-4">
        <h1 className="text-2xl font-bold text-[var(--text-primary)] text-center">Reset Password</h1>
        {submitted ? (
          <div className="text-center space-y-3">
            <p className="text-sm text-[var(--text-secondary)]">
              If that email is registered, you'll receive a password reset link shortly.
            </p>
            <a href="/login" onClick={e => { e.preventDefault(); navigate('/login') }} className="text-sm text-[var(--accent)]">
              Back to sign in
            </a>
          </div>
        ) : (
          <form onSubmit={handleSubmit} className="space-y-4">
            <p className="text-sm text-[var(--text-secondary)] text-center">
              Enter your email and we'll send you a reset link.
            </p>
            {error && <p className="text-[var(--danger)] text-sm text-center">{error}</p>}
            <input
              type="email"
              placeholder="Email"
              value={email}
              onChange={e => setEmail(e.target.value)}
              className="w-full px-3 py-2 bg-[var(--bg-surface)] border border-[var(--border-color)] rounded text-[var(--text-primary)] min-h-[44px] sm:min-h-0"
              required
              autoFocus
            />
            <button
              type="submit"
              disabled={loading}
              className="w-full py-2 bg-[var(--accent)] hover:bg-[var(--accent-hover)] text-white rounded font-medium min-h-[44px] sm:min-h-0 disabled:opacity-50"
            >
              {loading ? 'Sending...' : 'Send reset link'}
            </button>
            <p className="text-center text-sm">
              <a href="/login" onClick={e => { e.preventDefault(); navigate('/login') }} className="text-[var(--accent)]">
                Back to sign in
              </a>
            </p>
          </form>
        )}
      </div>
    </div>
  )
}
```

### ResetPassword.tsx

```typescript
// frontend/src/components/auth/ResetPassword.tsx
import { useState } from 'react'
import { useRouter } from '../../hooks/useRouter'
import { api } from '../../api/client'

interface ResetPasswordProps {
  token: string
}

export default function ResetPassword({ token }: ResetPasswordProps) {
  const { navigate } = useRouter()
  const [password, setPassword] = useState('')
  const [confirmPassword, setConfirmPassword] = useState('')
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState('')
  const [success, setSuccess] = useState(false)

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setError('')
    if (password !== confirmPassword) {
      setError('Passwords do not match.')
      return
    }
    setLoading(true)
    try {
      await api.post('/api/auth/reset-password', { token, new_password: password })
      setSuccess(true)
    } catch {
      setError('Invalid or expired reset link.')
    } finally {
      setLoading(false)
    }
  }

  return (
    <div className="min-h-screen flex items-center justify-center bg-[var(--bg-main)] px-4">
      <div className="w-full max-w-sm space-y-4">
        <h1 className="text-2xl font-bold text-[var(--text-primary)] text-center">Set New Password</h1>
        {success ? (
          <div className="text-center space-y-3">
            <p className="text-sm text-[var(--text-secondary)]">Your password has been reset successfully.</p>
            <a href="/login" onClick={e => { e.preventDefault(); navigate('/login') }} className="text-sm text-[var(--accent)]">
              Sign in
            </a>
          </div>
        ) : (
          <form onSubmit={handleSubmit} className="space-y-4">
            {error && <p className="text-[var(--danger)] text-sm text-center">{error}</p>}
            <input
              type="password"
              placeholder="New password"
              value={password}
              onChange={e => setPassword(e.target.value)}
              className="w-full px-3 py-2 bg-[var(--bg-surface)] border border-[var(--border-color)] rounded text-[var(--text-primary)] min-h-[44px] sm:min-h-0"
              minLength={6}
              required
              autoFocus
            />
            <input
              type="password"
              placeholder="Confirm password"
              value={confirmPassword}
              onChange={e => setConfirmPassword(e.target.value)}
              className="w-full px-3 py-2 bg-[var(--bg-surface)] border border-[var(--border-color)] rounded text-[var(--text-primary)] min-h-[44px] sm:min-h-0"
              minLength={6}
              required
            />
            <button
              type="submit"
              disabled={loading}
              className="w-full py-2 bg-[var(--accent)] hover:bg-[var(--accent-hover)] text-white rounded font-medium min-h-[44px] sm:min-h-0 disabled:opacity-50"
            >
              {loading ? 'Resetting...' : 'Reset password'}
            </button>
          </form>
        )}
      </div>
    </div>
  )
}
```

---

## Layout Components

### AppLayout.tsx

The authenticated app shell with header and nav. The top-right of the header has two buttons: the user's display name (navigates to `/profile` for user settings) and a logout button. There is no separate settings page — all user configuration lives under the profile page in sub-sections.

```typescript
// frontend/src/components/layout/AppLayout.tsx
import { useAuth } from '../../context/AuthContext'
import { useRouter } from '../../hooks/useRouter'

interface AppLayoutProps {
  currentView: string
  children: React.ReactNode
}

// Feature nav items only — no "settings" or "profile" here.
// Profile is accessed via the user's name button in the header.
const NAV_ITEMS = [
  { key: 'items', label: 'Items' },
]

export default function AppLayout({ currentView, children }: AppLayoutProps) {
  const { user, logout } = useAuth()
  const { navigate } = useRouter()

  return (
    <div className="min-h-screen bg-[var(--bg-main)]">
      <header className="bg-[var(--header-bg)] border-b border-[var(--border-color)] px-4 py-3">
        <div className="flex items-center justify-between">
          <h1 className="text-lg font-semibold text-[var(--text-primary)]">My App</h1>
          <div className="flex items-center gap-4">
            <a
              href="/profile"
              onClick={e => { e.preventDefault(); navigate('/profile') }}
              className="text-sm text-[var(--text-secondary)] hover:text-[var(--text-primary)]"
            >
              {user?.name}
            </a>
            <button
              onClick={logout}
              className="text-sm text-[var(--text-muted)] hover:text-[var(--text-primary)]"
            >
              Logout
            </button>
          </div>
        </div>
        <nav className="flex gap-1 mt-2 overflow-x-auto">
          {NAV_ITEMS.map(item => (
            <a
              key={item.key}
              href={`/${item.key}`}
              onClick={e => { e.preventDefault(); navigate(`/${item.key}`) }}
              className={`px-3 py-1.5 rounded text-sm min-h-[44px] sm:min-h-0 flex items-center ${
                currentView === item.key
                  ? 'bg-[var(--selected-bg)] text-[var(--accent)]'
                  : 'text-[var(--text-secondary)] hover:text-[var(--text-primary)]'
              }`}
            >
              {item.label}
            </a>
          ))}
        </nav>
      </header>
      <main className="p-4">{children}</main>
    </div>
  )
}
```

Key patterns:
- Header top-right: user's display name (links to profile) + logout button — both styled the same
- Nav links use `<a>` tags with `href` and `preventDefault` — enables right-click "Open in new tab", middle-click, and URL on hover
- No "Settings" nav item — profile is accessed via the user's name button

### UserProfile.tsx

The profile page is the user settings page. It contains the user's info and all app settings in sub-sections. Logout is in the header, not here.

```typescript
// frontend/src/components/UserProfile.tsx
import { useAuth } from '../context/AuthContext'

export default function UserProfile() {
  const { user } = useAuth()

  return (
    <div className="max-w-lg mx-auto space-y-6">
      {/* User Info */}
      <section className="bg-[var(--bg-surface)] border border-[var(--border-color)] rounded-lg p-4 space-y-3">
        <h2 className="text-lg font-semibold text-[var(--text-primary)]">Profile</h2>
        <div className="space-y-2 text-sm">
          <div>
            <span className="text-[var(--text-muted)]">Name</span>
            <p className="text-[var(--text-primary)]">{user?.name}</p>
          </div>
          <div>
            <span className="text-[var(--text-muted)]">Email</span>
            <p className="text-[var(--text-primary)]">{user?.email}</p>
          </div>
        </div>
      </section>

      {/*
        This is the single place for ALL app settings and configuration.
        Do not create a separate settings page — add sub-sections here.
        Examples: notifications, display preferences, connected accounts,
        API tokens, privacy settings, etc.
      */}
      {/* Example:
      <section className="bg-[var(--bg-surface)] border border-[var(--border-color)] rounded-lg p-4 space-y-3">
        <h2 className="text-lg font-semibold text-[var(--text-primary)]">Notifications</h2>
        ...
      </section>
      */}
    </div>
  )
}
```

Key patterns:
- **This is the only settings page** — there is no separate `/settings` route. All user preferences, app configuration, and account management live here as sub-sections
- Each section is a card (`bg-surface` + `border` + `rounded-lg`)
- Add new sub-sections (notifications, preferences, API tokens, etc.) as additional `<section>` cards
- Keep `max-w-lg mx-auto` so the profile doesn't stretch on wide screens

---

## URL Design

Plan URL paths for every navigable state:

| Path | What it shows |
|------|---------------|
| `/items` | List view |
| `/items/new` | Create form |
| `/items/:id` | Detail view |
| `/items/:id/edit` | Edit form |
| `/profile` | User profile & app settings |

---

## Deriving State from the URL

Replace `useState` for selection/mode with URL-derived values:

```typescript
// Before (broken back button, no deep links):
const [selectedId, setSelectedId] = useState<string | null>(null)

// After (URL is the source of truth):
const { path, navigate } = useRouter()
const selectedId = path.match(/^\/items\/(.+)/)?.[1] ?? null
// Click handler: navigate(`/items/${item.id}`)
// Close handler: navigate('/items')
```

State that is purely UI (search filters, sort mode, toggles) stays in `useState` — only navigable page transitions go in the URL.

### Using matchPath

For more complex routes, use the `matchPath` helper from `useRouter.ts`:

```typescript
import { matchPath } from '../hooks/useRouter'

const detailMatch = matchPath('/items/:id', path)
if (detailMatch) {
  const itemId = detailMatch.id
  // render detail view
}
```

---

## Mobile-First Design

### Principles

- **Start small**: Write the mobile layout as the default styles (no prefix). Add `sm:`, `md:`, `lg:` only when the layout needs to change at larger sizes.
- **Touch-friendly targets**: Interactive elements should be at least 44px tall on mobile. Use `min-h-[44px] sm:min-h-0`.
- **Stack on mobile, row on desktop**: Use `flex flex-col sm:flex-row`.
- **Full-width on mobile**: Buttons and inputs should be `w-full sm:w-auto`.
- **Responsive grids**: Use `grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-4` for card layouts.

### Common Patterns

```tsx
{/* Stack on mobile, row on desktop */}
<div className="flex flex-col sm:flex-row sm:items-center justify-between gap-3">
  <h2 className="text-lg font-semibold">Title</h2>
  <button className="w-full sm:w-auto min-h-[44px] sm:min-h-0 ...">Action</button>
</div>

{/* Responsive card grid */}
<div className="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-4 gap-3">
  {items.map(item => <Card key={item.id} ... />)}
</div>

{/* Horizontal scroll tabs on mobile */}
<nav className="flex gap-1 overflow-x-auto scrollbar-hide">
  {tabs.map(tab => <a ... />)}
</nav>
```

### Breakpoints (Tailwind defaults)

| Prefix | Min width | Typical device |
|--------|-----------|----------------|
| (none) | 0px | Phones |
| `sm:` | 640px | Large phones / small tablets |
| `md:` | 768px | Tablets |
| `lg:` | 1024px | Laptops / desktops |

Most apps only need `sm:` and `lg:`. Avoid overusing `md:` and `xl:`.

---

## Wiring Views into App.tsx

Update `App.tsx` to render actual components instead of placeholders:

```typescript
function ViewContent({ view }: { view: View }) {
  switch (view) {
    case 'items':
      return <ItemList />
    case 'profile':
      return <UserProfile />
    default:
      return null
  }
}

// In the authenticated section of App:
return (
  <AppLayout currentView={currentView}>
    <ViewContent view={currentView} />
  </AppLayout>
)
```

---

## Checklist

- [ ] Auth components built (Login, Register, ForgotPassword, ResetPassword, LandingPage)
- [ ] Layout components built (AppLayout with header, nav, and user name link to profile)
- [ ] UserProfile component built — single place for user info and all app settings (no separate settings page)
- [ ] Feature components organized by feature directory
- [ ] URL paths designed for all navigable views
- [ ] Views derive state from URL, not useState
- [ ] Nav links use `<a>` with `href` and `preventDefault`
- [ ] Mobile-first responsive design applied (touch targets, stacking, grids)
- [ ] All views reachable via URL navigation
- [ ] Browser back/forward works correctly
- [ ] Page refresh preserves the current view

---

## Optional: Google Sign-In button + email verification UX

If the backend ships Google Sign-In and/or mandatory email verification (see Phase 02's optional section), read `${CLAUDE_SKILL_DIR}/references/google-auth.md` for the frontend counterparts: `GoogleOAuthProvider` in `main.tsx`, the `GoogleLogin` button in Login/Register, the `/verify-email/:token` route + component, and the "Check your inbox" flow after registration.
