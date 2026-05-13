# Manual Steps

## Why

Steps are the content of a manual -- a sequential, visual walkthrough of how to use or set up a project. Each step has a description and a screenshot. Public visitors see the active manual's steps as a timeline on the project detail page. The admin can add new steps and edit existing ones inline.

## What

Steps are displayed in the bottom section of the ProjectPage (`/projects/:projectId`). The public view shows a read-only timeline. The admin view adds inline editing per step and a form to create new steps. Steps belong to the active manual and are auto-numbered on creation.

### Public View

The steps section is visible when an active manual has steps OR when the admin is logged in.

**Title:** `"{manual.title} - Steps"` if an active manual exists, otherwise `"Manual Steps"`.

**Timeline layout:** Each step is a two-column row (stacks on mobile):
- **Left column:** Step number circle (blue, `w-12 h-12`) with a vertical connecting line (`bg-blue-300`) between steps (hidden on the last step). Below the circle: step description text.
- **Right column:** Step image (`max-w-md`, rounded, bordered, with shadow).

### Admin View

**User actions (admin only):**
- **Edit step:** Click pencil icon next to description -> step switches to edit mode -> modify description and/or upload new image via Cloudinary -> click "Save" -> step returns to read mode, data refreshes
- **Cancel edit:** Click "Cancel" -> reverts to original values, returns to read mode
- **Add step:** Click green "+" circle button below the last step -> inline create form appears -> fill description, upload image via Cloudinary -> click "Save" -> form resets, step appears in timeline
- **Cancel add:** Click "Cancel" -> form hides, data resets

### API Endpoints

| Method | Endpoint | Auth | Request Body | Response | Status |
|--------|----------|------|-------------|----------|--------|
| POST | `/api/steps` | Yes | `RequestCreateManualStep` | `ManualStep` | 201 |
| PATCH | `/api/steps/:stepId` | Yes | `RequestUpdateStep` | `ManualStep` | 200 |

Steps are not fetched independently -- they come nested in the project GET response (within the active manual).

**Create request body:**
```json
{
  "manualId": "required",
  "description": "required",
  "image_url": "required"
}
```
`step_number` is auto-calculated on the server: finds the current max `step_number` for the manual and increments by 1. Uses a Prisma `$transaction` to prevent race conditions.

**Update request body:**
```json
{
  "description": "required",
  "image_url": "required"
}
```
Updates only `description` and `image_url`. Does not change `step_number` or `manualId`.

**Server-side validation:**
- Create missing fields: `"All step fields are required."` (400)
- Update missing fields: `"Description and image are required."` (400)

**Server-side errors:**
- Create: `"Error while adding a new manual step"` (500)
- Update: `"Error updating manual step"` (500)

**Client-side errors:** `alert()` with `"Failed to create step. Please try again."` or `"Failed to update step. Please try again."`

### Database Schema

**Table: `ManualStep`**

| Column | Type | Constraints |
|--------|------|------------|
| id | String | PK, cuid |
| manualId | String | FK to ProjectManual, NOT NULL |
| step_number | Int | NOT NULL |
| description | String | NOT NULL |
| image_url | String | NOT NULL |

Unique constraint on `(manualId, step_number)`. Cascade delete when parent manual is deleted.

### StepItem Component

Each step is rendered by the `StepItem` component, which manages its own edit state independently.

**Props:** `step`, `isLastStep`, `isAdmin`, `onStepUpdated`

**Read mode (two-column grid):**
- Left: Step number circle -> description text. Admin sees a pencil icon button next to the description.
- Right: Step image (`alt="Step {step_number}"`). Image is hidden during edit mode.
- Connecting line: Vertical blue line from below the circle to the bottom of the row. Hidden on last step (`isLastStep`).

**Edit mode (replaces left column content):**
- Description textarea (4 rows, pre-filled)
- Cloudinary upload widget (pre-filled with current image)
- "Save" (green) and "Cancel" (gray) buttons
- Right column image is hidden during editing

