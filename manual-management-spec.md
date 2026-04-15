# Manual Management (Admin)

## Why

Each project can have multiple versioned manuals (e.g. "Getting Started v1", "Getting Started v2"). Only one manual can be active at a time -- the active manual's steps are displayed publicly on the project detail page. The admin needs to create, edit, delete, and toggle which manual is active.

## What

Manual management lives in the middle section of the ProjectPage (`/projects/:projectId`), visible only when logged in. Manuals are displayed as cards in a responsive grid. Each card supports inline editing (no modal). A "set active" radio button controls which manual's steps are shown publicly.

**User actions (admin only):**
- **View all manuals** for a project (full list with all versions)
- **Create manual:** Click "+ Create New Manual" -> inline form appears -> fill title, description, version -> click "Create" -> form resets, manual appears in grid
- **Edit manual:** Click pencil icon -> card switches to edit mode with pre-filled inputs -> modify fields -> click "Save" -> card returns to read mode
- **Delete manual:** Click X icon -> browser confirm dialog `"Are you sure you want to delete this manual?"` -> confirm -> manual removed
- **Set active:** Click radio button on a manual -> that manual becomes active (previous active is deactivated) -> its steps appear in the public bottom section

### API Endpoints

| Method | Endpoint | Auth | Request Body | Response | Status |
|--------|----------|------|-------------|----------|--------|
| GET | `/api/manuals/:projectId` | No | -- | `Manual[]` (select fields) | 200 |
| POST | `/api/manuals` | Yes | `RequestCreateManual` | `Manual` (with steps) | 201 |
| PATCH | `/api/manuals/:id` | Yes | `RequestUpdateManual` | `Manual` | 200 |
| PATCH | `/api/manuals/:projectId/:id/set-active` | Yes | -- | `Manual` (with steps) | 201 |
| DELETE | `/api/manuals/:id` | Yes | -- | `Manual` | 200 |

**GET response shape (list):** Returns only selected fields, sorted by `version` descending:
```json
{
  "id": "cuid",
  "title": "Getting Started",
  "description": "How to set up the project",
  "version": "1.0",
  "isActive": true
}
```

**Create request body:**
```json
{
  "projectId": "required",
  "title": "required",
  "description": "required",
  "version": "required"
}
```
`isActive` defaults to `false` on creation. Response includes empty `steps` array.

**Update request body:**
```json
{
  "title": "required",
  "description": "required",
  "version": "required"
}
```
Only updates `title`, `description`, `version`. Does not change `projectId` or `isActive`.

**Set active:** Uses a Prisma `$transaction` -- first sets ALL manuals for the project to `isActive: false`, then sets the target manual to `isActive: true`. Returns the activated manual with its steps.

**Delete:** Cascade deletes all `ManualStep` records belonging to the manual.

**Server-side validation:**
- Missing projectId on GET: `"Project ID is required"` (400)
- Missing manual ID on update: `"Manual ID is required"` (400)
- Missing fields on create/update: `"Error: some fields are missing"` (400)
- Missing IDs on set-active: `"Manual ID and Project ID are required"` (400)

**Server-side errors:**
- GET: `"Error retrieving project's manuals"` (500)
- Create: `"Error creating project's manual"` (500)
- Update: `"Error updating project's manual"` (500)
- Set active: `"Error setting active project's manual"` (500)
- Delete: `"Error deleting manual"` (500)

**Client-side errors:** `alert()` with `"Failed to update manual. Please try again."`, `"Failed to delete manual. Please try again."`, `"Failed to set active manual. Please try again."`, `"Failed to create manual. Please try again."`

### Database Schema

**Table: `ProjectManual`**

| Column | Type | Constraints |
|--------|------|------------|
| id | String | PK, cuid |
| projectId | String | FK to Project, NOT NULL |
| title | String | NOT NULL |
| description | String | NOT NULL |
| version | String | NOT NULL |
| isActive | Boolean | default false |

Relation: Belongs to Project (cascade delete). Has many ManualStep (cascade delete).

### Data Flow

1. **Fetch:** When ProjectPage mounts and `isLoggedIn` is true, fetches all manuals via `manualService.getAllByProject(projectId)` in the same `Promise.all` as the project data. If not logged in, resolves to empty array.
2. **Mutation:** After any create/update/delete/set-active, calls `fetchProjectData()` which re-fetches both the project and manuals.
3. **Active manual effect:** Setting a manual as active causes the project GET to include that manual's steps, which then render in the bottom section of the page.

### UI Layout

