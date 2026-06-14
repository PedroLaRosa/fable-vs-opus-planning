# Design — Add My Library

## Context

The learner portal is a single page. `/workshops` (`projects/frontend/src/workshops/infrastructure/ui/views/workshops-page.tsx`) renders both a "My courses" progress rail (`MyCoursesSection` over `useMyCourses`) and the enroll catalog (`CatalogList` over `useCatalog`). `/` redirects to `/workshops` (`shared/infrastructure/ui/views/router.tsx`) and the magic-link callback lands there too. The enrolled-courses data comes from `GET /me/courses`, served by the backend `enrollments` module and consumed via the `workshops` frontend slice's `CourseConsumptionGateway`.

The data needed to order courses by recency already exists in Postgres but is dropped on the read path: `lesson_completions.completed_at` (`timestamptz DEFAULT now() NOT NULL`) and `enrollments.enrolled_at` exist, but `PostgresEnrollmentsRepository.findCompletedLessons` selects only `{ enrollment_id, lesson_id }`, `Enrollment` (`domain/entities/enrollment.ts`) holds only `completedLessonIds: Id[]`, and `serializeEnrolledCourse` emits no timestamp. So the frontend `EnrolledCourse` has `progress`, `resumeLessonSlug`, and `completedLessonIds` — no way to know _when_ anything happened.

Both projects follow hexagonal architecture with vertical slicing. The frontend navigation rule (`projects/frontend/CLAUDE.md`) requires every route to be reachable from a rendered affordance, enforced by `shared/infrastructure/ui/tests/route-reachability.integration.test.tsx`.

## Goals / Non-Goals

**Goals:**

- A dedicated subscriber home at `/my-library`: a Continue-learning rail (two most-recently-active in-progress courses) plus an enrolled tab and a completed tab.
- Order enrolled courses by _most recent lesson completion_, using only data already in the database (no migration, no new tracking).
- `/workshops` becomes the enroll-only catalog; every existing `/workshops*` route and the player/outline are untouched.
- One shared course card across both pages; consistent account/nav chrome via a shared learner shell.

**Non-Goals:**

- Un-marking a lesson, a `last_activity_at` column, or any new table/column — see proposal Non-goals. Ordering is "most recent completion," derived from `completed_at`.
- "Last accessed/opened" tracking; resume target stays "first uncompleted lesson."
- Backend ordering of `GET /me/courses` — the list stays a set; ordering/grouping is a frontend concern (below).

## Decisions

### 1. Surface `lastCompletedAt` (and `enrolledAt`) from existing columns — no migration

The `enrollments` read path is widened to carry timestamps that already exist:

- `findCompletedLessons` additionally selects `completed_at`; per enrollment it computes `lastCompletedAt = max(completed_at)` (or none when the enrollment has no completions).
- `Enrollment.restore` accepts `lastCompletedAt: Maybe<Date>`; the aggregate exposes it. `enrolledAt` is already on the aggregate.
- `ListMyCoursesUseCase.EnrolledCourse` gains `lastCompletedAt: Maybe<Date>`; `serializeEnrolledCourse` emits `lastCompletedAt` (ISO string or `null`) and `enrolledAt` (ISO string).
- Frontend `EnrolledCoursePayload` / `toEnrolledCourse` / `EnrolledCourse` carry `lastCompletedAt: string | null` and `enrolledAt: string`.

**Why compute the max during reconstitution** (in the repository / use case) rather than store per-completion timestamps on the aggregate: the domain never modelled completion times, and the only consumer is an ordering key. Passing a single restored `lastCompletedAt` keeps `completedLessonIds` and the `completeLesson`/`progress` behaviour untouched.
**Alternative considered:** carry a `Map<lessonId, completedAt>` on `Enrollment`. Rejected — it reshapes the aggregate and every test builder for one derived scalar.

**`lastCompletedAt` spans all completion rows of the enrollment**, including completions of lessons later removed from the course (whereas `progress`/`completedLessonIds` count only _current_ lessons). The timestamp answers "when did the learner last mark a lesson complete," which is true regardless of later authoring edits. The divergence is real but tiny and documented under Risks.

### 2. Ordering and grouping live on the frontend, as a pure function

`GET /me/courses` stays an unordered set; all view rules are a pure, unit-tested function over the `EnrolledCourse[]` (no clock, no fetch). Timestamps are compared as **ISO-8601 strings lexicographically** (ISO-8601 sorts chronologically as text) — no `Date` parsing, no timezone hazard. Ties break by course title then id for determinism.

The three derived views:

```
Continue-learning rail  =  progress < 100  AND  lastCompletedAt != null
                           → sort by lastCompletedAt desc → take 2          (0–2 cards)

"My Library" tab        =  ALL enrolled courses
                           group A: lastCompletedAt != null → lastCompletedAt desc
                           group B: lastCompletedAt == null → enrolledAt   desc
                           render A then B                                  (completed courses appear here too)

"Completed" tab         =  progress === 100   (⇔ resumeLessonSlug == null) → lastCompletedAt desc
```

`progress < 100 AND lastCompletedAt != null` is the literal reading of "courses he marked a lesson as completed and the course isn't yet 100% completed." A course whose only completions are for since-removed lessons has `progress === 0` but `lastCompletedAt != null`; it may surface in the rail — acceptable (the learner genuinely worked on it).

### 3. `/my-library` reuses the `workshops` slice; no new gateway

