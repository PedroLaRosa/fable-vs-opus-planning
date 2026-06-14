# Tasks — Split the student portal: My Library + catalog-only Workshops

## 1. Backend — enrollment activity timestamps

- [ ] 1.1 Enrollments repository read returns each enrollment together with its latest lesson-completion timestamp (in-memory fake stamps completion times deterministically)
- [ ] 1.2 Postgres enrollments repository derives the latest completion timestamp per enrollment from existing completion rows
- [ ] 1.3 My-courses listing exposes the enrollment date and the latest completion timestamp for each course
- [ ] 1.4 My-courses HTTP response and OpenAPI schema include the enrollment date and nullable latest-completion timestamp
- [ ] 1.5 Derived progress reaches 100 only when every current lesson is complete (rounded down)

## 2. Backend — catalog includes enrolled courses

- [ ] 2.1 Catalog listing no longer excludes the caller's enrolled courses and drops its enrollments dependency

## 3. Frontend — enrolled-course domain

- [ ] 3.1 Enrolled course carries enrollment and latest-completion timestamps with derived last-engagement, completion, and started facts, propagated through every construction site (HTTP adapter, in-memory fake, test builders)
- [ ] 3.2 Library shelf policy: last-engagement ordering with deterministic tiebreaks, completed filter, and continue-learning selection with untouched-course fallback

## 4. Frontend — shared UI

- [ ] 4.1 Shared course card renders thumbnail, title, description, optional progress with completion status, enrolled badge, configurable affordance label, closed-enrollment state, and links to the course outline
- [ ] 4.2 Shared learner header provides My Library ↔ Workshops navigation with active-page marking plus the existing account controls (email, logout, owner-only Dashboard link)

## 5. Frontend — My Library

- [ ] 5.1 My Library routes and page: continue-learning hero and route-driven All/Completed tabs with loading, error, and empty states (routes and their referencing affordances land together)
- [ ] 5.2 My Library is mounted as a protected route area reusing the existing course-consumption provider and query keys so mutations refresh the library

## 6. Frontend — landing swap

- [ ] 6.1 Root, post-login, and role-fallback redirects land on My Library

## 7. Frontend — Workshops becomes catalog-only

- [ ] 7.1 Workshops page drops the enrolled-courses section and renders the full catalog with shared cards, enrolled badges, and Go-to-course / Course-details affordances

## 8. End-to-end coverage

- [ ] 8.1 Review the spec deltas and write every representable e2e journey (library hero, tab filtering and re-sorting after completion, catalog badge and enrollment flow, landing redirects), updating the existing learner journey where it touched the removed section
