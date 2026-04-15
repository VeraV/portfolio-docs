# Technology Management (Admin)

## Why

When adding or editing a project, the admin may need a technology that doesn't exist in the system yet. Rather than leaving the form, the admin can create a new technology inline via a nested modal -- keeping the flow uninterrupted.

## What

Technology creation is accessible only from within the ProjectForm's TechnologySelector. There is no standalone technology management page. The admin clicks the green "+" button in the TechnologySelector header, fills out the TechnologyForm modal, and the new technology appears in the "Available" column immediately.

**User actions:**
- **Create technology:** Click "+" in TechnologySelector -> nested modal opens -> fill name, logo URL, official website, select category -> click "Create Technology" -> modal closes, technology list refreshes, new tech appears in "Available" column
- **Cancel:** Click "×" or "Cancel" -> modal closes, no changes

### API Endpoints

| Method | Endpoint | Auth | Request Body | Response | Status |
|--------|----------|------|-------------|----------|--------|
| POST | `/api/technology` | Yes | `RequestCreateTechnology` | `Technology` (with category) | 200 |
| GET | `/api/tech-category` | No | -- | `TechCategory[]` | 200 |

**Create technology request body:**
```json
{
  "name": "required, unique",
  "logo_url": "required",
  "official_site_url": "required",
  "categoryId": "required"
}
```

**Server-side validation:** Returns 400 with `"Error: some fields are missing"` if any field is missing.

**Server-side errors:**
- Create technology: `"Error creating technology"` (500)
- Get categories: `"Error getting Tech Categories from DB"` (500)

**Client-side error:** `alert()` with `"Failed to create technology: {error message}"`

**Create response:** Returns the created technology with its `category` relation included.

### TechnologyForm Modal

A nested modal (`z-[60]`, above the ProjectForm's `z-50`) with a scrollable form.

**Header:** "Add New Technology" with "×" close button.

**Form fields (top to bottom):**

1. **Technology Name** -- text input, required, placeholder: "React"
2. **Logo URL** -- url input, required, placeholder: "https://example.com/logo.png"
   - **Logo preview:** Appears below the input when a URL is entered
     - On successful load: Shows logo image (64x64px, `object-contain`) in a centered white box
     - On load error: Shows red-bordered box with `"Unable to load image. Please check the URL."`
     - Changing the URL resets the error state
3. **Official Website** -- url input, required, placeholder: "https://reactjs.org"
4. **Category** -- `<select>` dropdown populated from `GET /api/tech-category`, required. Defaults to the first category in the list.

**Form actions (bottom):**
- "Cancel" button (gray) -- closes the modal
- "Create Technology" button (blue) -- submits the form

**Form reset:** All fields reset to empty (except category which defaults to first) every time the modal opens.

### Data Flow

1. **Categories** are fetched by `ProjectForm` when it opens and passed to `TechnologyForm` via `categories` prop
2. **On submit:** `ProjectForm.handleSubmitTechnology` calls `technologyService.create(techData)`
3. **On success:** Closes TechnologyForm, re-fetches all technologies via `technologyService.getAll()`, updates `allTechnologies` state in ProjectForm
4. **Result:** The new technology appears in the TechnologySelector's "Available" column, ready to be selected

### Database Schema

**Table: `Technology`**

| Column | Type | Constraints |
|--------|------|------------|
| id | String | PK, cuid |
| name | String | UNIQUE, NOT NULL |
| logo_url | String | NOT NULL |
| official_site_url | String | NOT NULL |
| categoryId | String | FK to TechCategory, NOT NULL |
| sort_order | Int | nullable |

**Table: `TechCategory`**

| Column | Type | Constraints |
|--------|------|------------|
| id | String | PK, cuid |
| name | String | UNIQUE, NOT NULL |

Relation: Technology belongs to one TechCategory.

## Constraints

### Must

- **Auth required:** POST `/api/technology` protected by `isAuthenticated` middleware
- **Nested modal:** TechnologyForm renders at `z-[60]`, above the ProjectForm at `z-50`
- **No standalone page:** Technology creation is only accessible from within the ProjectForm flow
- **Category pre-load:** Categories are fetched when ProjectForm opens, not when TechnologyForm opens
- **Immediate availability:** After creation, the technology list refreshes so the new tech is available for selection without closing the ProjectForm
- **Logo preview:** Validates image URL by attempting to render it with `onLoad`/`onError` handlers
- **Unique name:** Enforced at database level (Prisma unique constraint)

### Out of Scope

- No technology editing (update)
- No technology deletion
- No technology reordering (sort_order not editable in UI)
- No category creation (categories are pre-seeded in the database)
- No category editing or deletion
- No bulk technology import
- No logo file upload (logo is a URL, not a Cloudinary upload)
- No technology search or filtering in the selector

## Tasks

### Server

**1. Technology Create Route**
**What:** POST `/api/technology` -- protected. Validates all four required fields. Creates technology with Prisma, returns it with `category` relation included.
**Files:**
- `server/src/routes/technology.routes.ts` (lines 26-55)

**2. TechCategory GET Route**
**What:** GET `/api/tech-category` -- public. Returns all tech categories from the database.
**Files:**
- `server/src/routes/tech-category.routes.ts`

**3. Request Type Definition**
**What:** `RequestCreateTechnology` interface with typed body: `name`, `logo_url`, `official_site_url`, `categoryId`.
**Files:**
- `server/src/types/requests.ts` (lines 38-45)

### Client

**4. Technology Service (create method)**
**What:** `create(technologyData)` -> POST `/api/technology`. Uses JWT-intercepted axios instance.
**Files:**
- `client/src/services/technology.service.js` (lines 24-27)

**5. TechCategory Service**
**What:** Singleton class with axios instance (no JWT interceptor needed). `getAll()` -> GET `/api/tech-category`.
**Files:**
- `client/src/services/tech-category.service.js`

**6. ProjectForm Integration**
**What:** Fetches tech categories on open. Manages `isTechFormOpen` state. `handleSubmitTechnology` calls `technologyService.create()`, on success closes TechnologyForm and re-fetches technology list. Passes `categories` prop to TechnologyForm.
**Files:**
- `client/src/components/ProjectForm/ProjectForm.jsx` (lines 16, 43-52, 110-138)

**7. TechnologyForm Component**
**What:** Nested modal form for creating a technology. Fields: name, logo URL (with preview and error handling), official website, category dropdown. Resets on open. On submit calls `onSubmit(formData)`.
**Files:**
- `client/src/components/TechnologyForm/TechnologyForm.jsx`

## Current State

Fully implemented on both client and server. No existing tests.