The new views live in the `workshops` frontend slice and reuse `WorkshopsUseCasesProvider`, the existing `useMyCourses` hook, and the `CourseConsumptionGateway` — the data is already there client-side. A new shared route module `shared/infrastructure/ui/views/my-library/router.tsx` mounts `/my-library/*` behind `ProtectedRoute` + `WorkshopsUseCasesProvider` (mirroring `views/workshops/router.tsx`) and lazy-loads the library page.
**Alternative considered:** a brand-new `my-library` frontend module with its own gateway/adapter. Rejected — it would duplicate the consumption gateway for the exact same `/me/courses` read. (The OpenSpec _capability_ `my-library` is still its own spec; capability ≠ code module.)

### 4. Route-based tabs, not the in-page `Tabs` widget

The two tabs are distinct URLs (`/my-library`, `/my-library/completed`), so they are `NavLink`s styled as a tablist (active via `aria-current`), each a real anchor — satisfying the reachability rule and giving each tab a shareable URL. The existing `Tabs` feature component swaps panels in place without routing and is **not** reused here.

### 5. One shared `CourseCard` (presentational, variant-driven)

A `CourseCard` in `shared/infrastructure/ui` renders thumbnail · `Course` tag · title · description, then a footer that varies by mode:

- **library / in progress** → progress bar + `Resume` (→ resume lesson at `/workshops/:courseSlug/lessons/:lessonSlug`).
- **library / completed** → full progress + `✓ Completed` + `Revisit` (→ course outline `/workshops/:courseSlug`), since `resumeLessonSlug` is `null` at 100%.
- **catalog** → `Open course` (open enrollment) or `Enrollment closed`, navigating to the outline `/workshops/:courseSlug` (unchanged catalog behaviour).

It is pure presentation driven by props (no fetching), consuming only existing design tokens (no new tokens). `workshops-page.tsx`'s `CatalogCard` and the deleted `EnrolledCourseRow` collapse into it.

### 6. Shared learner shell carries the chrome and primary nav

The account chrome currently inline in `workshops-page.tsx` (email, logout, owner-only Dashboard link) plus primary nav links **My Library** and **Browse workshops** move into a shared `LearnerShell` rendered by both pages. On `/workshops` the observable chrome (logout, owner Dashboard link) is unchanged — those existing workshops requirements still hold; the shell is a refactor. `/my-library` specifies its own equivalent chrome.

### 7. Home and post-login redirect move to `/my-library`

`/` redirects to `/my-library` and the post-login destination becomes `/my-library`. `/workshops` keeps all routes and is reached via the shell's **Browse workshops** link and the library's empty state. A learner with zero enrollments lands on `/my-library` and sees an empty state whose primary CTA is **Browse workshops**.

### 8. Testing

- **Backend:** unit (`Enrollment` exposes `lastCompletedAt`; `ListMyCoursesUseCase` maps it; serializer emits ISO/`null` + `enrolledAt`), integration on pg-mem (`PostgresEnrollmentsRepository` selects `completed_at` and returns the per-enrollment max), and the OpenAPI test for the new nullable field. No migration test — the column exists.
- **Frontend:** unit tests for the ordering/grouping pure function (group A/B split, rail take-2, completed filter, tie-break, all-`null` and empty cases); component tests for the library page states (rail present/absent, both tabs, loading, error, empty-with-Browse-CTA); the updated `workshops-page` test (catalog only, no rail); the reachability integration test gains the new routes and affordances; a Playwright `learner-library-journey` e2e (enroll → complete a lesson → course tops the rail and the library tab → complete all → it leaves the rail and appears under Completed). The `InMemoryCourseConsumptionRepository` fake and `EnrolledCourse` entity gain the new fields; a small test factory keeps builders terse.

## Risks / Trade-offs

- **Existing tests assume the old portal shape** (workshops-page renders the My courses rail; `/` and post-login land on `/workshops`). → Enumerate and update: `workshops-page.test.tsx`, `route-reachability.integration.test.tsx`, and any auth-callback/redirect test asserting `/workshops` as the landing.
- **`EnrolledCourse` entity + `feedMyCourse` fake gain fields**, rippling to every builder that constructs an enrolled course. → `lastCompletedAt` is nullable so existing rows pass `null`; add a test factory to supply `enrolledAt` once.
- **Completed courses appear in both the My Library tab and the Completed tab** (Udemy "All courses" parity). → Intentional and specced; flagged as an open question to confirm.
- **`lastCompletedAt` (all completions) vs `progress` (current lessons) can diverge** for courses edited after completion. → Documented; affects only ordering, never correctness of progress.
- **Ordering ties** on identical `lastCompletedAt`. → Deterministic secondary sort by title then id.
- **Two timestamps cross the API as strings.** → Lexicographic ISO-8601 compare is correct and avoids `Date`/tz bugs; `null` `lastCompletedAt` is explicit, not `undefined`.

## Migration Plan

1. **No database migration** — `lesson_completions.completed_at` and `enrollments.enrolled_at` already exist.
2. **Backend deploys first:** additive nullable `lastCompletedAt` + `enrolledAt` on the `/me/courses` payload; older frontend ignores unknown fields.
3. **Frontend deploys after:** new `/my-library` routes, the `/` + post-login redirect, the shared shell/card, and `/workshops` reduced to catalog-only.
4. **Rollback:** revert the frontend (routes/redirect/page); the extra payload fields are harmless to leave in place.

## Open Questions

- **My Library tab scope:** does the main tab list _all_ enrolled courses including completed ones (current spec, matches "all courses the user is enrolled" and Udemy "All courses"), or only in-progress? Default: include completed.
- **Completed-card CTA:** label `Revisit` → course outline. Confirm wording/target.
- None blocking implementation.
