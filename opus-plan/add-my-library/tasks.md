# Tasks — Add My Library

## 1. Backend — surface enrolment and last-completion timestamps

- [ ] 1.1 Let the enrollment aggregate carry the time of its most recent lesson completion, absent when no lesson is completed
- [ ] 1.2 Expose the enrolment time and the last-completion time on the "my courses" result
- [ ] 1.3 Reconstitute the last-completion time from the stored completion timestamps when reading enrolments from Postgres
- [ ] 1.4 Include `enrolledAt` and a nullable `lastCompletedAt` in the my-courses response payload
- [ ] 1.5 Document `enrolledAt` and `lastCompletedAt` on the my-courses OpenAPI schema

## 2. Frontend — carry the timestamps through the consumption slice

- [ ] 2.1 Extend the enrolled-course model with the enrolment time and the nullable last-completion time
- [ ] 2.2 Map the new timestamp fields from the my-courses read in both the HTTP gateway and the in-memory fake

## 3. Frontend — library ordering rules

- [ ] 3.1 Derive the Continue-learning selection: in-progress courses with completion activity, two most recent first
- [ ] 3.2 Derive the My Library ordering: courses with completion activity by most recent completion, then never-started courses by most recent enrolment
- [ ] 3.3 Derive the Completed listing: fully completed courses by most recent completion

## 4. Frontend — shared presentation

- [ ] 4.1 Build the shared course card with library in-progress, library completed, and catalog variants
- [ ] 4.2 Build the shared learner shell with the account chrome and primary navigation between My Library and the workshops catalog

## 5. Frontend — pages and routing

- [ ] 5.1 Register the My Library and Completed routes and redirect the application root and post-login destination to My Library
- [ ] 5.2 Render the Continue-learning rail on the My Library home
- [ ] 5.3 Render the My Library enrolled tab with its loading, error, and empty-with-browse states
- [ ] 5.4 Render the Completed tab with its empty state
- [ ] 5.5 Reduce the workshops page to the enroll-only catalog using the shared card, removing the enrolled-courses rail

## 6. Reachability and end-to-end coverage

- [ ] 6.1 Update the route-reachability coverage for the new routes and the affordances that reach them
- [ ] 6.2 Review the spec deltas and implement every e2e-representable scenario (enrol, complete a lesson so the course tops the rail and the library tab, complete the course so it leaves the rail and appears under Completed, and navigate between the library and the catalog)
