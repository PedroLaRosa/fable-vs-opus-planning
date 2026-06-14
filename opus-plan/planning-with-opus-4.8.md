# Planning with Opus 4.8 — splitting the students portal into `/my-library`

A transcript of an OpenSpec `explore → propose` planning session (model: Claude Opus 4.8, 1M context).
The work product is the change `openspec/changes/add-my-library/`. This doc captures how we got there:
the exploration, the data-model discovery that reshaped the plan, the decisions, and the artifacts produced.

---

## 1. The request (`/opsx:explore`)

> Let's split the main students portal. The main view for my subscribers would be `/my-library`. The
> `/workshops` will exist but we're going to extract the "My courses" section from there and place it in
> `/my-library`. In `/workshops` we leave everything like it is — all the routes stay the same. In my-library
> we display all the student's courses: the last two courses he marked a lesson as completed and the course
> isn't yet 100% completed, then two tabs — `/my-library` (all courses the user is enrolled, sorted by the
> date of the last time the user marked (or unmarked) a lesson as completed) and `/my-library/completed` (all
> courses 100%-completed). Imitate how Udemy organizes the user's courses. `/workshops` becomes the catalog
> where users enroll. Both pages share the same card style. Only use the data we already have in our
> database — nothing new.

---

## 2. Exploration — grounding in the codebase

Investigated the frontend `workshops` slice, the backend `enrollments` module, the Drizzle schema, and the
React Router wiring. Key files read:

- `projects/frontend/src/workshops/domain/entities/enrolled-course.ts` — `{ summary, progress, resumeLessonId, resumeLessonSlug, completedLessonIds }`. **No timestamp.**
- `projects/frontend/src/workshops/infrastructure/ui/views/workshops-page.tsx` — renders both the "My courses" rail (`MyCoursesSection`) and the catalog (`CatalogList`).
- `projects/frontend/src/workshops/infrastructure/adapters/http-course-consumption-repository.ts` — `GET /me/courses` payload has no timestamp, no inherent ordering.
- `projects/services/drizzle/migrations/0012_add_lesson_completions.sql` — `lesson_completions.completed_at timestamptz DEFAULT now() NOT NULL`. **It exists.**
- `projects/services/drizzle/migrations/0011_add_enrollments.sql` — `enrollments.enrolled_at timestamptz DEFAULT now() NOT NULL`.
- `projects/services/src/enrollments/infrastructure/adapters/postgres-enrollments-repository.ts` — `findCompletedLessons` selects only `{ enrollment_id, lesson_id }` — **drops `completed_at`**.
- `projects/services/src/enrollments/domain/entities/enrollment.ts` — `completeLesson` is **append-only and idempotent**; there is no un-mark.
- `projects/frontend/src/shared/infrastructure/ui/{routes.ts,app.tsx,views/router.tsx,views/workshops/router.tsx}` — routing; `/` → `/workshops`; `/workshops/*` mounted behind `ProtectedRoute` + `WorkshopsUseCasesProvider`.
- `projects/frontend/CLAUDE.md` — hard navigation rule: every route must be reachable from a rendered affordance (enforced by a reachability integration test).

### The decisive finding

The timestamps the feature needs **already exist in Postgres** but are **thrown away on the read path** —
one column dropped at step 2 of a 6-layer chain:

```
lesson_completions.completed_at        ✅ EXISTS (timestamptz, default now())
        │
        ▼  ✗ DROPPED HERE
findCompletedLessons()  → selects only { enrollmentId, lessonId }   (add completed_at)
        ▼
Enrollment (domain)     → holds Id[] only, no timestamps            (derive lastCompletedAt)
        ▼
ListMyCoursesUseCase    → EnrolledCourse { progress, resume…, completedLessonIds }   (add lastCompletedAt)
        ▼
serializeEnrolledCourse → JSON                                      (add lastCompletedAt)
        ▼
GET /me/courses  EnrolledCoursePayload                              (add lastCompletedAt)
        ▼
frontend EnrolledCourse entity                                     (add lastCompletedAt)
        ▼
/my-library  → sort + "Continue learning" rail
```

So "use only data we already have — nothing new" holds **if** "nothing new" means no new tables/columns/tracking.
We do need to **plumb an existing column through 6 layers**. No migration. That's the backbone of the work.

### The page map

```
BEFORE                              AFTER
─────────────────────────           ─────────────────────────────────────────────
/  → redirects to /workshops        /  → redirects to /my-library   (subscriber home)

/workshops                          /my-library                      ← NEW
 ├─ "My courses" rail                ├─ "Continue learning": last 2 (≥1 completion, <100%)
 └─ Catalog (enroll)                 ├─ Tab ·  My Library   → all enrolled, by last completion
                                     └─ Tab ·  Completed    → /my-library/completed (100%)
/workshops/:slug …  (outline/player)
                                    /workshops                        ← pure catalog (enroll)
                                    /workshops/:slug …  UNCHANGED (all routes stay)
```