**Section container:** White card with "User Manuals" heading (h2). Only rendered when `isLoggedIn`.

**Grid layout:** Responsive grid -- 1 column (mobile), 2 columns (md), 3 columns (lg), 4 columns (xl) with `gap-4`.

**Manual card (read mode):**
- Edit button (blue, pencil icon) and Delete button (red, X icon) at the top
- Manual title (bold)
- Description (small gray text)
- Version label: `"Version: {version}"` (extra small gray text)
- Radio button with "Active" label -- checked if `manual.isActive`, calls `handleSetActive` on change. All radio buttons share the name `"activeManual"`.

**Manual card (edit mode):**
- Title input (text, placeholder: "Title")
- Description textarea (3 rows, placeholder: "Description")
- Version input (text, placeholder: "Version")
- "Save" button (green) and "Cancel" button (gray) side by side

**Create new manual card:**
- Default state: Dashed border card with "+ Create New Manual" button (green)
- Form state: Green-tinted card ("New Manual" heading) with title input, description textarea, version input, "Create" button (green), and "Cancel" button (gray)
- Cancel resets form data to empty and hides the form

## Constraints

### Must

- **Auth required:** All CUD endpoints protected by `isAuthenticated` middleware. GET is public but only called when logged in.
- **Inline editing:** Edit mode replaces the card content in-place (no modal)
- **Single active:** Only one manual per project can be active at a time, enforced by the `$transaction` on set-active
- **Radio group:** All manual radio buttons share `name="activeManual"` so only one can be checked
- **Cascade delete:** Deleting a manual cascades to all its ManualStep records
- **Confirm dialog:** Delete uses native `window.confirm()`
- **Version sorting:** Manual list sorted by `version` descending (from API)
- **Refresh on mutation:** All mutations trigger `fetchProjectData()` to refresh both project and manuals

### Out of Scope

- No manual duplication/cloning
- No manual reordering (sorted by version)
- No manual versioning automation (version is a free-text field)
- No bulk operations
- No manual preview before setting active
- No undo for delete

## Tasks

### Server

**1. Manual GET Route**
**What:** GET `/api/manuals/:projectId` -- public. Returns all manuals for a project with selected fields (`id`, `title`, `description`, `version`, `isActive`), sorted by `version` descending.
**Files:**
- `server/src/routes/manual.routes.ts` (lines 11-28)

**2. Manual Create Route**
**What:** POST `/api/manuals` -- protected. Validates required fields (`projectId`, `title`, `description`, `version`). Creates manual with `isActive: false`, returns with steps included.
**Files:**
- `server/src/routes/manual.routes.ts` (lines 118-140)

**3. Manual Update Route**
**What:** PATCH `/api/manuals/:id` -- protected. Validates ID and required fields. Updates only `title`, `description`, `version`.
**Files:**
- `server/src/routes/manual.routes.ts` (lines 31-57)

**4. Manual Set Active Route**
**What:** PATCH `/api/manuals/:projectId/:id/set-active` -- protected. Uses `$transaction` to set all project manuals to inactive, then set the target to active. Returns activated manual with steps.
**Files:**
- `server/src/routes/manual.routes.ts` (lines 61-99)

**5. Manual Delete Route**
**What:** DELETE `/api/manuals/:id` -- protected. Deletes manual by ID, cascade removes all ManualStep records.
**Files:**
- `server/src/routes/manual.routes.ts` (lines 103-115)

**6. Request Type Definitions**
**What:** `RequestCreateManual` (body: `projectId`, `title`, `description`, `version`) and `RequestUpdateManual` (body: `title`, `description`, `version`).
**Files:**
- `server/src/types/requests.ts` (lines 47-62)

### Client

**7. Manual Service**
**What:** Singleton class with JWT-intercepted axios instance. Methods: `getAllByProject(projectId)`, `create(manualData)`, `update(id, manualData)`, `setActive(projectId, manualId)`, `delete(id)`.
**Files:**
- `client/src/services/manual.service.js`

**8. ProjectPage Manual Section**
**What:** Middle section of ProjectPage, only rendered when `isLoggedIn`. Manages state for editing (`editingManualId`, `editFormData`), creating (`showCreateForm`, `createFormData`). Renders manual cards in responsive grid with read/edit modes, radio buttons for active state, and inline create form.
**Files:**
- `client/src/pages/ProjectPage/ProjectPage.jsx` (lines 26-33, 70-141, 287-484)

## Current State

Fully implemented on both client and server. No existing tests.
