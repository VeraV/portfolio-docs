# Project Details Page (Public View)

## Why

Visitors who click a project card need a detailed view -- the full description, tech stack, links to the live site and GitHub repos, and a step-by-step manual (if one is active). This page is the deep-dive into each portfolio project.

## What

A public page at `/projects/:projectId` showing full project information and the active manual's steps. The page fetches a single project from the API, which includes the tech stack and the active manual with its steps.

**User actions:**
- **View project details** -- image, name, description, tech stack logos
- **Click external links** -- "View Live Site", "Client GitHub", "Server GitHub" (if exists), "Server Deploy" (if exists) -- all open in new tabs
- **Read manual steps** -- if an active manual exists, its steps are displayed as a vertical timeline

### API Endpoint (Public)

| Method | Endpoint | Auth | Response | Status |
|--------|----------|------|----------|--------|
| GET | `/api/projects/:id` | No | `Project` (with techStack, active manuals, steps) | 200 |

**Project detail response shape:**
```json
{
  "id": "cuid",
  "name": "Project Name",
  "description_short": "Description text",
  "image_url": "https://...",
  "client_github_url": "https://...",
  "client_deploy_url": "https://...",
  "server_github_url": "https://..." | null,
  "server_deploy_url": "https://..." | null,
  "sort_order": 1 | null,
  "techStack": [
    {
      "id": "cuid",
      "technology": {
        "id": "cuid",
        "name": "React",
        "logo_url": "https://...",
        "official_site_url": "https://..."
      }
    }
  ],
  "manuals": [
    {
      "id": "cuid",
      "title": "Getting Started",
      "isActive": true,
      "steps": [
        {
          "id": "cuid",
          "step_number": 1,
          "description": "Step description",
          "image_url": "https://..."
        }
      ]
    }
  ]
}
```

**Key detail:** The GET `/api/projects/:id` endpoint only includes manuals where `isActive: true`. Steps within each manual are ordered by `step_number` ascending.

**Server-side errors:**
- Missing ID: `"Project ID is required"` (400)
- Not found: `"Project not found"` (404)
- Server error: `"Error retrieving the project"` (500)

**Client-side error:** `"Failed to load project. Please try again later."`

### Data Flow

On mount, `ProjectPage` fetches data via `Promise.all`:
1. `projectService.getOne(projectId)` -- always fetched
2. `manualService.getAllByProject(projectId)` -- only fetched if `isLoggedIn` (for admin manual management, covered in manual-management-spec). For public visitors, resolves to empty array.

Re-fetches when `projectId` or `isLoggedIn` changes.

### Page Layout (Three Sections)

#### Top Section -- Project Details

A white card with a 3-column grid (stacks on mobile):

**Column 1 -- Image:**
- Project image (`h-64`, `object-cover`, rounded, bordered)

**Column 2 -- Info:**
- Project name (h1, `text-3xl font-bold`)
- Short description paragraph
- "Technologies:" label with tech stack logos (`w-8 h-8` each, flex-wrap row with `title` and `alt` set to technology name)

**Column 3 -- Links:**
- "Project Links:" label
- "View Live Site" button (blue) -- always shown, links to `client_deploy_url`
- "Client GitHub" button (dark gray) -- always shown, links to `client_github_url`
- "Server GitHub" button (gray) -- only shown if `server_github_url` exists
- "Server Deploy" button (gray) -- only shown if `server_deploy_url` exists
- All links open in new tab (`target="_blank"`, `rel="noopener noreferrer"`)

#### Middle Section -- User Manuals (Admin Only)

Only visible when `isLoggedIn`. Covered in manual-management-spec.

#### Bottom Section -- Manual Steps

Visible when there are steps to show OR when the admin is logged in. Shows the active manual's steps as a vertical timeline.

**Title:** `"{manual.title} - Steps"` if an active manual exists, otherwise `"Manual Steps"`.

**Steps:** Rendered via `StepItem` components (see manual-steps-spec). Each step shows a step number circle, description, and image in a timeline layout.

**Admin-only elements** within this section (add step button, create step form) are covered in manual-steps-spec.

### Loading & Error States

- **Loading:** Full-page `Loading` spinner
- **Error:** Centered red-bordered message: `"Failed to load project. Please try again later."`
- **Not found:** Centered text: `"Project not found"` (when API returns 404 or project is null)

## Constraints

### Must

- **Route param:** `projectId` extracted from URL via `useParams()`
- **Auth-aware fetching:** Manual list only fetched when logged in; public visitors get project data only
- **Conditional links:** Server GitHub and Server Deploy links only rendered when the corresponding URL exists (not null)
- **Active manual only:** The API filters manuals to `isActive: true` -- the public view never sees inactive manuals
- **Step ordering:** Steps sorted by `step_number` ascending (determined by API)
- **External links:** All project links use `<a>` with `target="_blank"` and `rel="noopener noreferrer"`

### Out of Scope

- No project editing from this page (edit is on the home page card)
- No previous/next project navigation
- No comments or feedback section
- No sharing functionality
- No full/long description field (only `description_short`)

## Tasks

### Server

**1. Project GET One Route**
**What:** GET `/api/projects/:id` -- public. Validates ID presence, finds project by ID with nested includes: `techStack -> technology` and `manuals` (where `isActive: true`) with `steps` ordered by `step_number` ascending. Returns 404 if not found.
**Files:**
- `server/src/routes/project.routes.ts` (lines 28-55)

### Client

**2. Project Service (getOne method)**
**What:** `getOne(id)` -> GET `/api/projects/:id`. Uses JWT-intercepted axios instance.
**Files:**
- `client/src/services/project.service.js` (lines 24-26)

**3. ProjectPage Component (public sections)**
**What:** Fetches project data on mount. Renders three-section layout: project details card (image, info, links), admin manual management (conditional), and manual steps timeline. Handles loading, error, and not-found states.
**Files:**
- `client/src/pages/ProjectPage/ProjectPage.jsx`

## Current State

Fully implemented on both client and server. No existing tests.
