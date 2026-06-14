# course-consumption — Delta

## MODIFIED Requirements

### Requirement: Learner lists their enrolled courses with progress

The system SHALL expose `GET /me/courses` to any authenticated caller, responding `200 OK` with an array of the caller's enrolled courses **regardless of each course's current `status`** (`open`, `closed`, `unlisted`, or `hidden`). An enrolment SHALL NEVER be dropped from this listing because the course was later closed, unlisted, or hidden. Each entry SHALL include the course summary (`id`, `title`, `description`, `thumbnailUrl`), the derived completion `progress` (a percentage 0–100), the `resumeLessonId` — the id of the first lesson (by section then lesson position) the caller has not completed, or `null` when all lessons are complete — and `completedLessonIds`, the ids of the lessons in the course that the caller has completed (an empty array when none). Lesson ids in `completedLessonIds` that no longer exist in the course SHALL NOT be included. Each entry SHALL additionally include `enrolledAt` — the ISO-8601 timestamp of the enrolment — and `lastCompletedAt` — the ISO-8601 timestamp of the most recent lesson completion recorded for the enrolment, or `null` when the caller has completed no lessons. Both timestamps SHALL be derived from existing data (`enrollments.enrolled_at` and `lesson_completions.completed_at`) with no new persistence. Each listed course SHALL remain fully accessible to the caller (course outline and lesson bodies) under the existing enrolled-learner access rules, so the caller can always resume study. Courses the caller is not enrolled in SHALL NOT appear.

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

#### Scenario: My courses entry includes enrolment and last-completion timestamps

- **GIVEN** an authenticated user enrolled in a course who has completed at least one lesson
- **WHEN** the client sends `GET /me/courses`
- **THEN** that course's entry includes an `enrolledAt` ISO-8601 timestamp
- **AND** its `lastCompletedAt` equals the time of the most recent lesson completion for that enrolment

#### Scenario: Enrolled course with no completed lessons reports null last-completion

- **GIVEN** an authenticated user enrolled in a course in which they have completed no lessons
- **WHEN** the client sends `GET /me/courses`
- **THEN** that course's entry includes an `enrolledAt` ISO-8601 timestamp
- **AND** its `lastCompletedAt` is `null`

#### Scenario: Fully completed course reports no resume target

- **GIVEN** an authenticated user who has completed every lesson of an enrolled course
- **WHEN** the client sends `GET /me/courses`
- **THEN** that course's `progress` is 100
- **AND** its `resumeLessonId` is `null`
- **AND** its `completedLessonIds` contains every current lesson id of the course
- **AND** its `lastCompletedAt` is non-null

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
