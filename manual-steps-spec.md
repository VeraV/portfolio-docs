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

**2. Step Update Route**
**What:** PATCH `/api/steps/:stepId` -- protected. Validates required fields (`description`, `image_url`). Updates only description and image_url.
**Files:**
- `server/src/routes/step.routes.ts` (lines 43-60)

**3. Request Type Definitions**
**What:** `RequestCreateManualStep` (body: `manualId`, `description`, `image_url`) and `RequestUpdateStep` (body: `description`, `image_url`).
**Files:**
- `server/src/types/requests.ts` (lines 64-77)

### Client

**4. Step Service**
**What:** Singleton class with JWT-intercepted axios instance. Methods: `create(stepData)` -> POST `/api/steps`, `update(stepId, stepData)` -> PATCH `/api/steps/:stepId`.
**Files:**
- `client/src/services/step.service.js`

**5. StepItem Component**
**What:** Renders a single step in timeline layout. Manages own `isEditing` state and `editData`. Read mode: step number circle, description (with admin edit button), image, connecting line. Edit mode: textarea, Cloudinary upload, save/cancel buttons. On save calls `stepService.update()` then `onStepUpdated()`.
**Files:**
- `client/src/components/StepItem/StepItem.jsx`

**6. ProjectPage Step Section**
**What:** Bottom section of ProjectPage. Renders `StepItem` components for each step in the active manual. Admin-only: green "+" button to open create form, inline form with description textarea, Cloudinary upload, and disabled-until-complete save button. Validates active manual exists before creating.
**Files:**
- `client/src/pages/ProjectPage/ProjectPage.jsx` (lines 36-41, 144-172, 486-572)

## Current State

Fully implemented on both client and server. No existing tests.
