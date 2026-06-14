# workshops Specification (delta)

## MODIFIED Requirements

### Requirement: Workshops route exists and is protected

The system SHALL expose a `/workshops` route that renders the course catalog, reachable only when a valid session exists and from the shared learner header navigation. The route SHALL remain a consumer of the protected-route primitive for non-owner users. The catalog SHALL list **all** shown courses — including courses the caller is already enrolled in — (title, description preview, thumbnail rendered as `<img>` when set, otherwise a typographic placeholder using existing design tokens) via a TanStack Query `useQuery` over a `CourseConsumptionGateway` port, rendered with the shared course card. Entries for courses the caller is enrolled in (determined by cross-referencing the caller's my-courses query) SHALL display an "Enrolled" badge and a "Go to course" affordance; this enrolled treatment SHALL take precedence over the closed-enrollment treatment. Entries for courses the caller is not enrolled in SHALL display a "Course details" affordance when enrollment is open, while a course with closed enrollment SHALL still be listed (so learners see courses they missed) and SHALL navigate to its outline without offering an enroll affordance. Every entry SHALL navigate to that course's outline at `/workshops/:courseSlug`. The page SHALL render an empty state when no courses are shown, a loading indicator while the catalog query is in flight, and an inline error state with a retry control on failure, all using existing design tokens. While the my-courses query has not resolved, entries SHALL render without the enrolled treatment.

#### Scenario: User reaches the catalog from the learner header

- **GIVEN** an authenticated user on `/my-library`
- **WHEN** they activate the Workshops link in the learner header
- **THEN** they arrive at `/workshops`
- **AND** the course catalog is rendered

#### Scenario: Selecting a course opens its outline

- **GIVEN** a user viewing the catalog with at least one shown course
- **WHEN** they activate a course entry
- **THEN** they navigate to `/workshops/:courseSlug`

#### Scenario: A closed course is listed without an enroll affordance

- **GIVEN** a user viewing the catalog with a course whose enrollment is closed and in which they are not enrolled
- **WHEN** the catalog renders
- **THEN** the closed course is listed so the user can see it
- **AND** the entry communicates that enrollment is closed rather than offering enrollment

#### Scenario: Enrolled course is badged with a Go to course affordance

- **GIVEN** a user enrolled in one of the shown courses
- **WHEN** the catalog renders and the my-courses query resolves
- **THEN** that course's entry displays an "Enrolled" badge
- **AND** its affordance reads "Go to course" and navigates to `/workshops/:courseSlug`

#### Scenario: Enrolled treatment overrides closed enrollment

- **GIVEN** a user enrolled in a course whose enrollment has since been closed
- **WHEN** the catalog renders
- **THEN** that course's entry displays the "Enrolled" badge and "Go to course" affordance
- **AND** it is not presented as enrollment closed

#### Scenario: Non-enrolled open course offers course details

- **GIVEN** a user not enrolled in a shown course with open enrollment
- **WHEN** the catalog renders
- **THEN** that course's entry displays a "Course details" affordance and no "Enrolled" badge

### Requirement: Owner reaches the Dashboard from the Workshops header

The shared learner header SHALL render a navigation control linking to `/dashboard` (using `MainRoutes.Dashboard`) only when the current caller holds the `owner` role, determined via the existing role lookup (`useHasRole('owner')`). The control SHALL be a React Router link (a real anchor, keyboard operable), placed in the header account area alongside the existing email and logout controls, and SHALL consume only existing design tokens. While the role lookup is in flight (unresolved), and for any authenticated non-owner caller, the control SHALL NOT be rendered in the DOM. This control SHALL NOT alter the existing catalog or logout behaviour of the page.

#### Scenario: Owner sees the Dashboard link in the workshops header

- **GIVEN** an authenticated caller whose role is `'owner'` on `/workshops`
- **WHEN** the role lookup resolves
- **THEN** a navigation control linking to `/dashboard` is visible in the header
- **AND** activating it navigates to `/dashboard`

#### Scenario: Non-owner does not see the Dashboard link

- **GIVEN** an authenticated caller whose role is `null` (no role row) on `/workshops`
- **WHEN** the role lookup resolves
- **THEN** no navigation control linking to `/dashboard` is rendered
- **AND** the catalog renders unchanged

#### Scenario: Dashboard link is absent while the role is loading

- **GIVEN** an authenticated caller on `/workshops`
- **WHEN** the role lookup has not yet resolved
- **THEN** no navigation control linking to `/dashboard` is rendered

#### Scenario: Existing workshops behaviour is preserved

- **GIVEN** an authenticated owner on `/workshops`
- **WHEN** the page renders with the Dashboard link present
- **THEN** the logout action and catalog continue to behave as before

## ADDED Requirements

### Requirement: Learner navigates between Workshops and My Library

The Workshops page SHALL render the shared learner header containing React Router navigation links to `/my-library` (My Library) and `/workshops` (Workshops), with `aria-current="page"` exposed on the link matching the current route. The header SHALL also surface the existing account controls (email, logout, owner-only Dashboard link).

#### Scenario: Learner reaches My Library from the workshops header

- **GIVEN** an authenticated user on `/workshops`
- **WHEN** they activate the My Library link in the header
- **THEN** they navigate to `/my-library`

#### Scenario: Active page is marked in the header navigation

- **GIVEN** an authenticated user on `/workshops`
- **WHEN** the header renders
- **THEN** the Workshops link exposes `aria-current="page"`
- **AND** the My Library link does not

## REMOVED Requirements

### Requirement: Learner sees enrolled courses with progress

**Reason**: The learner's enrolled courses move from the workshops page to the new `my-library` capability (continue-learning hero plus All/Completed tabs at `/my-library`).
**Migration**: See `specs/my-library/spec.md`. Progress indicators and completion status render on the shared course card; the "Continue" deep-link to the resume lesson is replaced by cards navigating to the course outline at `/workshops/:courseSlug`, where the existing outline-level continue affordance remains.
