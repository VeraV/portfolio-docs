# Public Navigation & Layout

## Why

Every page shares a common layout with a persistent navbar. Users need to navigate between the home page, about page, and project detail pages. The navbar also serves as the entry point for admin login/logout (covered in auth-spec).

## What

A top-level layout consisting of a persistent Navbar rendered above all routes. Navigation uses React Router v6 client-side routing. There is no catch-all 404 route -- unknown URLs render an empty content area below the navbar.

**User actions:**
- **Navigate to Home** via "Home" button in navbar -> renders HomePage at `/`
- **Navigate to About** via "About" button in navbar -> renders ProfilePage at `/about`
- **Navigate to a project** by clicking a project card on the home page -> renders ProjectPage at `/projects/:projectId`
- **Visit unknown URL** -> renders NotFoundPage with "Page Not Found" heading

### Routes

| Path | Component | Access | Guard |
|------|-----------|--------|-------|
| `/` | HomePage | Public | -- |
| `/about` | ProfilePage | Public | -- |
| `/projects/:projectId` | ProjectPage | Public | -- |
| `/login` | LoginPage | Anonymous only | `<IsAnon>` |
| `*` | NotFoundPage | Public | -- |

### Navbar Layout

The navbar is always visible on every page.

**Left side (always visible):**
- "Home" button -- links to `/`
- "About" button -- links to `/about`

**Right side (depends on auth state):**
- **Logged out:** Key icon button linking to `/login`
- **Logged in:** "Welcome, {name}" text + logout icon button (see auth-spec for details)

**Styling:** White background with shadow (`bg-white shadow-md`), max-width container (`max-w-7xl`), horizontally centered. Buttons are plain text with hover color transition to blue.

### Loading Component

A full-viewport loading spinner used by route guards (`IsPrivate`, `IsAnon`) while authentication state is being resolved.

**Visual:** Three bouncing dark circles (`#333`) centered on screen, animated with staggered delays.

### NotFoundPage

Rendered by a catch-all `path="*"` route. Any URL that doesn't match a defined route falls through to this page. The navbar is still visible (since it's rendered outside of `<Routes>`).

**Content:**
- Heading: "Page Not Found"
- Paragraph: "This page doesn't seem to exist"

## Constraints

### Must

- **Router:** React Router v6 with `<Routes>` and `<Route>` components
- **Navbar persistence:** Rendered outside of `<Routes>` in `App.jsx`, so it appears on every page
- **Client-side navigation:** All navbar links use React Router `<Link>` component (no full page reloads)
- **Auth-aware navbar:** Consumes `AuthContext` to toggle between login icon and welcome/logout UI
- **Loading spinner:** Used by `IsPrivate` and `IsAnon` guards during auth loading state

### Must Not

- Do not render the Navbar inside `<Routes>` — it must persist across route changes
- Do not use `<a href>` for in-app navigation; only `<Link>` (full page reloads break the SPA experience)
- Do not add a 404-redirect for unknown URLs — the catch-all renders `NotFoundPage` in place
- Do not introduce additional navbar entries without updating this spec (single source of truth)
- Do not couple `IsPrivate` / `IsAnon` to the loading spinner via direct DOM access; they consume it as a component

### Out of Scope

- No breadcrumbs or secondary navigation
- No footer component
- No sidebar or drawer navigation
- No active link highlighting (current page not visually indicated in navbar)
- No mobile hamburger menu / responsive nav collapse

## Tasks

### Client

**1. App Shell & Router**
**What:** Top-level component rendering `<Navbar />` above `<Routes>`. Defines five routes: `/` (HomePage), `/about` (ProfilePage), `/projects/:projectId` (ProjectPage), `/login` (LoginPage wrapped in `<IsAnon>`), and `*` catch-all (NotFoundPage).
**Files:**
- `client/src/App.jsx`

**Verify:** No dedicated test; indirectly covered by `npm test -- auth` (in `tests/`) — the login route only loads if the router and `<IsAnon>` wrapper are wired correctly.

**2. Navbar**
**What:** Persistent navigation bar. Left side: Home and About links using `<Link>`. Right side: auth-aware UI consuming `AuthContext` -- key icon when logged out, welcome message and logout button when logged in. Uses Heroicons (`KeyIcon`, `ArrowRightStartOnRectangleIcon`).
**Files:**
- `client/src/components/Navbar/Navbar.jsx`

**Verify:** Partially covered by `npm test -- auth` (in `tests/`) — the auth E2E asserts welcome message + logout button after login and key icon after logout. Home/About link clicks are not asserted.

**3. Loading Spinner**
**What:** Full-viewport centered spinner with three bouncing circles. Used as loading state in route guard components.
**Files:**
- `client/src/components/Loading/Loading.jsx`
- `client/src/components/Loading/Loading.css`

**Verify:** No tests cover this task yet.

**4. NotFoundPage**
**What:** Simple page component with "Page Not Found" heading and "This page doesn't seem to exist" paragraph. Rendered by the catch-all `path="*"` route.
**Files:**
- `client/src/pages/NotFoundPage/NotFoundPage.jsx`
- `client/src/pages/NotFoundPage/NotFoundPage.css`

**Verify:** No tests cover this task yet.

## Validation

End-to-end verification after all tasks complete.

### Automated checks

- E2E: `npm test -- auth` (in `tests/`) — partially exercises Navbar's auth-aware right-hand side
- No dedicated navigation E2E spec yet (no test for clicking Home/About, no test for the 404 page)

### Manual checks (UI)

1. Visit `/` → Navbar visible with "Home", "About", and key icon (when logged out)
2. Click "About" → navigates to `/about` without a full page reload (no spinner from the browser)
3. Click "Home" → returns to `/`
4. Visit a non-existent URL like `/whatever` → NotFoundPage renders ("Page Not Found"), Navbar still visible
5. Log in → Navbar right side switches to "Welcome, Admin" + logout icon
6. Log out → Navbar right side reverts to key icon
7. Visit `/login` while logged in → IsAnon redirects to `/` (covered in auth-spec)

### Cross-feature dependencies

- `auth-spec.md` — Navbar's right side is auth-aware; `<IsAnon>` and `<IsPrivate>` route guards are defined there
- Every other spec depends on this one — `/about`, `/`, `/projects/:id`, `/login` routes registered here
- `Loading` component is consumed by `IsPrivate` and `IsAnon` (auth-spec) during auth state resolution

## Current State

Fully implemented on the client.

**Tests in place:**
- Implicit Navbar coverage from `tests/specs/auth.spec.ts` (welcome message, logout button, key icon assertions)

**Untested:**
- Home/About link click navigation
- NotFoundPage rendering on unknown URLs
- Loading spinner visual / mount behavior
