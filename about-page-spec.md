# About Page

## Why

The about page gives visitors context about who the portfolio owner is -- background, skills, and how to get in touch. It builds trust and personality beyond just listing projects.

## What

A static, public page at `/about` with no API calls or dynamic data. All content is hardcoded in the component. No admin editing capability.

**Page sections (top to bottom):**

1. **Title:** "About Me" (centered, h1)
2. **Profile photo:** Circular image of Vera, hosted on Cloudinary
3. **Introduction:** "Hello! I'm Vera" heading with a short bio paragraph about being a Junior Full Stack Developer
4. **My Journey:** Three paragraphs covering IT degree, six years in software companies in Belarus (research, startup, enterprise), Business Analysis course, and Ironhack bootcamp (400+ hours, MERN stack)
5. **Skills & Expertise:** Bulleted list:
   - Frontend Development: React, HTML5, CSS3, Bootstrap, Tailwind CSS
   - Backend Development: Node.js, TypeScript, Express, RESTful APIs
   - Database Management: PostgreSQL, MongoDB, Prisma ORM
   - Deployment & DevOps: Docker, Render, Netlify/Vercel
   - Version Control: Git & GitHub
   - Agile methodologies and collaborative development
6. **What Drives Me:** Two paragraphs about user-focused development, continuous learning, yoga teaching, and singing in a choir in Lisbon
7. **Call to Action:** Separated by a top border line
   - Text: "Currently seeking Junior Full Stack Developer opportunities in Lisbon or remote."
   - Text: "Feel free to check what I've built or contact me directly"
   - "View My Work" button -- links to `/` (home page)
   - "Email Me" button -- `mailto:veremei.vera@gmail.com`

### Layout & Styling

- Gray background (`bg-gray-50`), white card container with shadow, max-width `max-w-3xl`, centered
- Profile photo: 192x192px (`w-48 h-48`), circular (`rounded-full`), with shadow
- Section headings: `text-2xl font-semibold`
- Body text: justified alignment, relaxed line height
- Skills list: disc markers, inside positioning
- CTA buttons: "View My Work" is filled blue, "Email Me" is outlined blue (inverts on hover)

## Constraints

### Must

- **Purely static:** No API calls, no state, no context consumption
- **Navigation:** Uses React Router `<Link>` for "View My Work" button (client-side navigation to `/`)
- **External link:** "Email Me" uses a standard `<a href="mailto:...">` tag
- **Image:** Profile photo served from Cloudinary CDN (hardcoded URL)

### Must Not

- Do not introduce any API calls, state, or context consumption from this page (it must stay purely presentational)
- Do not add admin editing UI for the page content (all copy is hardcoded by design)
- Do not migrate content to a CMS or markdown file in this iteration; if needed later it becomes its own spec
- Do not add a contact form here (covered by the `mailto:` button only)

### Out of Scope

- No admin editing of about page content (all hardcoded)
- No social media links on this page (those are in the HeroSection on the home page)
- No contact form
- No downloadable resume/CV

## Tasks

### Client

**1. ProfilePage Component**
**What:** Static page component rendering all sections -- title, profile photo, introduction, journey, skills list, passions, and call-to-action with "View My Work" (`<Link to="/">`) and "Email Me" (`mailto:`) buttons. No props, no state, no API calls.
**Files:**
- `client/src/pages/ProfilePage/ProfilePage.jsx`

**Verify:** No tests cover this task yet.

## Validation

End-to-end verification after all tasks complete.

### Automated checks

- No automated tests cover this spec yet (no server endpoints, no E2E spec written)

### Manual checks (UI)

1. Visit `/about` → page renders with white card on gray background
2. Profile photo loads (Cloudinary URL); broken image would be visible
3. All seven content sections appear in order: title, photo, intro, journey, skills, what-drives-me, CTA
4. Click "View My Work" → client-side navigation to `/` (no full page reload)
5. Click "Email Me" → mail client opens with `veremei.vera@gmail.com` prefilled
6. Page works for both logged-in and logged-out users (no auth-aware rendering)

### Cross-feature dependencies

- `navigation-spec.md` — the `/about` route must be registered for this page to be reachable
- `auth-spec.md` — Navbar (which is shared layout) is auth-aware; this page does not consume auth itself but renders alongside it

## Current State

Fully implemented.

**Tests in place:**
- None yet

**Untested:**
- The entire ProfilePage component (no tests written; static content limits test ROI)
