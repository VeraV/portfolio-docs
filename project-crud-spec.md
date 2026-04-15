# Project CRUD (Admin)

## Why

The admin needs to manage portfolio projects -- create new ones, update existing ones, and remove outdated ones. All project management happens through a modal form on the home page, without navigating away.

## What

A modal-based CRUD system for projects, accessible only when logged in. The "Add New Project" button and per-card Edit/Delete buttons appear conditionally based on auth state. Create and Edit share the same `ProjectForm` modal component. Delete uses a browser `confirm()` dialog.

**User actions:**
- **Create project:** Click "+ Add New Project" button -> modal opens with empty form -> fill fields, select technologies, upload image -> click "Create Project" -> modal closes, project list refreshes
- **Edit project:** Click pencil icon on project card -> modal opens pre-filled with project data -> modify fields -> click "Update Project" -> modal closes, project list refreshes
- **Delete project:** Click X icon on project card -> browser confirm dialog `Are you sure you want to delete "{name}"?` -> confirm -> project removed, list refreshes

### API Endpoints (Admin, all require JWT)

| Method | Endpoint | Auth | Request Body | Response | Status |
|--------|----------|------|-------------|----------|--------|
| POST | `/api/projects` | Yes | `RequestCreateProject` | `Project` | 201 |
| PUT | `/api/projects/:id` | Yes | `RequestUpdateProject` | `Project` | 200 |
| DELETE | `/api/projects/:id` | Yes | -- | `Project` | 200 |

**Request body (Create & Update):**
```json
{
  "name": "required",
  "description_short": "required",
  "client_github_url": "required",
  "client_deploy_url": "required",
  "server_github_url": "optional",
  "server_deploy_url": "optional",
  "image_url": "required",
  "technologyIds": ["tech-id-1", "tech-id-2"]
}
```

**Server-side validation:** Returns 400 with `"Error: some fields are missing"` if `name`, `description_short`, `client_github_url`, `client_deploy_url`, or `image_url` are missing.

**Server-side errors:**
- Create: `"Error creating project"` (500)
- Update: `"Error updating project"` (500)
- Delete: `"Error deleting project"` (500)

**Client-side errors:** `alert()` with `"Failed to create project. Please try again."`, `"Failed to update project. Please try again."`, or `"Failed to delete project. Please try again."`

**Update strategy:** PUT replaces the entire project including tech stack -- deletes all existing `ProjectTechStack` relations (`deleteMany: {}`), then recreates them from the submitted `technologyIds`.

**Delete strategy:** Cascade delete automatically removes all `ProjectTechStack` and `ProjectManual` (and their `ManualStep`) records.

### ProjectForm Modal

A full-screen overlay modal (`z-50`) with a scrollable form (`max-h-[90vh]`). The header is sticky and shows either "Add New Project" or "Edit Project" depending on mode.

**Form fields (top to bottom):**

1. **Project Name** -- text input, required, placeholder: "My Awesome Project"
2. **Short Description** -- textarea (3 rows), required, placeholder: "A brief description of your project..."
3. **Project Image** -- Cloudinary upload widget (see below)
4. **Client GitHub URL** -- url input, required
5. **Client Deploy URL** -- url input, required
6. **Server GitHub URL** -- url input, optional (label shows "(optional)")
7. **Server Deploy URL** -- url input, optional (label shows "(optional)")
8. **Technologies** -- TechnologySelector component (see below)

**Form actions (bottom):**
- "Cancel" button (gray) -- closes the modal
- "Create Project" / "Update Project" button (blue) -- submits the form

**Data flow on open:**
- Fetches fresh `technologies` and `techCategories` lists when modal opens
- If editing: populates form with project data, extracts `technologyIds` from `project.techStack`
- If creating: resets form to empty

**Data flow on submit:**
- Calls `onSubmit(formData, project?.id)` -- parent `HomePage` determines create vs. update based on whether `projectId` is present
- On success: closes modal, calls `fetchData()` to refresh the project list

### Cloudinary Upload Widget

A reusable image upload component using the Cloudinary Upload Widget (loaded globally via `window.cloudinary`).

**Configuration:**
- Cloud name: `dojvyjghs`
- Upload preset: `portfolio_unsigned`
- Folder: `portfolio`
- Sources: local file, URL, camera
- Max file size: 5MB
- Allowed formats: JPG, JPEG, PNG, GIF, WebP
- Max dimensions: 2000x2000px

**States:**
- **No image:** Shows dashed blue border upload button with "Upload Image" text and format/size hints
- **Image uploaded:** Shows image preview (`h-48`, `object-cover`) with "Change Image" overlay on hover, and a gray "Upload Different Image" button below

**Returns:** `secure_url` from Cloudinary response to the parent via `onImageUpload` callback.

### TechnologySelector

