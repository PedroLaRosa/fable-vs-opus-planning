# Add My Library

## Why

The learner portal does two jobs in one page: `/workshops` shows both the enrolled "My courses" progress rail and the enroll catalog. As a subscriber accumulates courses there is no dedicated home to resume learning, no ordering by recent activity, and no separation of finished courses. Split the portal — `/my-library` becomes the subscriber home for continuing and organizing enrolled courses (Udemy-style), while `/workshops` keeps every route exactly as-is but becomes purely the enroll catalog.

## What Changes

- New `/my-library` route, the subscriber home: a **Continue learning** rail of the two most-recently-active in-progress courses, plus two tabs — `/my-library` (all enrolled courses, ordered by most recent lesson completion) and `/my-library/completed` (100%-complete courses).
- `/` now redirects to `/my-library` instead of `/workshops`.
- `/workshops` keeps all routes (`/workshops`, `/workshops/:courseSlug`, `/workshops/:courseSlug/lessons/:lessonSlug`); only the index page changes — the "My courses" progress rail is removed and it becomes the enroll-only catalog.
- Both pages reuse one shared course card: thumbnail · tag · title · description, then a progress bar + **Resume** (library, in progress), a **Revisit** affordance (library, completed), or an **Enroll/Open** affordance (catalog).
- The `GET /me/courses` payload gains a nullable `lastCompletedAt` — the per-enrollment max of `lesson_completions.completed_at`, surfaced from **existing** data with no schema change and no new tracking.
- Ordering: courses with at least one completion sort by `lastCompletedAt` descending; courses with no completions fall back to `enrolled_at` descending and sort below them. **Continue learning** = enrolled, in progress (`progress < 100`), with at least one completion — newest completion first, take two.

## Capabilities

### New Capabilities

- `my-library`: the subscriber home — the Continue-learning rail, the enrolled and completed tabs, the ordering/grouping rules, the empty states, and the shared course-card presentation on the library side.

### Modified Capabilities

- `course-consumption`: the "my courses" read (`GET /me/courses`) payload includes a nullable `lastCompletedAt` (the max `completed_at` across the enrollment's completions; `null` when none), surfaced from the existing `lesson_completions.completed_at` column.
- `workshops`: the `/workshops` index becomes the enroll-only catalog — the "My courses" progress rail is removed from it — and it adopts the shared course card; the home/post-login redirect target moves from `/workshops` to `/my-library`. The course outline and lesson player routes are unchanged.

## Impact

- **projects/services — `enrollments` module** (serves `course-consumption`): `infrastructure/adapters/postgres-enrollments-repository.ts` (`findCompletedLessons` also selects `completed_at`; carry the timestamps), `domain/entities/enrollment.ts` (expose a `lastCompletedAt`), `application/list-my-courses-use-case.ts` (`EnrolledCourse` gains `lastCompletedAt`), `infrastructure/http/serializers.ts`, and `infrastructure/http/enrollments.openapi.ts` (my-courses schema). **No migration** — `lesson_completions.completed_at` (`timestamptz DEFAULT now()`) already exists.
- **projects/frontend — `workshops` module**: `infrastructure/adapters/http-course-consumption-repository.ts` (`EnrolledCoursePayload` + `toEnrolledCourse` carry `lastCompletedAt`), `domain/entities/enrolled-course.ts` (+ `lastCompletedAt`), `infrastructure/ui/views/workshops-page.tsx` (drop the "My courses" section; catalog-only).
- **projects/frontend — new `my-library` views**: the page, the two tab routes, the Continue-learning rail, and the ordering/grouping logic. Reuses the existing `WorkshopsUseCasesProvider`, `useMyCourses` hook, and `CourseConsumptionGateway` (the data already exists client-side) — the new views attach to the `workshops` slice rather than duplicating its gateway; final placement decided in design.md.
- **projects/frontend — shared UI / routing**: a shared `CourseCard` consumed by both catalog and library; `shared/infrastructure/ui/routes.ts` (`MyLibrary`, `MyLibraryCompleted`), `views/router.tsx` (mount `/my-library/*`, redirect `/` → `/my-library`), new `views/my-library/router.tsx`. The account/nav chrome in `workshops-page.tsx` is lifted into a shared learner shell so both pages carry it.
- **Cross-cutting**: the route-reachability integration test (`shared/infrastructure/ui/tests/route-reachability.integration.test.tsx`) gains the new routes and their affordances (the two tabs reach each other; the library empty state links to `/workshops`). No new design tokens; no workspace dependency changes; the OpenAPI surface gains one nullable field.

## Non-goals

- **Un-marking / un-completing a lesson.** It does not exist today (mark-complete is append-only and idempotent) and is not added here — no toggle, no DELETE endpoint, no `last_activity_at` column. The spec's "(or unmarked)" is therefore out of scope; ordering is purely "most recent completion."
- **No new database tables, columns, or tracking.** `lastCompletedAt` derives entirely from the existing `lesson_completions.completed_at`.
- **No "last accessed/opened" tracking.** Recency means recently _completed_, not recently _viewed_; the resume target stays "first uncompleted lesson."
- **No Udemy parities beyond the two tabs** — no wishlist, archived, custom lists, ratings, search/filter, or certificates.
- **`/workshops` routing, the course outline, and the lesson player are unchanged.**
