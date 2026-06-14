# course-consumption Specification (delta)

## MODIFIED Requirements

### Requirement: Learner browses the course catalog

The system SHALL expose `GET /courses` to any authenticated caller (composing `authMiddleware` only, NOT `requireRole`). The response SHALL be a `200 OK` array of course summaries (`id`, `slug`, `title`, `description`, `thumbnailUrl`, `enrollmentOpen`) for **shown** courses only (status `open` or `closed`), sorted by `createdAt` descending. The catalog SHALL include courses the caller is already enrolled in; distinguishing enrolled entries is a client concern (cross-referenced with `GET /me/courses`). `enrollmentOpen` SHALL be `true` for `open` courses and `false` for `closed` courses. Courses that are `hidden` or `unlisted` SHALL NOT appear. The response SHALL NOT include sections, lessons, or `contentJson`. Unauthenticated callers SHALL receive `401`.

#### Scenario: Catalog lists shown courses with enrollability

- **GIVEN** an authenticated user and four courses with statuses `open`, `closed`, `unlisted`, and `hidden`
- **WHEN** the client sends `GET /courses`
- **THEN** the server responds `200 OK` with exactly the `open` and `closed` course summaries
- **AND** the `open` course has `enrollmentOpen: true` and the `closed` course has `enrollmentOpen: false`
- **AND** the `unlisted` and `hidden` courses are absent
- **AND** no `sections` or `contentJson` fields are present

#### Scenario: Enrolled course appears in the catalog

- **GIVEN** an authenticated user enrolled in one of two `open` courses
- **WHEN** the client sends `GET /courses`
- **THEN** the server responds `200 OK` with both course summaries

#### Scenario: Catalog lists all shown courses when enrolled in every one

- **GIVEN** an authenticated user enrolled in every shown course
- **WHEN** the client sends `GET /courses`
- **THEN** the server responds `200 OK` with every shown course summary

#### Scenario: Catalog is empty when nothing is shown

- **GIVEN** an authenticated user and only `hidden` or `unlisted` courses
- **WHEN** the client sends `GET /courses`
- **THEN** the server responds `200 OK` with an empty array

#### Scenario: Unauthenticated caller is rejected

- **GIVEN** a request without a valid bearer token
- **WHEN** the client sends `GET /courses`
- **THEN** the server responds `401 Unauthorized`

### Requirement: Learner lists their enrolled courses with progress

The system SHALL expose `GET /me/courses` to any authenticated caller, responding `200 OK` with an array of the caller's enrolled courses **regardless of each course's current `status`** (`open`, `closed`, `unlisted`, or `hidden`). An enrolment SHALL NEVER be dropped from this listing because the course was later closed, unlisted, or hidden. Each entry SHALL include the course summary (`id`, `title`, `description`, `thumbnailUrl`), the derived completion `progress` (a percentage 0–100), the `resumeLessonId` — the id of the first lesson (by section then lesson position) the caller has not completed, or `null` when all lessons are complete — `completedLessonIds`, the ids of the lessons in the course that the caller has completed (an empty array when none), `enrolledAt` — the enrollment timestamp as an ISO 8601 string — and `lastCompletedLessonAt` — the most recent `completed_at` among the enrollment's lesson completions as an ISO 8601 string, or `null` when the caller has completed no lessons. Lesson ids in `completedLessonIds` that no longer exist in the course SHALL NOT be included. The order of the array is unspecified; ordering is a client concern. Each listed course SHALL remain fully accessible to the caller (course outline and lesson bodies) under the existing enrolled-learner access rules, so the caller can always resume study. Courses the caller is not enrolled in SHALL NOT appear.

#### Scenario: Learner sees enrolled courses with progress and resume target

- **GIVEN** an authenticated user enrolled in two courses, having completed some lessons in one
- **WHEN** the client sends `GET /me/courses`
- **THEN** the server responds `200 OK` with both enrolled courses
- **AND** each includes a `progress` percentage and a `resumeLessonId`
- **AND** the `resumeLessonId` is the first lesson not yet completed by position order

#### Scenario: My courses entry includes completed lesson ids

- **GIVEN** an authenticated user enrolled in a published course with four lessons who has completed exactly two of them
- **WHEN** the client sends `GET /me/courses`
- **THEN** that course's entry includes `completedLessonIds` containing exactly those two lesson ids
- **AND** its `progress` is `50`

#### Scenario: My courses entry reports the latest completion timestamp

- **GIVEN** an authenticated user enrolled in a course who completed two lessons at different times
- **WHEN** the client sends `GET /me/courses`
- **THEN** that course's entry includes `enrolledAt` as an ISO 8601 string
- **AND** its `lastCompletedLessonAt` equals the most recent of the two completion timestamps