A two-column picker for associating technologies with a project.

**Layout:** Two-column grid (stacks on mobile):
- **Left column ("Selected"):** Blue border/background, shows currently selected technology logos. Click a logo to remove it. Empty state: "Click technologies to add →"
- **Right column ("Available"):** Gray border/background, shows unselected technology logos. Click a logo to add it. Empty state: "All technologies selected!"

**Each technology** is rendered as a 48x48px button with the technology logo (36x36px). Hover effects indicate add (blue) or remove (red).

**Counts:** Each column header shows the count: "Selected (3)" / "Available (5)".

**"+" button:** Green circular button in the header opens the TechnologyForm modal to create a new technology (see technology-management-spec).

**Helper text:** "Click icons to move between sections" below the columns.

## Constraints

### Must

- **Auth required:** All CUD endpoints protected by `isAuthenticated` middleware
- **Modal pattern:** Form renders as overlay, returns `null` when `isOpen` is false
- **Shared form:** Same `ProjectForm` component for create and edit, distinguished by `project` prop (null = create)
- **Tech stack replace:** Update always deletes all existing tech stack relations and recreates them
- **Cascade delete:** Deleting a project cascades to ProjectTechStack, ProjectManual, and ManualStep
- **Cloudinary:** Image upload handled by third-party widget, not a custom file upload
- **Confirm dialog:** Delete uses native `window.confirm()`, not a custom modal
- **Refresh on mutation:** After any create/update/delete, `fetchData()` re-fetches all projects and technologies

### Out of Scope

- No client-side form validation beyond HTML `required` attributes and `type="url"`
- No drag-and-drop reordering of projects (sort_order is not editable in the UI)
- No bulk operations (delete multiple, etc.)
- No image cropping or editing
- No undo/redo for delete
- Technology creation within the form is covered in technology-management-spec

## Tasks

### Server

**1. Project Create Route**
**What:** POST `/api/projects` -- protected. Validates required fields, creates project with nested `techStack.create` for technology relations in a single Prisma operation. Returns 201.
**Files:**
- `server/src/routes/project.routes.ts` (lines 57-107)

**2. Project Update Route**
**What:** PUT `/api/projects/:id` -- protected. Validates required fields and project ID. Updates project data and replaces tech stack using `techStack.deleteMany` + `techStack.create`. Returns updated project with nested includes.
**Files:**
- `server/src/routes/project.routes.ts` (lines 109-169)

**3. Project Delete Route**
**What:** DELETE `/api/projects/:id` -- protected. Deletes project by ID, cascade removes all related records. Returns deleted project.
**Files:**
- `server/src/routes/project.routes.ts` (lines 171-183)

**4. Request Type Definitions**
**What:** `RequestCreateProject` and `RequestUpdateProject` interfaces with typed body including `technologyIds: string[]`.
**Files:**
- `server/src/types/requests.ts` (lines 12-36)

### Client

**5. Project Service (CUD methods)**
**What:** `create(projectData)` -> POST `/api/projects`, `update(id, projectData)` -> PUT `/api/projects/:id`, `delete(id)` -> DELETE `/api/projects/:id`. All use JWT-intercepted axios instance.
**Files:**
- `client/src/services/project.service.js` (lines 29-42)

**6. HomePage Admin Handlers**
**What:** `handleAddProject` (opens form with no project), `handleEditProject` (opens form with project), `handleCloseForm` (closes modal, clears editing state), `handleSubmitProject` (calls create or update based on projectId, refreshes list), `handleDeleteProject` (confirm dialog, deletes, refreshes list).
**Files:**
- `client/src/pages/HomePage/HomePage.jsx` (lines 56-114)

**7. ProjectForm Modal**
**What:** Modal form for create/edit. Fetches technologies and categories on open. Populates from `project` prop when editing, resets when creating. Fields: name, description, image (Cloudinary), GitHub URLs, deploy URLs, technology selector. Submit calls `onSubmit(formData, project?.id)`.
**Files:**
- `client/src/components/ProjectForm/ProjectForm.jsx`

**8. CloudinaryUpload Component**
**What:** Initializes Cloudinary Upload Widget once on mount with configured preset and constraints. Shows upload button and image preview. Returns `secure_url` via `onImageUpload` callback. Memoized with `React.memo`.
**Files:**
- `client/src/components/CloudinaryUpload/CloudinaryUpload.jsx`

**9. TechnologySelector Component**
**What:** Two-column grid splitting technologies into "Selected" and "Available" based on `selectedIds`. Click to move between columns. Shows counts, empty states, and a "+" button to open TechnologyForm. Calls `onChange(selectedIds)` on every change.
**Files:**
- `client/src/components/TechnologySelector/TechnologySelector.jsx`

## Current State

Fully implemented on both client and server. No existing tests.
