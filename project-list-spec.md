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

**2. Project GET All Route**
**What:** GET `/api/projects` -- public endpoint returning all projects with nested `techStack -> technology` includes, sorted by `sort_order` descending.
**Files:**
- `server/src/routes/project.routes.ts` (lines 9-26)

### Client

**3. Project Service**
**What:** Singleton class with axios instance and JWT interceptor. `getAll()` calls GET `/api/projects`. Also exposes `getOne()`, `create()`, `update()`, `delete()` (used elsewhere).
**Files:**
- `client/src/services/project.service.js`

**4. HomePage Data Fetching**
**What:** Fetches projects and technologies in parallel via `Promise.all` on mount. Passes `projects` array and admin handler callbacks to `ProjectsSection`.
**Files:**
- `client/src/pages/HomePage/HomePage.jsx` (lines 20-37, 116-134)

**5. ProjectsSection Component**
**What:** Renders "My Projects" heading, subtitle, and conditionally the "Add New Project" button (admin only). Maps `projects` array to `ProjectCard` components in a vertical flex column.
**Files:**
- `client/src/components/ProjectsSection/ProjectsSection.jsx`

**6. ProjectCard Component**
**What:** Clickable card that navigates to `/projects/:id`. Renders project image (responsive mobile/desktop), name, tech stack logos, and short description. When logged in, shows edit (pencil) and delete (X) icon buttons with `stopPropagation`. Uses Heroicons (`PencilSquareIcon`, `XMarkIcon`).
**Files:**
- `client/src/components/ProjectCard/ProjectCard.jsx`

## Current State

Fully implemented on both client and server. No existing tests.