**On save:** Calls `stepService.update(step.id, editData)`, then `onStepUpdated()` (which triggers `fetchProjectData()` in ProjectPage).
**On cancel:** Resets `editData` to original step values.

### Create Step Form (in ProjectPage)

Rendered below the last step when `showCreateStepForm` is true. Only available when logged in.

**Trigger:** Green circular "+" button (`w-12 h-12`, `PlusIcon`) centered below the step list. Hidden when the create form is open.

**Pre-condition:** Requires an active manual. If no active manual exists, shows `alert("No active manual found. Please set a manual as active first.")`.

**Form fields:**
1. **Description** -- textarea (4 rows), required, placeholder: "Describe this step..."
2. **Image** -- Cloudinary upload widget

**Form actions:**
- "Save" button (green) -- disabled when `description` or `image_url` is empty (`disabled:bg-gray-400 disabled:cursor-not-allowed`)
- "Cancel" button (gray) -- hides form, resets data

**On submit:** Creates step with `manualId` from the active manual, then resets form and refreshes data.

## Constraints

### Must

- **Auth required:** Both POST and PATCH endpoints protected by `isAuthenticated` middleware
- **Auto-numbering:** Step number calculated server-side using `aggregate` max + 1 in a `$transaction`
- **Unique step numbers:** Enforced by `@@unique([manualId, step_number])` constraint
- **Active manual required:** Creating a step requires an active manual (checked client-side)
- **Inline editing:** StepItem manages its own edit state, no modal
- **Cloudinary images:** Both create and edit use the Cloudinary upload widget for step images
- **Disabled submit:** Create form's "Save" button is disabled until both description and image are provided
- **Cascade delete:** Steps are deleted when their parent manual is deleted
- **Refresh on mutation:** Both create and update trigger `fetchProjectData()` to refresh the entire page

### Must Not

- Do not let the client supply a `step_number` on create — the server controls it inside the `$transaction`
- Do not allow PATCH to mutate `step_number` or `manualId`; only `description` and `image_url` are editable
- Do not expose POST/PATCH without `isAuthenticated`
- Do not render an empty steps section to the public when no active manual has steps (visible only when admin is logged in or steps exist)
- Do not allow creating a step when no active manual exists; the client must alert and refuse

### Out of Scope

- No step deletion (individual steps cannot be removed)
- No step reordering (step numbers are fixed once created)
- No step duplication
- No bulk step creation
- No step description formatting (plain text only)
- No image-only or description-only steps (both are always required)

## Tasks

### Server

**1. Step Create Route**
**What:** POST `/api/steps` -- protected. Validates required fields (`manualId`, `description`, `image_url`). Uses `$transaction` to find max `step_number` for the manual and create the new step with `step_number = max + 1`.
**Files:**
- `server/src/routes/step.routes.ts` (lines 9-41)

**Verify:** `npm test -- steps` (in `server/`) — covers 401, 400, auto-incremented `step_number` (existing → max+1), and `step_number = 1` for an empty manual.

**2. Step Update Route**
**What:** PATCH `/api/steps/:stepId` -- protected. Validates required fields (`description`, `image_url`). Updates only description and image_url.
**Files:**
- `server/src/routes/step.routes.ts` (lines 43-60)

**Verify:** `npm test -- steps` (in `server/`) — covers 401, 400, and that `step_number` and `manualId` stay unchanged across the update.

**3. Request Type Definitions**
**What:** `RequestCreateManualStep` (body: `manualId`, `description`, `image_url`) and `RequestUpdateStep` (body: `description`, `image_url`).
**Files:**
- `server/src/types/requests.ts` (lines 64-77)

**Verify:** No tests — type-only, intentionally not covered.

### Client

**4. Step Service**
**What:** Singleton class with JWT-intercepted axios instance. Methods: `create(stepData)` -> POST `/api/steps`, `update(stepId, stepData)` -> PATCH `/api/steps/:stepId`.
**Files:**
- `client/src/services/step.service.js`

