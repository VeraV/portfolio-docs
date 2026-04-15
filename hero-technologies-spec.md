# Home Page -- Hero Section & Technologies

## Why

The hero section is the first thing visitors see. It introduces the portfolio owner, provides social links, and showcases the technologies she works with -- giving an immediate sense of technical breadth before visitors scroll to the projects.

## What

The top portion of the home page (`/`), rendered by the `HeroSection` component. Technologies are fetched from the API on page load and passed as a prop. All content except the technology list is hardcoded.

**User actions:**
- **View hero** with name, title, and bio
- **Click LinkedIn** -> opens LinkedIn profile in new tab
- **Click GitHub** -> opens GitHub profile in new tab
- **Browse technologies** -> logos displayed in a flex-wrap row, each linking to the technology's official site (new tab)
- **Click "Learn More About Me"** -> navigates to `/about`

### Data Flow

`HomePage` fetches technologies and projects in parallel on mount via `Promise.all([projectService.getAll(), technologyService.getAll()])`. While loading, a `Loading` spinner is shown. On error, an error message is displayed. On success, technologies are passed to `<HeroSection technologies={technologies} />`.

### API Endpoints (Public)

| Method | Endpoint | Auth | Response | Status |
|--------|----------|------|----------|--------|
| GET | `/api/technology` | No | `Technology[]` | 200 |

**Technology response shape:**
```json
{
  "id": "cuid",
  "name": "React",
  "logo_url": "https://...",
  "official_site_url": "https://...",
  "categoryId": "cuid",
  "sort_order": 1,
  "category": { "id": "cuid", "name": "Frontend" }
}
```

Technologies are returned sorted by `sort_order` ascending.

**Server-side error:** `"Error retrieving all technologies"` (500)

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

Relation: Technology belongs to one TechCategory. The GET endpoint includes the `category` relation in the response.

### Hero Section Layout

1. **Name:** "Vera Fileyeva" (h1, responsive text size: `text-5xl` / `md:text-6xl`)
2. **Title:** "Junior Web Developer" (h2, blue, responsive: `text-2xl` / `md:text-3xl`)
3. **Bio paragraph:** Hardcoded text about full stack development background
4. **Social links:** Two buttons side by side
   - "LinkedIn" -- blue button, links to `https://www.linkedin.com/in/veramei-webdeveloper/`, opens in new tab
   - "GitHub" -- dark gray button, links to `https://github.com/VeraV`, opens in new tab
5. **Technologies:** Heading "Technologies I Work With", then a flex-wrap row of technology logos
   - Each logo is 48x48px (`w-12 h-12`), wrapped in a link to `tech.official_site_url` (new tab)
   - Logo shows `tech.name` as both `title` and `alt` attributes
   - Hover effect: scale up (`hover:scale-110`)
6. **CTA button:** "Learn More About Me" -- outlined blue button, links to `/about` via `<Link>`

### Loading & Error States

- **Loading:** Full-page `Loading` spinner (bouncing dots) while `Promise.all` is pending
- **Error:** Centered red-bordered message: `"Failed to load projects or technologies. Please try again later."`
- Both states are handled in `HomePage`, not in `HeroSection`

## Constraints

### Must

- **Data fetching:** Technologies fetched alongside projects in a single `Promise.all` in `HomePage`
- **Props:** `HeroSection` receives `technologies` array as prop, does not fetch data itself
- **External links:** LinkedIn and GitHub use `<a>` with `target="_blank"` and `rel="noopener noreferrer"`
- **Technology links:** Each logo links to `official_site_url` with `target="_blank"` and `rel="noopener noreferrer"`
- **Internal link:** "Learn More About Me" uses React Router `<Link to="/about">`
- **Sort order:** Technologies displayed in `sort_order` ascending order (determined by API)

### Out of Scope

- No filtering or grouping technologies by category in the hero section (flat list)
- No technology CRUD in this section (technology creation is part of the Project Form -- see project-crud-spec)
- No search or pagination for technologies
- No animation beyond hover scale on logos

## Tasks

### Server

**1. Technology Model (Prisma)**
**What:** Technology model with unique name, logo_url, official_site_url, categoryId (FK to TechCategory), and optional sort_order. TechCategory model with unique name.
**Files:**
- `server/prisma/schema.prisma` (Technology model, lines 16-24; TechCategory model, lines 10-14)

**2. Technology GET Route**
**What:** GET `/api/technology` -- public endpoint returning all technologies with their category relation, sorted by sort_order ascending.
**Files:**
- `server/src/routes/technology.routes.ts` (lines 9-23)

### Client

**3. Technology Service**
**What:** Singleton class with axios instance and JWT interceptor. `getAll()` calls GET `/api/technology`. Also exposes `create()` (used elsewhere).
**Files:**
- `client/src/services/technology.service.js`

**4. HomePage Data Fetching**
**What:** Fetches projects and technologies in parallel via `Promise.all` on mount. Manages `isLoading` and `error` states. Passes `technologies` array as prop to `HeroSection`.
**Files:**
- `client/src/pages/HomePage/HomePage.jsx` (lines 20-37, 43-53)

**5. HeroSection Component**
**What:** Receives `technologies` prop. Renders name, title, bio, LinkedIn/GitHub links, technology logo grid (each linking to official site), and "Learn More About Me" link to `/about`.
**Files:**
- `client/src/components/HeroSection/HeroSection.jsx`

## Current State

Fully implemented on both client and server. No existing tests.
