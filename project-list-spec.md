# Project List (Public View)

## Why

The project list is the core of the portfolio -- it shows visitors what the developer has built. Each project is presented as a card with an image, name, description, and tech stack logos, giving a quick overview before clicking into details.

## What

The project list is rendered below the hero section on the home page (`/`). Projects are fetched from the API on page load (in the same `Promise.all` as technologies -- see hero-technologies-spec). Each project renders as a clickable card that navigates to the project detail page.

**User actions:**
- **View project cards** -- each card shows project name, description, image, and tech stack logos
- **Click a card** -> navigates to `/projects/:projectId`
- **Admin only: "Add New Project" button** -- visible when logged in (covered in project-crud-spec)
- **Admin only: Edit/Delete buttons** on each card -- visible when logged in (covered in project-crud-spec)

### API Endpoint (Public)

| Method | Endpoint | Auth | Response | Status |
|--------|----------|------|----------|--------|
| GET | `/api/projects` | No | `Project[]` | 200 |

**Project response shape (list):**
```json
{
  "id": "cuid",
  "name": "Project Name",
  "server_github_url": "https://..." | null,
  "client_github_url": "https://...",
  "server_deploy_url": "https://..." | null,
  "client_deploy_url": "https://...",
  "description_short": "Short description text",
  "image_url": "https://...",
  "sort_order": 1 | null,
  "techStack": [
    {
      "id": "cuid",
      "projectId": "cuid",
      "technologyId": "cuid",
      "technology": {
        "id": "cuid",
        "name": "React",
        "logo_url": "https://...",
        "official_site_url": "https://...",
        "categoryId": "cuid",
        "sort_order": 1
      }
    }
  ]
}
```

Projects are returned sorted by `sort_order` descending.

**Server-side error:** `"Error retrieving all projects"` (500)

### Database Schema

**Table: `Project`**

| Column | Type | Constraints |
|--------|------|------------|
| id | String | PK, cuid |
| name | String | UNIQUE, NOT NULL |
| server_github_url | String | nullable |
| client_github_url | String | NOT NULL |
| server_deploy_url | String | nullable |
| client_deploy_url | String | NOT NULL |
| description_short | String | NOT NULL |
| image_url | String | NOT NULL |
| sort_order | Int | nullable |

**Table: `ProjectTechStack` (junction)**

| Column | Type | Constraints |
|--------|------|------------|
| id | String | PK, cuid |
| projectId | String | FK to Project, NOT NULL |
| technologyId | String | FK to Technology, NOT NULL |

Unique constraint on `(projectId, technologyId)`. Cascade delete on project deletion.

### ProjectsSection Layout

- **Title:** "My Projects" (h2, centered)
- **Subtitle:** "A collection of my recent work showcasing various technologies and skills" (centered)
- **"Add New Project" button:** Only visible when `isLoggedIn` is true (admin feature, see project-crud-spec)
- **Project cards:** Rendered in a vertical flex column with `gap-6`

### ProjectCard Layout

Each card is a clickable container (`cursor-pointer`) with hover lift effect (`hover:-translate-y-1 hover:shadow-lg`).

**Responsive layout:**
- **Mobile:** Image on top, then name + tech stack + description stacked below
- **Desktop (`md:`):** Two-column -- left column (name, tech stack logos, description), right column (image, `w-64` fixed width)

**Card content:**
1. **Project name** (h3, `text-2xl font-semibold`)
2. **Tech stack logos** -- flex-wrap row of small logos (`w-6 h-6`), each with `alt` and `title` set to `technology.name`
3. **Short description** -- justified text
4. **Project image** -- `object-cover`, rounded corners. Mobile: `h-48` full width. Desktop: `h-40` in 256px column

**Admin-only overlay (when logged in):**
- Edit button (pencil icon) and Delete button (X icon) positioned absolute `top-2 right-2`
- `e.stopPropagation()` on both to prevent card click navigation
- Delete button is red, edit button is white with border
- These buttons trigger callbacks covered in project-crud-spec

### Loading & Error States

Handled in `HomePage` (shared with hero section):
- **Loading:** Full-page `Loading` spinner
- **Error:** `"Failed to load projects or technologies. Please try again later."`

## Constraints

### Must

- **Data fetching:** Projects fetched in `HomePage` alongside technologies via `Promise.all`
- **Props:** `ProjectsSection` receives `projects` array and admin callbacks (`onAddProject`, `onEditProject`, `onDeleteProject`). `ProjectCard` receives `project`, `onEdit`, `onDelete`
- **Navigation:** Clicking a card calls `navigate(`/projects/${project.id}`)` via React Router `useNavigate`
- **Auth-aware:** Both `ProjectsSection` and `ProjectCard` consume `AuthContext` to conditionally render admin UI
- **Sort order:** Projects displayed in `sort_order` descending (determined by API)
- **Nested include:** API returns `techStack` with nested `technology` object for each relation

### Must Not

- Do not fetch projects inside `ProjectsSection` or `ProjectCard`; the parent `HomePage` owns the request
- Do not require auth on GET `/api/projects`; it must remain public
- Do not render admin-only controls (Add/Edit/Delete) when `isLoggedIn` is false
- Do not omit the nested `techStack.technology` include — the card relies on it for logo rendering
- Do not add a default value for missing `sort_order`; nulls are preserved and the API leaves them at the end of desc ordering

### Out of Scope

