# Planning with Fable 5 — a full session record

A complete record of the session that took "split the students portal" from a rough idea to a validated OpenSpec change, using Claude Fable 5 across four modes: **explore → decisions → plan → propose**.

- **Date**: 2026-06-13
- **Model**: Claude Fable 5 (`claude-fable-5`)
- **Branch**: `ux/my-courses`
- **Outcome**: OpenSpec change [`split-student-portal-my-library`](../split-student-portal-my-library/) — proposal, design, 3 spec deltas, 13 tasks; validated.
- **Raw transcript**: the verbatim JSONL transcript is omitted from this public repo (it contained local absolute paths); this page is the curated session record.

---

## Phase 1 — `/opsx:explore`: the brief

The session opened in explore mode with this brief (abridged to its requirements):

> Let's split the main students portal. The main view for my subscribers would be `/my-library`. The `/workshops` will exist but we're going to extract the "My courses" section from there and place it in `/my-library`. In `/workshops` we leave everything like it is — all the routes stay the same. In my-library we display all the student's courses: the last two courses he marked a lesson as completed and the course isn't yet 100% completed, then two tabs — `/my-library` (all courses the user is enrolled, sorted by the date of the last time the user marked (or unmarked) a lesson as completed) and `/my-library/completed` (all courses 100%-completed). Imitate how Udemy organizes the user's courses. `/workshops` becomes the catalog where users enroll. Both pages share the same card style. Only use the data we already have in our database — nothing new.

(The brief referenced two screenshots that never reached the model; everything was grounded in code instead.)

### What exploration found

The model mapped the codebase with one broad Explore subagent plus targeted `ctx_batch_execute` sweeps:

```
ROUTES (router.tsx, MainRoutes)
  /                        → redirect → /workshops
  /workshops               → WorkshopsPage
                              ├── "My progress" (MyCoursesSection)   ← the part to move
                              └── "Catalog" (CatalogCard grid)       ← stays
  /workshops/:courseSlug/** → outline + lesson player
  /dashboard/**            → owner admin (hidden entry)

DATA (drizzle migrations, projects/services)
  enrollments(user_id, course_id, enrolled_at)                  UNIQUE(user, course)
  lesson_completions(enrollment_id, lesson_id, completed_at)    UNIQUE(enrollment, lesson)

API GET /me/courses → summary + progress (0–100, server-computed) + resumeLessonSlug
```

### Five discoveries that reshaped the spec

1. **"Unmark" doesn't exist.** The brief said "marked **(or unmarked)** a lesson" — but completions are append-only (`completeLesson()` is idempotent, no delete path anywhere). The spec wording became "marked"; unmark was flagged as a future concern (deletion would regress the MAX timestamp).
2. **No "last activity" column — and none needed.** `last_engaged_at = MAX(completed_at) per enrollment, fallback enrolled_at` is derivable from existing data. The fallback is exactly Udemy's behavior: a freshly enrolled course tops the list.
3. **"Completed" was defined two contradictory ways.** Backend computes `progress`; the frontend derived "Completed" from `!resumeLessonSlug` — so a zero-lesson course (progress 0) displayed as Completed. The split forced one predicate: `progress === 100`.
4. **Completed isn't permanent.** Adding a lesson to a finished course drops it below 100 and migrates it between tabs. Same as Udemy; accepted as a consequence, not a bug.
5. **The navigation hard rule applies.** Every route needs a rendered affordance (`route-reachability` integration test), so route keys and the UI referencing them must land in the same commit.

### Open threads put to the user

1. Tab 1 semantics — "all courses" (Udemy-style, completed included) or in-progress only?
2. Continue-learning hero with fewer than 2 eligible courses — hide, or fall back?
3. Should catalog cards mark courses you're already enrolled in?
4. Module placement — new `my-library` vertical slice vs growing workshops?

---

## Phase 2 — the user's decisions (verbatim)

> - Tab 1 semantics: yes, Tab 1 = everything, tab 2 = completed only
> - Fallback to most-recently-enrolled untouched courses. Only hide if the user is not enrolled in any course
> - Yes, mark courses the user's enrolled in with both a badge and a "Go to course" instead of "Course details".
> - Keep my-library separated. We're just filtering courses in this page. For accessing the course we're redirected to `/workshops/:course-slug`. Don't create new path for courses based on my-library.

---

## Phase 3 — plan mode

With decisions locked, plan mode ran three sub-phases:

**Exploration gap-fill.** Two parallel Explore agents (backend my-courses flow end-to-end; frontend workshops module anatomy) plus direct queries. Notable workflow friction: both Explore agents returned meta-commentary ("I've delivered a comprehensive report…") instead of the report itself, and one `ctx_batch_execute` result blew past the inline token limit. Recovery: query the auto-indexed knowledge base (`ctx_search`) for the exact facts — entity fields, port signatures, the state-driven `tabs.tsx` props, the reachability test mechanism.

**Design.** A Plan agent produced the implementation design and caught a real landmine on its own: `Enrollment.progress()` uses `Math.round`, so 200/201 completions rounds 99.5 → 100 — `progress === 100` was not a sound completion predicate until switched to `Math.floor`.

