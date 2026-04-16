# Admin Authentication (Login Only)

## Why

The portfolio is a public-facing site with a single admin user (the portfolio owner). Authentication gates all content management actions -- creating/editing/deleting projects, managing manuals and steps, adding technologies. Without auth, these controls would be exposed to anyone.

## What

A stateless JWT-based authentication system with login only (no registration flow in the UI). The server issues a token on login, the client stores it in localStorage and sends it with every API request via `Authorization: Bearer <token>` header.

**User actions:**
- **Login** with email and password -> receives JWT token -> redirected to home page
- **Logout** via navbar button -> token removed from localStorage -> UI reverts to public mode

**Token lifetime:** 6 hours, configured via `jwt.sign()` with `expiresIn: "6h"`. After expiration, the user must log in again.

**Protected routes without valid token:** Return HTTP 401 (via `express-jwt` middleware). Client-side, the `IsPrivate` component redirects unauthenticated users to `/login` before any API call is made.

### API Endpoints

| Method | Endpoint | Auth | Request Body | Response | Status |
|--------|----------|------|-------------|----------|--------|
| POST | `/auth/login` | No | `{ email, password }` | `{ authToken }` | 200 |
| GET | `/auth/verify` | Yes | -- | `{ id, email, name }` | 200 |

**Validation:**
- `email`: required, must not be empty string
- `password`: required, must not be empty string

**Rate limiting:** POST `/auth/login` is rate-limited to 5 attempts per 15 minutes per IP using `express-rate-limit`. When the limit is exceeded, returns HTTP 429 with message `"Too many login attempts. Please try again later."` Standard `RateLimit-*` headers are returned; legacy `X-RateLimit-*` headers are disabled.

**Server-side error messages:**
- Empty email or password: `"Provide email and password."` (400)
- Email not found: `"User not found."` (401)
- Wrong password: `"Unable to authenticate the user"` (401)
- Rate limit exceeded: `"Too many login attempts. Please try again later."` (429)
- Server error: `"Error during login"` (500)

**Client-side error handling:** Displays the server's error message directly from `error.response.data.message`.

### Database Schema

**Table: `User`**

| Column | Type | Constraints |
|--------|------|------------|
| id | String | PK, cuid |
| email | String | UNIQUE, NOT NULL |
| password | String | NOT NULL (bcrypt hash) |
| name | String | NOT NULL |
| createdAt | DateTime | auto-generated |

No relations to other models (standalone).

### Password Change (CLI Utility)

There is no in-app password change. Instead, the admin changes passwords locally via a CLI utility:

```bash
cd server
npx ts-node prisma/change-password.ts
```

The script prompts interactively for:
1. Admin email address (must exist in the database)
2. New password (minimum 18 characters)
3. Password confirmation (must match)

It then bcrypt-hashes the new password (10 salt rounds) and updates the user record directly in the database.

**File:** `server/prisma/change-password.ts`

## Constraints

### Must

- **Server:** Express 4.x, `express-jwt` for token validation middleware, stateless (no sessions)
- **JWT:** HMAC-SHA256 via `jsonwebtoken`, secret from `TOKEN_SECRET` env var, claims: `id`, `email`, `name`
- **Password:** bcrypt with 10 salt rounds, compared using `bcrypt.compareSync()`, minimum 18 characters (enforced by CLI utility)
- **Rate limiting:** `express-rate-limit` on POST `/auth/login`, 5 attempts per 15 minutes per IP
- **Client:** React 19, react-router-dom v6, controlled form inputs with `useState`
- **Auth state:** React Context (`AuthContext`) with token verification on app load via `useEffect`
- **Route protection:** `IsPrivate` component wrapping children, `IsAnon` component wrapping login page
- **CORS:** Allowed origin from `ORIGIN` env var (dev: `http://localhost:3000`)
- **Token storage:** localStorage (key: `authToken`), user data in React Context state only
- **Navbar:** Shows key icon (login link) when logged out; shows `"Welcome, {name}"` and logout icon when logged in

### Out of Scope