- No search or filtering of projects
- No pagination (all projects loaded at once)
- No skeleton loading states (uses spinner)
- No empty state message when there are zero projects
- GitHub/deploy URLs are fetched but not displayed on the card (only on the detail page)

## Tasks

### Server

**1. Project & ProjectTechStack Models (Prisma)**
**What:** Project model with all fields and relations to ProjectTechStack (many-to-many through junction) and ProjectManual. ProjectTechStack with unique constraint on `(projectId, technologyId)` and cascade delete.
**Files:**
- `server/prisma/schema.prisma` (Project model, lines 28-39; ProjectTechStack model, lines 41-49)

**Verify:** No dedicated test; indirectly covered by `npm test -- projects` (in `server/`) — schema mismatches would break list/detail/CRUD tests.

**2. Project GET All Route**
**What:** GET `/api/projects` -- public endpoint returning all projects with nested `techStack -> technology` includes, sorted by `sort_order` descending.
**Files:**
- `server/src/routes/project.routes.ts` (lines 9-26)

**Verify:** `npm test -- projects` (in `server/`) — covers sort order desc and nested `techStack.technology` shape.

### Client

**3. Project Service**
**What:** Singleton class with axios instance and JWT interceptor. `getAll()` calls GET `/api/projects`. Also exposes `getOne()`, `create()`, `update()`, `delete()` (used elsewhere).
**Files:**
- `client/src/services/project.service.js`

**Verify:** No dedicated test; indirectly covered by `npm test -- project-list` (in `tests/`) — the home page only renders project cards if `getAll()` succeeded.

**4. HomePage Data Fetching**
**What:** Fetches projects and technologies in parallel via `Promise.all` on mount. Passes `projects` array and admin handler callbacks to `ProjectsSection`.
**Files:**
- `client/src/pages/HomePage/HomePage.jsx` (lines 20-37, 116-134)

**Verify:** No dedicated test; indirectly covered by `npm test -- project-list` (in `tests/`) — sort order and card visibility only pass once `Promise.all` resolves and the projects array reaches `ProjectsSection`. Loading and error branches are not asserted.

**5. ProjectsSection Component**
**What:** Renders "My Projects" heading, subtitle, and conditionally the "Add New Project" button (admin only). Maps `projects` array to `ProjectCard` components in a vertical flex column.
**Files:**
- `client/src/components/ProjectsSection/ProjectsSection.jsx`

**Verify:** `npm test -- project-list` (in `tests/`) — covers the heading + subtitle, the full set of seeded cards rendered in `sort_order` desc, and the "Add New Project" button being absent when logged out.

**6. ProjectCard Component**
**What:** Clickable card that navigates to `/projects/:id`. Renders project image (responsive mobile/desktop), name, tech stack logos, and short description. When logged in, shows edit (pencil) and delete (X) icon buttons with `stopPropagation`. Uses Heroicons (`PencilSquareIcon`, `XMarkIcon`).
**Files:**
- `client/src/components/ProjectCard/ProjectCard.jsx`

**Verify:** `npm test -- project-list` (in `tests/`) — covers project name rendering, click → `/projects/:id` navigation, and absence of Edit/Delete icons when logged out. Tech logo rendering and the responsive image switch are not directly asserted (covered indirectly via card visibility).

## Validation

End-to-end verification after all tasks complete.

### Automated checks

- Server-side: `npm test -- projects` (in `server/`) — covers GET `/api/projects` (sort order, nested includes)
- E2E: `npm test -- project-list` (in `tests/`) — covers section header, all seeded cards rendered in `sort_order` desc, click-to-navigate, and admin UI hidden when logged out
- Full server suite: `npm test` (in `server/`); full E2E suite: `npm test` (in `tests/`)

### Manual checks (UI)

1. Visit `/` → projects render below the hero section, sorted by `sort_order` desc (highest first)
2. Each card shows: name, tech stack logos, short description, image
3. Logos have `title` and `alt` matching technology name (hover to verify)
4. Click a card → navigates to `/projects/:id`
5. Logged out: no "Add New Project" button, no per-card pencil/X overlays
6. Logged in: "Add New Project" button visible above the list; pencil/X overlays appear on each card; clicking pencil/X does not trigger the card's navigate handler (`stopPropagation`)
7. Disable network → spinner appears, then error: "Failed to load projects or technologies. Please try again later."

### Cross-feature dependencies

- `hero-technologies-spec.md` — both fetched together via `Promise.all` in `HomePage`
- `project-details-spec.md` — destination of card clicks
- `project-crud-spec.md` — Add/Edit/Delete callbacks come from there
- `auth-spec.md` — `AuthContext` drives admin UI visibility
- Seeded `Project` rows in `server/prisma/seed.ts` — needed for any test that reads or modifies projects

## Current State

Fully implemented on both client and server.

**Tests in place:**
- `server/tests/projects.test.ts` covers GET `/api/projects` (sort order desc, nested techStack.technology shape)
- `tests/specs/project-list.spec.ts` — 5 E2E tests: section header + subtitle, all 5 seeded projects render, sort_order desc verified across the 4 numbered projects, click → `/projects/:id`, admin UI absent when logged out

**Untested:**
- Loading and error branches of `HomePage` (race conditions; would need network throttling)
- Tech logo rendering per card and the mobile/desktop responsive image switch (visual concerns; low value to assert)
- Position of `Yoga Path` (sort_order=null) in the desc-ordered list (Postgres NULLS-FIRST default; not worth pinning)
