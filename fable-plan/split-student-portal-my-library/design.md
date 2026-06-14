# Design — Split the student portal: My Library + catalog-only Workshops

## Context

Today `/workshops` (frontend `workshops` slice, `workshops-page.tsx`) renders both the learner's enrolled courses ("My progress") and the catalog. The backend already exposes everything we need:

- `GET /me/courses` (`enrollments` module) returns per course: summary, derived `progress` (0–100), resume target, completed lesson ids — but discards the `lesson_completions.completed_at` timestamps it already pays for in the query.
- `GET /courses` (`courses` module, `ListShownCoursesUseCase`) lists shown courses **excluding** the caller's enrolled ones (`list-shown-courses-use-case.ts:24`).
- DB: `enrollments(user_id, course_id, enrolled_at)`, `lesson_completions(enrollment_id, lesson_id, completed_at DEFAULT now(), UNIQUE(enrollment, lesson))`. Completions are append-only — no unmark exists.

Frontend facts that shape the design:

- `EnrolledCourse` (workshops domain) carries no timestamps; the page derives "Completed" from `!resumeLessonSlug`, which misclassifies a 0-lesson course (backend `progress()` returns 0 for it).
- `Enrollment.progress()` uses `Math.round`: 200/201 completions rounds 99.5 → 100, so `progress === 100` is not currently a sound completion predicate.
- Mutations (enroll, mark complete) invalidate `consumptionKeys.myCourses()` — any consumer of the my-courses data must share those keys to stay fresh.
- Hard navigation rule (frontend CLAUDE.md): every route needs a rendered affordance; `route-reachability.integration.test.tsx` scans sources for route-token references, so new route keys and their referencing UI must land together.
- `feature/tabs.tsx` is a state-driven ARIA tabs widget (same-page panels); `NavLink` with automatic `aria-current="page"` is the established pattern for route navigation (`dashboard-sidebar.tsx`).

## Goals / Non-Goals

**Goals:**

- `/my-library` as learner home: continue-learning hero, All/Completed route tabs, Udemy-style last-engagement ordering — from existing data only.
- `/workshops` reduced to the full catalog (enrolled courses included, badged), single shared course-card across both pages.
- A single, sound completion predicate (`progress === 100`) shared by backend derivation and frontend display.

**Non-Goals:**

- Unmark/uncomplete lessons; new DB schema; new course-consumption routes under `/my-library`; wishlist/archived/search; visual redesign beyond card/tabs/badge/header.

## Decisions

### D1. API exposes raw timestamps; ordering is a frontend rule

`GET /me/courses` gains `enrolledAt` (ISO string) and nullable `lastCompletedLessonAt` (= MAX(`completed_at`) per enrollment). No computed `lastEngagedAt` on the wire and no backend sorting (array order stays unspecified, documented in OpenAPI).

_Why:_ one fetch feeds three orderings (hero, All tab, Completed tab) plus the catalog badge; the orderings are UI policy and belong in pure, unit-testable frontend functions. _Alternative considered:_ serializer-computed `lastEngagedAt` + server sort — rejected: hides a UI rule in HTTP plumbing and forces re-fetches if the rule changes.

### D2. Backend threading via a repository read model, not entity growth

`Enrollment` stays a pure write model (completed lesson ids only). The port read is retyped in place:

```ts
// enrollments/domain/repositories/enrollments-repository.ts
interface EnrollmentActivity {
  readonly enrollment: Enrollment;
  readonly lastCompletedLessonAt: Maybe<Date>;
}
findByUser(userId: Id): Promise<EnrollmentActivity[]>;
```

_Why:_ `completed_at` is DB-generated (`DEFAULT now()`); entity-held timestamps would need a clock whose value the insert then discards — two sources of truth. After D3 removes the catalog's `findByUser` call, `ListMyCoursesUseCase` is the only caller, so the retype is contained. The Postgres adapter extends its existing completions select to pull `completedAt` and computes the MAX in the same grouping pass (no extra query). `InMemoryEnrollmentsRepository` stamps completion times at `save()` via an injectable clock (`now: () => Date = () => new Date()`), mirroring the DB default deterministically — no mocks.

### D3. Catalog includes enrolled courses; badge is a client-side cross-reference

`ListShownCoursesUseCase` drops the enrolled-exclusion filter **and its `EnrollmentsRepository` dependency** — it returns all shown summaries. The frontend builds `Set(myCourses.map(c => c.summary.id))` and badges matching catalog entries ("Enrolled" + CTA "Go to course"; enrolled overrides closed). Non-enrolled entries read "Course details"; closed-course treatment is preserved.

_Why:_ the workshops page already fetches both queries; an `enrolled: boolean` response field would duplicate state the client owns. While my-courses is in flight the catalog renders without badges and they pop in — acceptable.

### D4. Frontend: thin `my-library` slice reusing the workshops data path

