# my-library — Delta

## ADDED Requirements

### Requirement: My Library is the subscriber home

The system SHALL expose a `/my-library` route, behind the protected-route primitive, that renders the authenticated learner's library and SHALL be the default post-login destination; the application root `/` SHALL redirect to `/my-library`. The page SHALL read the caller's enrolled courses via a TanStack Query `useQuery` over the `CourseConsumptionGateway` (the existing "my courses" read) and SHALL render within a shared learner shell that exposes the caller's email, a logout action, an owner-only navigation control to `/dashboard` (rendered only when `useHasRole('owner')` resolves true — never while the role lookup is unresolved, never for a non-owner), and primary navigation linking to `/my-library` and to the workshops catalog at `/workshops`. All strings, states, and indicators SHALL use existing design tokens.

#### Scenario: Authenticated user lands on the library after login

- **GIVEN** a user who has just completed the magic-link callback successfully into an `ACTIVE` platform
- **WHEN** the callback redirects them to the post-login destination
- **THEN** they arrive at `/my-library`
- **AND** their enrolled courses are read and rendered

#### Scenario: Root redirects to the library

- **GIVEN** an authenticated user
- **WHEN** they navigate to `/`
- **THEN** they are redirected to `/my-library`

#### Scenario: Unauthenticated access is blocked

- **GIVEN** a visitor with no valid session
- **WHEN** they navigate to `/my-library`
- **THEN** the protected-route primitive directs them to the login route

#### Scenario: Browse navigation reaches the catalog

- **GIVEN** an authenticated learner on `/my-library`
- **WHEN** they activate the "Browse workshops" navigation control
- **THEN** they navigate to `/workshops`

#### Scenario: Owner reaches the Dashboard from the library header

- **GIVEN** an authenticated caller whose role is `owner` on `/my-library`
- **WHEN** the role lookup resolves
- **THEN** a navigation control linking to `/dashboard` is visible in the header
- **AND** while the role lookup is unresolved, or for a non-owner caller, no such control is rendered

### Requirement: Continue-learning rail surfaces recent in-progress courses

The library SHALL present a "Continue learning" rail of the caller's in-progress courses that have completion activity — courses whose `progress` is less than 100 and whose `lastCompletedAt` is non-null — ordered by `lastCompletedAt` descending and limited to the two most recent. Each rail entry SHALL be a course card offering a resume affordance to the course's first incomplete lesson at `/workshops/:courseSlug/lessons/:lessonSlug`. When no enrolled course qualifies, the rail SHALL NOT be rendered. Recency SHALL be derived from the most recent lesson completion only; the rail SHALL NOT depend on any "last viewed" signal.

#### Scenario: Rail shows the two most recently active in-progress courses

- **GIVEN** an authenticated learner with three in-progress courses whose most recent completions are at different times and one fully completed course
- **WHEN** the library renders
- **THEN** the rail shows exactly the two in-progress courses with the most recent completions, most-recent first
- **AND** the fully completed course is not in the rail

#### Scenario: Resume from a rail card opens the first incomplete lesson

- **GIVEN** a rail card for a course with a known first incomplete lesson
- **WHEN** the learner activates its resume affordance
- **THEN** they navigate to `/workshops/:courseSlug/lessons/:lessonSlug` for that lesson

#### Scenario: Rail is absent when no course has completion activity

- **GIVEN** an authenticated learner whose enrolled courses have no completed lessons
- **WHEN** the library renders
- **THEN** no Continue-learning rail is rendered

### Requirement: My Library tab lists all enrolled courses by recent activity

The `/my-library` index tab SHALL list every course the caller is enrolled in — regardless of `progress` or current course status — using the shared course card. Courses with at least one completed lesson (`lastCompletedAt` non-null) SHALL be ordered by `lastCompletedAt` descending and SHALL appear before courses with no completed lessons; courses with no completed lessons SHALL be ordered by `enrolledAt` descending. Ordering SHALL be deterministic, with ties broken by a fixed secondary key (course title, then id). The tab SHALL be presented alongside a "Completed" tab as a route-based tablist in which each tab is a real navigable link and the active tab is marked with `aria-current`.

#### Scenario: Enrolled courses ordered by most recent completion

