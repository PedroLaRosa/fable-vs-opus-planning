# my-library Specification (delta)

## ADDED Requirements

### Requirement: My Library is the learner home

The system SHALL expose a protected `/my-library` route (registered as `MainRoutes.MyLibrary`) that renders the caller's enrolled courses. The root path `/` SHALL redirect to `/my-library`, and a successful magic-link callback SHALL navigate to `/my-library`. The route SHALL be reachable from at least one rendered navigation affordance (the shared learner header), and SHALL consume the existing course-consumption data path (the `CourseConsumptionGateway` port and the shared my-courses query key) so that enrollment and lesson-completion mutations refresh the library. The page SHALL render a loading indicator while the my-courses query is in flight and an inline error state with a retry control on failure, using existing design tokens.

#### Scenario: Authenticated user lands on My Library after login

- **GIVEN** a user who has just completed the magic-link callback successfully
- **WHEN** the callback page redirects them
- **THEN** they arrive at `/my-library`

#### Scenario: Root path redirects to My Library

- **GIVEN** an authenticated user
- **WHEN** they visit `/`
- **THEN** they are redirected to `/my-library`

#### Scenario: Unauthenticated caller is redirected to login

- **GIVEN** no valid session
- **WHEN** the user visits `/my-library`
- **THEN** they are redirected to the login route

#### Scenario: Library shows a loading indicator while courses load

- **GIVEN** an authenticated user whose my-courses query is in flight
- **WHEN** `/my-library` renders
- **THEN** a loading indicator is shown

#### Scenario: Library shows an error state with retry on failure

- **GIVEN** an authenticated user whose my-courses query fails
- **WHEN** `/my-library` renders
- **THEN** an inline error state with a retry control is shown
- **AND** activating retry re-issues the query

### Requirement: Continue-learning hero highlights resumable courses

The `/my-library` page SHALL render a "Continue learning" section above the tabs containing at most two course cards. Eligible courses are those with at least one completed lesson and `progress < 100`, ordered by `lastCompletedLessonAt` descending. When fewer than two courses are eligible, the remaining slots SHALL be filled with untouched courses (zero completed lessons), ordered by `enrolledAt` descending. Fully completed courses SHALL NEVER appear in the hero. The section SHALL NOT be rendered when it would contain no cards (in particular when the caller has no enrollments).

#### Scenario: Hero shows the two most recently progressed unfinished courses

- **GIVEN** a learner enrolled in three unfinished courses with completions, most recent first: A, B, C
- **WHEN** `/my-library` renders
- **THEN** the hero shows exactly A and B in that order

#### Scenario: Hero fills remaining slots with the most recently enrolled untouched courses

- **GIVEN** a learner with one unfinished course with completions and two untouched courses, the newest enrollment being U1
- **WHEN** `/my-library` renders
- **THEN** the hero shows the in-progress course first and U1 second

#### Scenario: Completed courses never appear in the hero

- **GIVEN** a learner whose most recently progressed course is 100% complete
- **WHEN** `/my-library` renders
- **THEN** that course is absent from the hero

#### Scenario: Hero is hidden when the learner has no enrollments

- **GIVEN** an authenticated user with no enrollments
- **WHEN** `/my-library` renders
- **THEN** no "Continue learning" section is rendered

### Requirement: Library tabs filter enrolled courses by completion status

The page SHALL render route-driven tabs as a navigation landmark with two React Router links exposing `aria-current="page"` on the active entry: **All courses** at `/my-library` and **Completed** at `/my-library/completed` (registered as `MainRoutes.MyLibraryCompleted`). The All courses tab SHALL list every enrolled course, including completed ones, sorted by last engagement descending — `lastCompletedLessonAt` when present, otherwise `enrolledAt` — with deterministic tiebreaks (`enrolledAt` descending, then title ascending). The Completed tab SHALL list only courses whose `progress` equals 100, in the same ordering. Both routes SHALL render the same page component, and each tab SHALL render an empty state when it has no courses, using existing design tokens.

#### Scenario: All courses tab sorts by most recent engagement

- **GIVEN** a learner enrolled in courses A and B where B has the more recent lesson completion
- **WHEN** `/my-library` renders
- **THEN** B is listed before A

#### Scenario: Marking a lesson complete moves that course to the top

- **GIVEN** a learner on the All courses tab where course A is listed below course B
- **WHEN** they mark a lesson of A complete and return to `/my-library`
- **THEN** A is listed first

#### Scenario: Newly enrolled untouched course sorts by enrollment date

- **GIVEN** a learner who just enrolled in course N and has older completions elsewhere
- **WHEN** `/my-library` renders
- **THEN** N is listed first (its `enrolledAt` is the most recent engagement)

#### Scenario: Completed tab lists only fully completed courses

- **GIVEN** a learner with one course at `progress` 100 and one at `progress` 50
- **WHEN** they visit `/my-library/completed`
- **THEN** only the fully completed course is listed

#### Scenario: Tabs navigate between routes and mark the active tab

- **GIVEN** a learner on `/my-library`
- **WHEN** they activate the Completed tab
- **THEN** they navigate to `/my-library/completed`
- **AND** the Completed link exposes `aria-current="page"`

#### Scenario: Completed tab shows an empty state

- **GIVEN** a learner with enrollments but no fully completed course
- **WHEN** they visit `/my-library/completed`
- **THEN** an empty state is rendered

### Requirement: Library course cards link to the existing course routes

Each course in the hero and tab lists SHALL render the shared course card: thumbnail rendered as `<img>` when set (otherwise a typographic placeholder), title, description preview, a progress indicator with a completion status derived solely from `progress === 100` ("Completed") versus otherwise ("In progress"), and a "Go to course" affordance. Cards SHALL navigate to the course outline at `/workshops/:courseSlug` — the my-library capability SHALL NOT introduce routes for course consumption.

#### Scenario: Card navigates to the course outline

- **GIVEN** a learner viewing their library with an enrolled course
- **WHEN** they activate the course card
- **THEN** they navigate to `/workshops/:courseSlug` for that course

#### Scenario: Card derives completion status from progress

- **GIVEN** an enrolled course whose `progress` is 100
- **WHEN** its card renders
- **THEN** the card communicates "Completed"

#### Scenario: A course with no lessons is shown as in progress

- **GIVEN** an enrolled course with zero lessons (`progress` 0)
- **WHEN** its card renders
- **THEN** the card communicates "In progress" with 0% progress
- **AND** it never appears in the Completed tab