New slice `src/my-library/` holds only what is new: pure shelf logic and the page. Its router wraps the existing `WorkshopsUseCasesProvider` and the page consumes the existing `useMyCourses` hook (and thus `consumptionKeys.myCourses()`).

_Why (load-bearing):_ enroll/mark-complete mutations invalidate that exact key; a parallel provider/key-space would silently stop the library refreshing after completing a lesson. Cross-slice reuse at equal layers is established practice (workshops already imports `courses` domain). _Alternative:_ duplicate port/use case/keys in `my-library` — rejected for the staleness trap and dead weight.

### D5. Entity facts as getters; collection policy as pure functions

`EnrolledCourse` gains **required** `enrolledAt: Date`, `lastCompletedLessonAt: Date | null` (required ⇒ the compiler finds every construction site: HTTP adapter, in-memory fake, test builders) plus getters:

- `lastEngagedAt` → `lastCompletedLessonAt ?? enrolledAt`
- `isCompleted` → `progress === 100` (single completion predicate; fixes the `!resumeLessonSlug` bug)
- `hasStarted` → `completedLessonIds.length > 0`

Collection policy lives in `my-library/domain/services/library-shelf.ts` as pure functions:

- `sortByLastEngaged(courses)` — `lastEngagedAt` DESC, tiebreak `enrolledAt` DESC, then title ASC (deterministic for same-tick fixtures)
- `completedCourses(courses)` — `isCompleted`, same ordering
- `continueLearningSelection(courses)` — ≤ 2: started-and-unfinished by `lastCompletedLessonAt` DESC, remaining slots filled with untouched courses by `enrolledAt` DESC; completed never appears

### D6. Floor the progress derivation

`Enrollment.progress()` switches `Math.round` → `Math.floor`, so 100 is reached only when every current lesson is complete. Existing spec scenarios use exact fractions (4/5 = 80, 1/2 = 50) and are unaffected; display deltas like 2/3 → 66 (was 67) are cosmetic.

### D7. Route-driven tabs as `NavLink`s, not the ARIA tabs widget

`/my-library` and `/my-library/completed` render one `MyLibraryPage` with `filter: 'all' | 'completed'`, switched by a `<nav aria-label>` of two `NavLink`s (automatic `aria-current="page"`). _Why:_ `tabs.tsx` implements roving-tabindex same-page panels; URL-backed navigation is semantically a nav. The two `NavLink`s double as the reachability affordances for both new route keys.

### D8. Shared chrome

- `CourseCard` lives in `courses/infrastructure/ui/components/` — both slices already depend on `courses` (where `CourseSummary` lives); props: `{ summary, cta, enrollment?: { progress, isCompleted }, enrolled?, enrollmentClosed? }`; always links via `buildCourseOutlinePath`.
- `LearnerHeader` in `shared/infrastructure/ui/components/`: brand, `NavLink`s My Library ↔ Workshops, email, logout, owner-only Dashboard link (`useHasRole('owner')`); adopted by both pages. Keeps `MainRoutes.Workshops` referenced after the landing swap.
- Landing swap: `/` → `MyLibrary` (top router), auth-callback success → `MyLibrary`, dashboard `ProtectedByRole` fallback → `MyLibrary`.

## Risks / Trade-offs

- [Catalog response now includes enrolled courses — clients relying on the exclusion break] → Only our frontend consumes `GET /courses`; the workshops page gains the badge in the same change. Spec scenarios "Enrolled course is absent from catalog" are replaced, not silently violated.
- [Ghost recency: MAX(`completed_at`) spans completions of since-deleted lessons, while `completedLessonIds` filters to surviving lessons] → Accepted: a course whose only completions were deleted sorts by a ghost event in the All tab but is treated as untouched by the hero. Filtering the timestamp by surviving lessons would push lesson-id joins into the repository read; not worth it.
- [Future unmark-as-deletion would regress MAX(`completed_at`)] → Out of scope; flagged so a future unmark change considers an explicit activity timestamp.
- [Reachability choreography: route keys without referencing UI break the suite mid-sequence] → Route keys and the `NavLink`s referencing them land in the same task/commit.
- [Required entity props ripple through fakes/builders/fixtures] → Intentional: compile-time discovery; the entity-growth task updates all construction sites atomically.
- [Badge pop-in while my-courses loads] → Accepted; catalog already renders independently today.
- [Floor change shifts displayed percentages by ±1] → Scan existing tests for asserted non-exact percentages during that task.

## Migration Plan

Backend changes are additive (new response fields) except the catalog exclusion removal, which ships in the same deploy as the frontend badge. No DB migration. Rollback = revert the change; no data to clean up.

## Open Questions

None — landing destination, tab semantics, hero fallback (most-recently-enrolled untouched; hidden only when no enrollments), badge/CTA wording, and route ownership were settled with the product owner during exploration.