- **GIVEN** an authenticated learner enrolled in course A (a lesson completed today), course B (a lesson completed yesterday), and course C (no completed lessons)
- **WHEN** the `/my-library` tab renders
- **THEN** the courses appear in the order A, B, then C

#### Scenario: Never-started courses fall back to enrollment date

- **GIVEN** two enrolled courses with no completed lessons enrolled on different dates, and one enrolled course with completion activity
- **WHEN** the `/my-library` tab renders
- **THEN** the course with completion activity appears first
- **AND** the two never-started courses follow it, most recently enrolled first

#### Scenario: Completed courses still appear in the library tab

- **GIVEN** an authenticated learner with one fully completed course and one in-progress course
- **WHEN** the `/my-library` tab renders
- **THEN** both courses are listed

#### Scenario: Tabs are navigable links

- **GIVEN** an authenticated learner on `/my-library`
- **WHEN** they activate the "Completed" tab
- **THEN** they navigate to `/my-library/completed`
- **AND** the active tab is marked with `aria-current`

### Requirement: Completed tab lists fully completed courses

The system SHALL expose a `/my-library/completed` route, behind the protected-route primitive and within the shared learner shell, that lists only the caller's courses whose `progress` is 100, using the shared course card, ordered by `lastCompletedAt` descending. Each completed card SHALL indicate completion and SHALL offer a "Revisit" affordance navigating to the course outline at `/workshops/:courseSlug` (a fully completed course has no resume lesson). When the caller has no fully completed course, the tab SHALL render an empty state using existing design tokens.

#### Scenario: Only fully completed courses are listed

- **GIVEN** an authenticated learner with one course at 100% progress and one course at 40% progress
- **WHEN** they open `/my-library/completed`
- **THEN** only the 100% course is listed

#### Scenario: Revisit opens the course outline

- **GIVEN** a completed course card
- **WHEN** the learner activates "Revisit"
- **THEN** they navigate to `/workshops/:courseSlug` for that course

#### Scenario: Empty completed tab

- **GIVEN** an authenticated learner with no fully completed course
- **WHEN** they open `/my-library/completed`
- **THEN** an empty state is rendered using existing design tokens

### Requirement: Library and catalog share one course card

Enrolled-course cards (library) and catalog cards (workshops) SHALL be rendered by a single shared course-card component presenting the course thumbnail (an `<img>` when set, otherwise a typographic placeholder using existing design tokens), a course tag, the title, and the description. In a library context for an in-progress course the card SHALL show a completion progress indicator and a resume affordance; for a completed course it SHALL show full completion and a revisit affordance; in a catalog context it SHALL show an open-course affordance for open enrollment and SHALL indicate closed enrollment otherwise. The component SHALL be presentation-only (no data fetching) and SHALL consume only existing design tokens.

#### Scenario: In-progress library card shows progress and resume

- **GIVEN** an enrolled course at 60% progress with a known resume lesson
- **WHEN** its library card renders
- **THEN** the card shows a 60% progress indicator and a resume affordance to that lesson

#### Scenario: Catalog card shows an open-course affordance

- **GIVEN** a shown course with open enrollment in the catalog
- **WHEN** its catalog card renders
- **THEN** the card shows an affordance navigating to the course outline at `/workshops/:courseSlug`

### Requirement: Library loading, error, and empty states

While the enrolled-courses query is in flight the library SHALL render a loading indicator; on failure it SHALL render an inline error state with a retry control; when the caller has no enrollments it SHALL render an empty state whose primary affordance navigates to the workshops catalog at `/workshops`. All states SHALL use existing design tokens.

#### Scenario: Loading state

- **GIVEN** an authenticated learner whose enrolled-courses query has not resolved
- **WHEN** the library renders
- **THEN** a loading indicator is shown

#### Scenario: Error state with retry

- **GIVEN** an authenticated learner whose enrolled-courses query has failed
- **WHEN** the library renders
- **THEN** an inline error state with a retry control is shown
- **AND** activating retry refetches the enrolled courses

#### Scenario: Empty state links to the catalog

- **GIVEN** an authenticated learner with no enrollments
- **WHEN** the library renders
- **THEN** an empty state is shown
- **AND** its primary affordance navigates to `/workshops`