**Verify:** No dedicated test; indirectly covered by `npm test -- manual-steps` (in `tests/`) — the create and edit flows only succeed if `create()` and `update()` reach the API with the JWT.

**5. StepItem Component**
**What:** Renders a single step in timeline layout. Manages own `isEditing` state and `editData`. Read mode: step number circle, description (with admin edit button), image, connecting line. Edit mode: textarea, Cloudinary upload, save/cancel buttons. On save calls `stepService.update()` then `onStepUpdated()`.
**Files:**
- `client/src/components/StepItem/StepItem.jsx`

**Verify:** `npm test -- manual-steps` (in `tests/`) — covers admin pencil visibility per step, edit flow with description change (read → edit → save → back to read), and cancel flow reverting to original. The image-load conditional and connecting-line behaviour on the last step are not directly asserted.

**6. ProjectPage Step Section**
**What:** Bottom section of ProjectPage. Renders `StepItem` components for each step in the active manual. Admin-only: green "+" button to open create form, inline form with description textarea, Cloudinary upload, and disabled-until-complete save button. Validates active manual exists before creating.
**Files:**
- `client/src/pages/ProjectPage/ProjectPage.jsx` (lines 36-41, 144-172, 486-572)

**Verify:** `npm test -- manual-steps` (in `tests/`) — covers admin "+ Add new step" button visibility, the create form, the disabled-until-both-filled Save button, and the auto-incremented `step_number` after creation. The "no active manual" alert path is not asserted (would need a project with no active manual).

## Validation

End-to-end verification after all tasks complete.

### Automated checks

- Server-side: `npm test -- steps` (in `server/`) — covers POST/PATCH including the auto-increment logic
- E2E: `npm test -- manual-steps` (in `tests/`) — covers admin UI visibility, the add-step flow end-to-end with Cloudinary stubbed (Save disabled until both fields filled; step_number auto-increments), the edit flow, and cancel reverting
- Full server suite: `npm test` (in `server/`); full E2E suite: `npm test` (in `tests/`)

### Manual checks (UI)

1. Log in as admin → visit `/projects/:id` of a project with an active manual → bottom section shows the timeline + green "+" button
2. Click "+" → inline form appears → leave description blank → "Save" stays disabled → fill description → upload image via Cloudinary → "Save" enables → click → new step appears at the end with `step_number = max + 1`
3. Click pencil on an existing step → description textarea + Cloudinary widget appear → modify → Save → step returns to read mode with updated content
4. Click "+" while no manual is active → alert "No active manual found. Please set a manual as active first."
5. Log out → revisit page → green "+" and pencil icons are gone; timeline still renders if there are public steps
6. With no active manual and not logged in → bottom section is hidden entirely

### Cross-feature dependencies

- `manual-management-spec.md` — at least one active manual must exist for steps to be creatable; cascade delete here is triggered there
- `project-details-spec.md` — the public timeline rendering happens through the project GET (which only includes active manual steps)
- `auth-spec.md` — admin gating on the create form and edit pencil
- Cloudinary widget (CloudinaryUpload component, see `project-crud-spec.md`) for image uploads

## Current State

Fully implemented on both client and server.

**Tests in place:**
- `server/tests/steps.test.ts` — 7 integration tests covering POST (auth, validation, auto-increment, empty-manual case) and PATCH (auth, validation, immutability of step_number/manualId)
- `tests/specs/manual-steps.spec.ts` — 4 E2E tests: admin sees Edit pencils on each step + "Add new step" button, add-step flow with Cloudinary stubbed (Save disabled-until-filled + step_number auto-increments to 13), edit-step changes description, cancel reverts

**Untested:**
- "No active manual" alert path (would need a project where no manual is `isActive` — not easy to construct from the seed without an extra setup step)
- Connecting-line render on the last step (visual, low-value)
- Invalid Cloudinary image scenarios in the step form (covered by component logic; widget is stubbed)