- No signup/registration page -- admin user is seeded directly in the database
- No refresh token mechanism -- user must re-login after 6h
- No password reset/change in the UI -- handled via CLI utility (see above)
- No account lockout / brute force protection -- single-admin DoS risk outweighs benefit; mitigated by 18-char password minimum and IP rate limiting
- No email verification
- No user CRUD endpoints
- No role-based access control (single admin user)

## Tasks

### Server

**1. User Model (Prisma)**
**What:** User model in Prisma schema with id (cuid), email (unique), password (bcrypt hash), name, and createdAt timestamp.
**Files:**
- `server/prisma/schema.prisma` (User model, lines 73-79)

**2. JWT Middleware**
**What:** `express-jwt` middleware instance configured with TOKEN_SECRET, HS256 algorithm. Extracts Bearer token from Authorization header, validates it, and attaches decoded payload to `req.payload`.
**Files:**
- `server/src/middleware/jwt.middleware.ts`

**3. Auth Routes**
**What:** POST `/auth/login` -- rate-limited (5 attempts per 15 min per IP via `express-rate-limit`), validates non-empty email/password, looks up user by email, compares bcrypt password, signs JWT with {id, email, name} payload and 6h expiry. GET `/auth/verify` -- protected by `isAuthenticated` middleware, returns decoded token payload.
**Files:**
- `server/src/routes/auth.routes.ts`

**4. Custom Request Types**
**What:** `RequestWithPayload` type extending Express Request with `payload` property for decoded JWT data.
**Files:**
- `server/src/types/requests.ts`

**5. Route Registration**
**What:** Auth routes mounted at `/auth` prefix in app configuration.
**Files:**
- `server/src/app.ts`

**6. Password Change Utility**
**What:** Interactive CLI script using readline. Looks up user by email, validates new password (min 18 chars, confirmation match), bcrypt-hashes it (10 salt rounds), and updates the database via Prisma.
**Files:**
- `server/prisma/change-password.ts`

### Client

**7. Auth Service**
**What:** Singleton class with axios instance, base URL from `REACT_APP_SERVER_URL` env var. Request interceptor attaches Bearer token from localStorage. Methods: `login(requestBody)` -> POST `/auth/login`, `verify()` -> GET `/auth/verify`.
**Files:**
- `client/src/services/auth.service.js`

**8. Auth Context**
**What:** React Context providing `isLoggedIn`, `isLoading`, `user`, `storeToken`, `authenticateUser`, `logOutUser`. On mount, calls `verify()` to check existing token. `storeToken` saves to localStorage, `logOutUser` removes token and re-runs `authenticateUser`.
**Files:**
- `client/src/context/auth.context.jsx`

**9. Login Page**
**What:** Form with email (type="email") and password (type="password") inputs, both required. On submit, calls `authService.login()`, stores token, authenticates user, navigates to `/`. On error, displays server error message. Page title: "Admin Login", subtitle: "Sign in to manage your portfolio", footer: "Admin access only".
**Files:**
- `client/src/pages/LoginPage/LoginPage.jsx`

**10. Route Guards**
**What:** `IsPrivate` -- shows Loading spinner while auth is loading, redirects to `/login` if not authenticated, renders children if authenticated. `IsAnon` -- shows Loading while auth is loading, redirects to `/` if authenticated, renders children if not.
**Files:**
- `client/src/components/IsPrivate/IsPrivate.jsx`
- `client/src/components/IsAnon/IsAnon.jsx`

**11. Navbar Auth UI**
**What:** Left side: Home and About links (always visible). Right side: when logged out, shows key icon linking to `/login`; when logged in, shows "Welcome, {name}" text and logout icon button that calls `logOutUser`.
**Files:**
- `client/src/components/Navbar/Navbar.jsx`

**12. Router Configuration**
**What:** `/login` route wrapped in `<IsAnon>`. No routes currently use `<IsPrivate>` wrapper -- admin features are conditionally rendered within public pages based on `isLoggedIn` context value.
**Files:**
- `client/src/App.jsx`

## Current State

Fully implemented on both client and server. No existing tests.
