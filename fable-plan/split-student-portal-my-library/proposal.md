# Split the student portal: My Library + catalog-only Workshops

## Why

The `/workshops` page currently mixes two jobs: tracking the learner's own courses ("My progress") and browsing the catalog. As the catalog grows, learners need a dedicated home for their enrolled courses — with recency ordering and a completed view, imitating Udemy's "My learning" — while `/workshops` becomes purely the place to discover and enroll.

## What Changes

- New `/my-library` route becomes the learner home: a "Continue learning" hero (up to 2 most recently progressed, unfinished courses, filled with most recently enrolled untouched courses) above route-driven tabs — **All courses** (`/my-library`, every enrollment sorted by last engagement) and **Completed** (`/my-library/completed`, only 100%-completed courses).
- `/` and the post-login redirect change from `/workshops` to `/my-library`. **BREAKING** (default landing moves).
- `/workshops` keeps only the catalog; the "My progress" section is removed.
- The backend catalog (`GET /courses`) stops excluding courses the caller is enrolled in; the frontend marks those entries with an "Enrolled" badge and a "Go to course" CTA (non-enrolled entries read "Course details"). **BREAKING** (catalog response now includes enrolled courses).
- `GET /me/courses` adds two derived fields per course: `enrolledAt` and nullable `lastCompletedLessonAt` (= latest `lesson_completions.completed_at`). No schema changes — existing columns only.
- One shared course-card component (thumbnail, title, description, progress bar + status when enrolled) is used by both pages; cards always link to the existing `/workshops/:courseSlug` routes — no new course-consumption routes.
- Progress derivation is floored so `progress === 100` holds only when every current lesson is complete (fixes 99.5% rounding to 100 and the frontend deriving "Completed" from a missing resume target).
- A shared learner header with My Library ↔ Workshops navigation replaces the page-local workshops header chrome.

## Capabilities

### New Capabilities

- `my-library`: the learner's home at `/my-library` — continue-learning hero, All/Completed route tabs, last-engagement sorting, and navigation affordances to course outlines and the catalog.

### Modified Capabilities

- `course-consumption`: catalog no longer excludes enrolled courses; `GET /me/courses` entries gain `enrolledAt` and `lastCompletedLessonAt`; derived `progress` reaches 100 only when all current lessons are complete (floor, not round).
- `workshops`: no longer the default post-login destination; the enrolled-courses-with-progress requirement moves out (to `my-library`); catalog entries reflect enrollment (badge + "Go to course" vs "Course details"); header gains shared learner navigation.

## Impact

- **services**: `enrollments` module (repository port/read model, list-my-courses use case, serializer, OpenAPI), `courses` module (`ListShownCoursesUseCase` drops its enrollments filter and dependency), `Enrollment.progress()` rounding.
- **frontend**: new `my-library` slice; `workshops` slice (entity growth, HTTP adapter payload, page reduced to catalog); `courses` slice (shared `CourseCard` UI component); `shared` (route registry + top router + auth-callback/dashboard redirects, shared `LearnerHeader`). Route-reachability and login-redirect integration tests are sensitive to the route/redirect changes.
- **OpenAPI surface**: `GET /courses` semantics and `GET /me/courses` schema change (additive fields, required in response).
- **Design tokens**: card/tab/badge styling consumes existing `tokens.css` variables only; no `DESIGN.md` changes expected.
- No new workspace dependencies; no database migrations.

## Non-goals

- No "unmark lesson complete" feature (completions stay append-only); the sort reacts to marking only.
- No new course-consumption routes under `/my-library` — outlines and lessons stay at `/workshops/:courseSlug/**`.
- No new database tables or columns; all new API fields are derived from existing data.
- No wishlist/archived/lists tabs, search, or catalog filtering.
- No visual redesign beyond the unified card, tabs, badge, and shared header.
