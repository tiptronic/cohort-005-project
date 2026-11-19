# Cadence events that could trigger Notifications

Research for [#6](https://github.com/tiptronic/cohort-005-project/issues/6), part of the map [#4 Wayfinder: In-app Notifications](https://github.com/tiptronic/cohort-005-project/issues/4).

**Question:** Which moments in the current codebase could trigger a Notification, and who would care about each?

**Source:** the codebase at commit `37ff89a`. Cadence has no service layer. All business logic lives in route `action`s and the helper functions next to them in `app/routes/*`, so every citation below points to a route file or to `app/db/schema.ts`. Line numbers refer to that commit.

This file is only an inventory. Which Notifications v1 sends is decided later, in [#9](https://github.com/tiptronic/cohort-005-project/issues/9).

## Who the recipients are

- **Student**: `UserRole.Student` (`app/db/schema.ts:12`). The refund code and `GLOSSARY.md` call this person the "learner".
- **Instructor**: `UserRole.Instructor`. A course has exactly one owning Instructor, `courses.instructorId` (`app/db/schema.ts:73`).
- **Platform Admin**: `UserRole.Admin` (`app/db/schema.ts:14`). Platform Admins have staff rights on every course (`app/lib/access.server.ts`, `isStaff`).
- **Team Admin**: a `team_members` row with `TeamMemberRole.Admin` (`app/db/schema.ts:41`, `:263`). This is the person who bought Seats for a team. The team is created on their first team purchase by `getOrCreateTeamForUser` (`app/routes/courses.$slug.purchase.tsx:210`).

## Starting candidates

| # | Event | Exists? | Where it happens | Recipient(s) |
|---|-------|---------|------------------|--------------|
| 1 | Refund Request approved / declined | **No** | `RefundStatus.Approved` and `RefundStatus.Declined` exist (`app/db/schema.ts:35-39`), and `refunds.resolvedAt`/`resolvedByUserId` are ready for them (`app/db/schema.ts:250-251`). No code writes them, and no route or UI lets anyone decide a Refund Request. `RefundStatus.Open` is the only value ever written (`app/routes/courses.$slug.refund.ts:213`, `scripts/seed.ts:2778`). | Student (would be) |
| 2 | Refund Issued / Failed | **No** | The glossary defines Pending, Issued and Failed Refunds, but the schema has no column or status for them. `refunds` stores only `amountRefunded` (`app/db/schema.ts:246`), and no payment provider is integrated. | Student. Platform Admin for a Failed Refund. (Both would be.) |
| 3 | New Open Refund Request | **Yes** | `action` in `app/routes/courses.$slug.refund.ts:113` inserts the row at `:209-216`. The same action already builds a confirmation email to the Student (`buildRefundRequestEmail`, `:91`) and only logs it to the console (`:219-228`), because no mail transport is wired up. That email promises "we will email you again when it is done". | Platform Admins, the only role that could decide a Refund Request. The Student could also get a confirmation, and the course Instructor might care. |
| 4 | Seat claimed | **Yes** | `redeemCoupon` in `app/routes/redeem.$code.tsx:41` sets `coupons.redeemedAt` (`:84-90`) and creates the claimant's Enrollment (`:92-96`). The `action` calls it at `:179`. A Seat is a `coupons` row. | Team Admin of `coupons.teamId`. The claiming Student may also be told about their new Enrollment. |
| 5 | Question (comment) on a course | **Yes**, but on a **lesson**, not a course | `createComment` with `parentId = null` (`app/routes/courses.$slug.lessons.$lessonId.tsx:404`, insert at `:430-434`), called from the `create-comment` intent at `:898-909`. A "question" is a top-level comment. `/instructor/questions` lists unanswered questions (`getUnansweredQuestions`, `app/routes/instructor.questions.tsx:42`). | Course Instructor. Platform Admins also see the queue. |
| 6 | Reply to a question | **Yes**, through two paths | (a) Anyone with access replying on the lesson page: the same `createComment` with `parentId` set (`courses.$slug.lessons.$lessonId.tsx:404`, called at `:903`). (b) Staff replying from the queue: the `action` in `app/routes/instructor.questions.tsx:188` calls a separate copy of `createComment` (`:118`, insert at `:152-156`) at `:211`. Replies go only one level deep. | Author of the question. Others who replied in the same thread might also care. |
| 7 | New Enrollment | **Yes**, through two paths | (a) Self purchase: `action` in `app/routes/courses.$slug.purchase.tsx:341` calls `createPurchase` (`:180`) at `:370` and `enrollUser` (`:145`) at `:371`. `enrollUser` has a `sendEmail` flag with the stub `// Would send welcome email here` (`:174`), and this call passes `false`. (b) Seat claim: `app/routes/redeem.$code.tsx:92-96`. | Course Instructor, who gains a Student. The Student could also get a welcome. |
| 8 | New lesson/module published on an enrolled course | **No publish moment** | Modules and lessons have no status or draft flag. They go live as soon as they are created: `createModule` (`app/routes/instructor.$courseId.tsx:207`, `add-module` intent at `:723`) and `createLesson` (`:265`, `add-lesson` intent at `:762`). A new lesson starts with no content, which arrives later through `updateLesson` (`app/routes/instructor.$courseId.lessons.$lessonId.tsx:66`, called at `:226`). A "new lesson" Notification would first need a definition of when a lesson counts as ready. Only courses have a publish step (see A1). | Students enrolled in the course (would be) |

## Other events found

| # | Event | Where it happens | Recipient(s) |
|---|-------|------------------|--------------|
| A1 | Course status change (Draft / Published / Archived) | Instructor: `update-status` intent in `app/routes/instructor.$courseId.tsx:695-698` calls `updateCourseStatus` (`:154`). Platform Admin: `action` in `app/routes/admin.courses.tsx:141-144` calls `updateCourseStatus` (`:76`). | Course Instructor, when a Platform Admin changes the status. Enrolled Students, when a course is archived. Platform Admins, when a course is published. |
| A2 | Team purchase (Seats bought) | `action` in `app/routes/courses.$slug.purchase.tsx:376-378` calls `createTeamPurchase` (`:245`), which creates a Purchase, creates or reuses the team (`:210`) and generates the coupons (`generateCoupons`, `:222`). | Team Admin (confirmation). Course Instructor (a sale). |
| A3 | Self purchase (a sale) | `createPurchase` at `app/routes/courses.$slug.purchase.tsx:370`, in the same action as the Enrollment in #7a | Course Instructor |
| A4 | Role changed by a Platform Admin | `update-role` intent in `app/routes/admin.users.tsx:126-130` calls `updateUserRole` (`:60`) | The affected user, for example on promotion to Instructor |
| A5 | Profile edited by a Platform Admin | `update-user` intent in `app/routes/admin.users.tsx:119-124` | The affected user |
| A6 | Comment deleted by staff | `deleteComment` in `app/routes/courses.$slug.lessons.$lessonId.tsx:459` lets staff delete other people's comments (`:464`). It is called at `:935`. | Author of the deleted comment |
| A7 | Course rated | `rateCourse` in `app/routes/courses.$slug.tsx:334`, called by `action` at `:601`. A rating is 1–5 stars, and a Student can change theirs later. | Course Instructor |
| A8 | Quiz attempt submitted / passed | `computeResult` in `app/routes/courses.$slug.lessons.$lessonId.tsx:531` inserts `quizAttempts` with `passed` (`:599-611`). Called by the `submit-quiz` intent at `:970`. | The Student already sees the result on the page, so a Notification adds little. The Instructor might care. |
| A9 | Lesson completed / course completed | `markLessonComplete` in `app/routes/courses.$slug.lessons.$lessonId.tsx:218`, called at `:948`. Course completion is never stored. Detecting it would mean comparing a Student's completed lessons with all lessons in the course. | Student (a milestone). The Instructor might care. |
| A10 | New course created | `createCourse` in `app/routes/instructor.new.tsx:46`, called at `:149`. A new course always starts as Draft. | Platform Admins (low value) |
| A11 | Sign-up | insert at `app/routes/signup.tsx:78`. A new user always starts as a Student. | The new Student (a welcome). Platform Admins could also be told. |

## Ruled out or not modelled

- **Refund Request decisions (#1)** and **Refund Issued/Failed (#2)** depend on work that does not exist yet: a review UI for the team, and states that track the money. The Notification spec has to wait for that work or define it.
- **New lesson published (#8)** has no publish moment in the data model.
- **Seat changes other than a claim**: a Seat cannot be revoked, reassigned or expire. `coupons` has only `redeemedAt` and `redeemedByUserId` (`app/db/schema.ts:277`).
- **Video watch events** (`app/routes/api.video-tracking.ts:30`) are telemetry and not worth a Notification.

## Observations for later tickets

- Two placeholders for outbound messages already exist: the console-logged refund confirmation (`app/routes/courses.$slug.refund.ts:91-228`) and the `sendEmail` stub in `enrollUser` (`app/routes/courses.$slug.purchase.tsx:174`). They show where a Notification could be sent from.
- Every trigger point is a synchronous SQLite write inside a route action. No trigger uses a transaction, a queue or an event bus. This bears on the map's open "Failure handling" question.
- `createComment` exists twice, in the lesson route and in `instructor.questions.tsx`. A Notification hook has to go into both copies, unless they are merged first.
