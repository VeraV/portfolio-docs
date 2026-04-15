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

## Current State

Fully implemented. No existing tests.