### The card

Today the shapes differ: catalog is a `.card` (thumb + tag + title + desc); "my courses" is a flat
`.enrolledCourseRow` (progress bar + Continue). Unifying them is the visual half of the task.

### The real tension surfaced

The spec said "the last time the user marked **(or unmarked)** a lesson" — but **un-marking doesn't exist
today.** `markLessonComplete` is append-only and idempotent; there's no toggle, no DELETE endpoint, and
`completed_at` lives on the row, so once a row is deleted there's no surviving timestamp to sort by. That
forks the design — hence the questions below.

---

## 3. Decisions (asked & answered)

| Question                                      | Options                                                                                     | **Chosen**                                                                        |
| --------------------------------------------- | ------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **Sort key** (un-marking isn't a feature)     | most-recent-completion · add un-mark + activity column · un-mark but keep completed_at sort | **Most recent completion** — `MAX(completed_at)`; "(or unmarked)" out of scope    |
| **Never-started courses** (no `completed_at`) | fall back to `enrolled_at` · blend into one sort · hide until started                       | **Fall back to `enrolled_at`**, listed below courses with activity                |
| **Card content**                              | minimal (thumb + progress/CTA) · rich (+ tag + description)                                 | **Rich** — thumbnail · tag · title · description, then progress + Resume / Enroll |

Selected card preview:

```
LIBRARY                CATALOG
┌────────────┐        ┌────────────┐
│  [ thumb ] │        │  [ thumb ] │
│ Course•tag │        │ Course•tag │
│ Title      │        │ Title      │
│ a short    │        │ a short    │
│ blurb …    │        │ blurb …    │
│ ▓▓▓▓░░ 60% │        │ Enroll →   │
│ Resume →   │        │            │
└────────────┘        └────────────┘
```

---

## 4. Synthesis ("Locked in")

### Decisions

| Area                  | Decision                                                                                                                         |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Sort key              | `MAX(completed_at)` per course. No un-mark, no new column. "(or unmarked)" out of scope.                                         |
| Never-started courses | Fall back to `enrolled_at`, sorted **after** courses with completion activity.                                                   |
| Card                  | One `CourseCard`, rich: thumbnail · tag · title · description, then progress + **Resume →** (library) or **Enroll →** (catalog). |

### Derived semantics (rules that fall out)

```
Continue-learning rail  =  enrolled  AND  ≥1 completion  AND  progress < 100
                           → order by MAX(completed_at) desc → take 2  (0–2 cards)

"My Library" tab        =  ALL enrolled courses
                           group A: has ≥1 completion → MAX(completed_at) desc
                           group B: 0 completions     → enrolled_at desc
                           (A always above B)

"Completed" tab         =  progress === 100  (⇔ resumeLessonSlug === null)
```

Three smaller calls made (spec was silent), flagged for confirmation:

- **"My Library" tab shows completed courses too** (literal "all courses the user is enrolled"); they also appear under Completed — Udemy "All courses" parity.
- **`/` now redirects to `/my-library`**; `/workshops` stays reachable via a "Browse workshops" nav link + the library's empty state.
- **Completed cards need a non-Resume CTA** (`resumeLessonSlug` is null at 100%) → **Revisit →** to the course outline.

### Blast radius (no migration — `completed_at` already exists)

```
BACKEND  projects/services
  postgres-enrollments-repository.ts   findCompletedLessons → also SELECT completed_at
  domain/entities/enrollment.ts        carry timestamps → expose lastCompletedAt()
  application/list-my-courses-use-case.ts   EnrolledCourse + lastCompletedAt
  infrastructure/http/serializers.ts   + lastCompletedAt   (+ openapi + tests)

FRONTEND  projects/frontend
  http-course-consumption-repository.ts   EnrolledCoursePayload + toEnrolledCourse
  domain/entities/enrolled-course.ts      + lastCompletedAt
  routes.ts                               MyLibrary, MyLibraryCompleted
  views/router.tsx                        mount /my-library/*  ·  redirect / → /my-library
  NEW  my-library/router.tsx + page + tabs   (reuse WorkshopsUseCasesProvider + useMyCourses)
  NEW  shared CourseCard                   (library + catalog variants)
  workshops-page.tsx                       drop "My courses" rail → catalog only
  tests/route-reachability.integration     assert new routes reachable
```

The only genuinely new behavior is the sort/grouping logic and the tabs; everything else is _moving_ the
existing "My courses" data and _plumbing one already-stored column_ up the stack.

---

## 5. Proposal (`/opsx:propose`) — artifacts produced

Change: **`add-my-library`** at `openspec/changes/add-my-library/` — validates strict, 4/4 artifacts.

| Artifact                             | What it captures                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **proposal.md**                      | Why split the portal; what changes; capabilities (**new** `my-library`; **modified** `course-consumption`, `workshops`); impact across both projects; Non-goals pinning down "no un-mark, no new DB column, no last-accessed tracking."                                                                                                                                                                                               |
| **design.md**                        | 8 decisions — surface `lastCompletedAt`/`enrolledAt` from existing columns (no migration); frontend pure-function ordering with **lexicographic ISO-8601 compare** (no Date/tz hazard); reuse the `workshops` slice (no new gateway); route-based `NavLink` tabs (not the in-page `Tabs` widget); shared `CourseCard`; shared `LearnerShell`; `/` + post-login → `/my-library`; testing split. Plus risks, migration, open questions. |
| **specs/my-library/spec.md**         | New capability — home + redirect + shell, Continue-learning rail, the two ordered tabs, shared card, loading/error/empty states (7 requirements, 22 scenarios).                                                                                                                                                                                                                                                                       |
| **specs/course-consumption/spec.md** | MODIFIED my-courses requirement (full block restated) adding `enrolledAt` + nullable `lastCompletedAt`, with two new timestamp scenarios.                                                                                                                                                                                                                                                                                             |
| **specs/workshops/spec.md**          | MODIFIED catalog route (drops "default post-login destination", adopts shared card, no enrolled rail); REMOVED the "My courses" rail with Reason + Migration pointing at `my-library`.                                                                                                                                                                                                                                                |
| **tasks.md**                         | 6 groups, inside-out (backend timestamps → frontend model → ordering rules → shared UI → pages/routing → reachability + e2e last), each sized to one TDD cycle.                                                                                                                                                                                                                                                                       |

### Notable design decisions (the "why")

1. **Compute `lastCompletedAt` during reconstitution** (repo/use case), passing a single `Maybe<Date>` into
   `Enrollment.restore`, rather than carrying a `Map<lessonId, completedAt>` on the aggregate — the only
   consumer is an ordering key; keeps `completeLesson`/`progress` untouched.
2. **`lastCompletedAt` spans all completion rows** (incl. completions of since-removed lessons), while
   `progress`/`completedLessonIds` count only current lessons — it answers "when did the learner last mark a
   lesson complete," true regardless of later authoring edits. Tiny, documented divergence.
3. **Ordering lives on the frontend as a pure, unit-tested function**; `GET /me/courses` stays an unordered
   set. Timestamps compared as ISO-8601 strings (sorts chronologically as text). Ties break by title then id.
4. **`/my-library` reuses the `workshops` slice** (provider, `useMyCourses`, gateway) — the OpenSpec
   _capability_ `my-library` is its own spec, but capability ≠ code module.
5. **Route-based tabs** (`NavLink`, `aria-current`) give each tab a shareable URL and satisfy the
   reachability rule; the in-page `Tabs` panel widget is not reused.

### Consistency check performed

The post-login destination is defined abstractly in `auth`/`platform-status` (which only pin `/dashboard`
for the owner-bootstrap path) and concretely only in the `workshops` spec — so moving the learner default to
`/my-library` is correctly localized to the `workshops` + `my-library` deltas; no `auth` edit needed.

### Open questions carried into design.md

- My Library tab scope: all enrolled (incl. completed) vs in-progress only. Default: include completed.
- Completed-card CTA: label **Revisit** → course outline. Confirm wording/target.

---

## 6. Process notes / learnings

- **The data-model dig changed the plan.** The phrase "nothing new in the database" looked like a constraint
  on the feature; the real find was that the needed timestamp already exists and is silently dropped on read.
  Reading the _read path_ (not just the schema) is what surfaced it.
- **Literal wording hid a fork.** "(or unmarked)" implied a feature that doesn't exist; surfacing it as an
  explicit question (rather than silently picking one reading) kept scope honest and avoided a phantom
  `last_activity_at` column.
- **Three crisp questions > a long survey.** Each had a recommended default first; the card question used a
  side-by-side ASCII preview so the choice was visual, not abstract.
- **Spec hygiene:** MODIFIED requirements restate the _entire_ block (lose nothing at archive time); the moved
  rail is a REMOVED requirement with Reason + Migration, not a silent deletion.

**Next step:** `/opsx:apply` to implement top-down through `tasks.md` under the RED → GREEN → COMMIT →
REFACTOR → COMMIT protocol (xp-tdd, testing-standard, coding-standards, hexagonal-architecture).
