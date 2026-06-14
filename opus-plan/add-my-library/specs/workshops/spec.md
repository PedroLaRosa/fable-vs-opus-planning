# workshops — Delta

## MODIFIED Requirements

### Requirement: Workshops route exists and is protected

The system SHALL expose a `/workshops` route that renders the course catalog, reachable only when a valid session exists, and SHALL remain a consumer of the protected-route primitive for non-owner users. The `/workshops` route SHALL be reachable from the shared learner shell's "Browse workshops" navigation; it is NOT the default post-login destination (that is `/my-library`). The catalog SHALL list shown courses (title, description preview, thumbnail rendered as `<img>` when set, otherwise a typographic placeholder using existing design tokens) via a TanStack Query `useQuery` over a `CourseConsumptionGateway` port, rendered with the shared course card. Each catalog entry SHALL reflect whether the course's enrollment is open: a course with open enrollment SHALL be an affordance that navigates to that course's outline at `/workshops/:courseSlug`, while a course with closed enrollment SHALL still be listed (so learners see courses they missed) and SHALL navigate to its outline without offering an enroll affordance. The page SHALL NOT render the caller's enrolled courses or their progress (that is the `my-library` capability). The page SHALL render an empty state when no courses are shown, a loading indicator while the catalog query is in flight, and an inline error state with a retry control on failure, all using existing design tokens.

#### Scenario: Catalog renders for an authenticated learner

- **GIVEN** an authenticated learner
- **WHEN** they navigate to `/workshops`, for example via the shell's "Browse workshops" control
- **THEN** the course catalog is rendered

#### Scenario: Selecting a course opens its outline

- **GIVEN** a user viewing the catalog with at least one shown course
- **WHEN** they activate a course entry
- **THEN** they navigate to `/workshops/:courseSlug`

#### Scenario: A closed course is listed without an enroll affordance

- **GIVEN** a user viewing the catalog with a course whose enrollment is closed
- **WHEN** the catalog renders
- **THEN** the closed course is listed so the user can see it
- **AND** the entry communicates that enrollment is closed rather than offering enrollment

## REMOVED Requirements

### Requirement: Learner sees enrolled courses with progress

**Reason**: The enrolled-courses progress view (the "My courses" rail) moves off `/workshops` and into the new `my-library` capability; `/workshops` becomes the enroll-only catalog.

**Migration**: Enrolled courses with their progress and continue/resume affordance now render on `/my-library` — the Continue-learning rail (two most recently active in-progress courses) and the "My Library" tab (all enrolled courses, ordered by most recent completion), with fully completed courses also under `/my-library/completed`. The underlying `GET /me/courses` read is unchanged except that it now also carries `lastCompletedAt` and `enrolledAt` (see the `course-consumption` delta). No data is lost; only the rendering location moves.