#### Scenario: Untouched enrollment reports no completion timestamp

- **GIVEN** an authenticated user enrolled in a course with no completed lessons
- **WHEN** the client sends `GET /me/courses`
- **THEN** that course's entry includes its `enrolledAt`
- **AND** its `lastCompletedLessonAt` is `null`

#### Scenario: Fully completed course reports no resume target

- **GIVEN** an authenticated user who has completed every lesson of an enrolled course
- **WHEN** the client sends `GET /me/courses`
- **THEN** that course's `progress` is 100
- **AND** its `resumeLessonId` is `null`
- **AND** its `completedLessonIds` contains every current lesson id of the course

#### Scenario: Learner with no enrollments

- **GIVEN** an authenticated user with no enrollments
- **WHEN** the client sends `GET /me/courses`
- **THEN** the server responds `200 OK` with an empty array

#### Scenario: Enrolled course that has been closed still appears

- **GIVEN** an authenticated learner enrolled in a course whose status is later set to `closed`
- **WHEN** the client sends `GET /me/courses`
- **THEN** the server responds `200 OK` and that course appears in the list
- **AND** it includes its `progress` percentage, `resumeLessonId`, and `completedLessonIds`

#### Scenario: Enrolled course that has been unlisted still appears

- **GIVEN** an authenticated learner enrolled in a course whose status is later set to `unlisted`
- **WHEN** the client sends `GET /me/courses`
- **THEN** the server responds `200 OK` and that course appears in the list
- **AND** it includes its `progress` percentage, `resumeLessonId`, and `completedLessonIds`

#### Scenario: Enrolled course that has been hidden still appears

- **GIVEN** an authenticated learner enrolled in a course whose status is later set to `hidden`
- **WHEN** the client sends `GET /me/courses`
- **THEN** the server responds `200 OK` and that course appears in the list
- **AND** it includes its `progress` percentage, `resumeLessonId`, and `completedLessonIds`

### Requirement: Learner marks a lesson complete and progress is derived

The system SHALL expose `POST /courses/:courseId/lessons/:lessonId/complete` to any authenticated caller who is enrolled in the course, regardless of the course's `status`. Marking complete SHALL record the lesson as completed for that enrollment and SHALL be idempotent (a repeated call leaves a single completion). Completion uniqueness SHALL be enforced by a `UNIQUE(enrollment_id, lesson_id)` constraint. A caller who is not enrolled SHALL receive `403`. The course's completion percentage SHALL be derived as the count of completed lessons that still exist in the course divided by the total number of lessons in the course, **rounded down to the nearest integer**, so that `progress` is `100` if and only if every current lesson is complete; it SHALL NOT be stored.

#### Scenario: Enrolled learner marks a lesson complete

- **GIVEN** an authenticated user enrolled in a course with four lessons
- **WHEN** the client sends `POST /courses/:courseId/lessons/:lessonId/complete` for one lesson
- **THEN** the server responds `200 OK`
- **AND** that lesson is recorded as completed for the caller's enrollment

#### Scenario: Enrolled learner marks complete after the course is hidden

- **GIVEN** an authenticated user enrolled in a course that has since been set to `hidden`
- **WHEN** the client sends `POST /courses/:courseId/lessons/:lessonId/complete`
- **THEN** the server responds `200 OK`
- **AND** that lesson is recorded as completed for the caller's enrollment

#### Scenario: Marking the same lesson complete twice is idempotent

- **GIVEN** an enrolled user who has already completed a lesson
- **WHEN** the client sends the complete request for that lesson again
- **THEN** the server responds `200 OK`
- **AND** exactly one completion row exists for that enrollment and lesson

#### Scenario: Non-enrolled caller cannot mark completion

- **GIVEN** an authenticated user not enrolled in the course
- **WHEN** the client sends `POST /courses/:courseId/lessons/:lessonId/complete`
- **THEN** the server responds `403 Forbidden`
- **AND** no completion row is created

#### Scenario: Progress reflects current lessons after authoring changes

- **GIVEN** an enrolled user who has completed all four lessons of a course (progress 100%)
- **WHEN** the owner later adds a fifth lesson to the course
- **THEN** the caller's derived progress for that course becomes 80% (four of five)

#### Scenario: Progress never reports 100 for an incomplete course

- **GIVEN** an enrolled user who has completed two of three lessons of a course
- **WHEN** the client derives the course's progress via `GET /me/courses`
- **THEN** the `progress` is `66` (rounded down)
- **AND** `progress` `100` is reported only when all three lessons are complete
