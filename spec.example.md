# Feature Name

## Why

[1-2 sentences: What problem this solves. Why it matters now.]

## What

[Concrete deliverable. Specific enough to verify when done.]

## Constraints

### Must

- [Required patterns, libraries, conventions]

### Must Not

- [No new dependencies unless specified]
- [Don't modify unrelated code]

### Out of Scope

- [Adjacent features we're explicitly not building]

## Tasks

### T1: [Title]

**What:** [What to build]
**Files:** `path/to/file`, `path/to/test`
**Verify:** (pick one)

- `<command to run>` — when there's a dedicated test
- `No dedicated test; indirectly covered by <command>.` — exercised under a related test
- `No tests cover this task yet.` — test missing but warranted
- `No tests — type-only, intentionally not covered.` — types or trivially presentational components

### T2: [Title]

**What:** ...
**Files:** ...
**Verify:** ...

## Validation

[End-to-end verification after all tasks complete]

### Automated checks

- Full suite per layer (`npm test`, `./mvnw test`, etc.)
- Feature-specific commands listing the test files relevant to this spec

### Manual checks (UI)

1. [Happy-path flow — what the user does, what they should see]
2. [Edge cases pulled from Constraints / Out of Scope]
3. [Auth, routing, or empty-state checks where applicable]

### Cross-feature dependencies

- [Other specs whose tests or seed data this feature relies on]
- [Tests referenced by more than one spec — e.g., a guard tested in one spec but depended on by another]

## Current State

[What exists now. Saves agent from exploring.]

- Relevant files: `path/to/file.ts`
- Existing patterns to follow