**Verification of the plan's claims.** Spot-checks confirmed the auth-callback redirect and dashboard fallback targets. One check initially *failed wrongly*: `rg --include` (a grep flag rg doesn't have) returned "0 matches for findByUser", contradicting the Plan agent. Reading the file directly proved the agent right — `findByUser` exists at `enrollments-repository.ts:8`. Lesson: a falsifying grep with a suspect flag is not evidence.

The final plan (8 decision areas, 9 task groups, risks, verification script) was written to the plan file. Key design calls:

| Decision | Choice | Why |
|---|---|---|
| API shape | raw `enrolledAt` + nullable `lastCompletedLessonAt`; no server sort | one fetch feeds hero + both tabs + badge; ordering is UI policy |
| Backend threading | repository read model (`EnrollmentActivity`), entity stays a pure write model | `completed_at` is DB-generated; entity timestamps = two sources of truth |
| Frontend slice | thin `my-library` reusing `WorkshopsUseCasesProvider` + `consumptionKeys.myCourses()` | mutations invalidate that key; separate keys would silently stop refreshes |
| Tabs | `NavLink`s + `aria-current`, not the ARIA tabs widget | URL-backed navigation is a nav; links double as reachability affordances |
| Card | `CourseCard` in `courses/infrastructure/ui` | both slices already depend on `courses` (home of `CourseSummary`) |
| Completion | `isCompleted ⇔ progress === 100`, floored | single sound predicate across backend and frontend |

---

## Phase 4 — `ExitPlanMode` rejected → `/opsx:propose`

The user declined direct execution and redirected to the OpenSpec workflow. The propose phase generated all four artifacts (`openspec new change` → `proposal.md` → `design.md` → spec deltas → `tasks.md`).

### The discovery that corrected the plan

While reading the existing specs to write the deltas, one scenario jumped out:

> **Scenario: Enrolled course is absent from catalog** — GIVEN an authenticated user enrolled in one of two `open` courses, WHEN `GET /courses`, THEN exactly the non-enrolled course.

The backend catalog **excludes** enrolled courses (`ListShownCoursesUseCase`, `list-shown-courses-use-case.ts:24`). The plan's assumption that the enrolled badge was a pure client-side cross-reference was therefore wrong — the courses would never appear in the catalog response at all. The fix became its own task group and spec delta: the catalog stops excluding enrolled courses and drops its `EnrollmentsRepository` dependency entirely (which, as a side effect, makes the earlier "single caller of `findByUser`" claim true *after* the change).

This is the headline learning of the session: **the specs caught what code exploration missed.** Three rounds of code reading never surfaced the exclusion, because nothing in the *frontend* hinted at it — the requirement only existed server-side and in the spec.

### Artifacts produced

```
openspec/changes/split-student-portal-my-library/
├── proposal.md      why, what changes (2 BREAKING), capabilities, impact, non-goals
├── design.md        8 decisions (D1–D8), risks → mitigations, migration plan
├── specs/
│   ├── my-library/spec.md           4 ADDED requirements, 18 scenarios
│   ├── course-consumption/spec.md   3 MODIFIED (catalog inclusion, timestamps, floored progress)
│   └── workshops/spec.md            2 MODIFIED, 1 ADDED (header nav), 1 REMOVED (moved to my-library)
└── tasks.md         13 tasks / 8 groups, backend-first, e2e sweep last
```

`openspec validate split-student-portal-my-library` → **valid**.

---

## Workflow learnings

1. **Explore mode before plan mode pays off twice.** The explore phase surfaced the questions worth asking (tab semantics, hero fallback, badge, module placement) *before* any design work, so plan mode started with zero ambiguity and the user answered everything in one message.
2. **Subagent final messages are lossy.** Twice an Explore agent did the work but returned "I delivered a report" instead of the report. Mitigation that worked: auto-indexed tool output + `ctx_search` recovery, and asking agents for *exact field lists and verbatim quotes* rather than summaries.
3. **Verify the verifier.** A wrong rg flag produced a false "claim refuted". When a spot-check contradicts a thorough agent, suspect the spot-check first — read the file.
4. **Specs are a second source of truth that code reading doesn't replace.** The catalog-exclusion requirement was invisible from the frontend and three exploration passes. Reading the existing specs while writing deltas is what caught it.
5. **Small spec bugs hide in derivations.** `Math.round` (99.5 → 100) and `!resumeLessonSlug` (0-lesson course = "Completed") both broke the completion predicate the new feature depended on. Defining the predicate once (`progress === 100`, floored) fixed feature and bugs together.
6. **Future-proofing notes belong in the change, not the code.** "Unmark would regress MAX(completed_at)" is recorded in design.md risks, where the future change will find it.

## Next step

Run `/opsx:apply` to implement — tasks execute under the project's XP TDD protocol (RED → GREEN → COMMIT → REFACTOR → COMMIT, in-memory fakes, commit per task, tick `tasks.md`).
