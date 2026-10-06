# 06. API Specification

**Purpose:** contract for the REST API. **Scope:** all endpoints under `/api/v1/`. The OpenAPI document generated from code is the machine-readable source; this file is the design contract it must satisfy.

See also: [Backend Architecture](04-backend-architecture.md) · [Database Design](05-database-design.md) · [Authentication](07-authentication.md) · [Authorization](08-authorization.md) · [AI Architecture](09-ai-architecture.md) · [Google Integrations](13-google-integrations.md)

## 1. Conventions

| Topic | Rule |
|---|---|
| Base path | `/api/v1/`; health probes at `/healthz` and `/readyz` (unversioned) |
| Format | JSON, UTF-8; IDs are UUID strings; timestamps ISO-8601 UTC with `Z`; dates `YYYY-MM-DD`; times `HH:MM` |
| Auth | `Authorization: Bearer <access JWT>`; refresh via HttpOnly cookie on `/auth/*` only |
| Auth codes | **P** public · **U** authenticated user acting on own resources · **A** admin role (MFA once enforced) |
| Authorization | **Owner** = resource `user_id` equals token subject; otherwise **404 `not_found`**. `user_id` is never accepted in bodies |
| Pagination | Cursor: `?limit=25&cursor=…` (limit 1–100). List response `Page<T>` |
| Sorting/filtering | `sort=field` or `sort=-field` from a documented allowlist; filters as query params |
| Idempotency | `Idempotency-Key` header (≤ 64 chars) on endpoints marked **†**; same key + same body replays the stored response for 24 h; same key + different body → 409 |
| Async | Endpoints marked **⏳** return `202 Accepted` with `AIRequest` or job resource and `Location` header |
| Concurrency | PATCH on tasks, subtasks, planned sessions requires `version`; mismatch → 409 `version_conflict` |
| Rate limits | Headers `RateLimit-Limit`, `RateLimit-Remaining`, `Retry-After` on 429; limits in [Security](17-security.md#3-rate-limiting) |
| Unknown fields | Rejected (`extra=forbid`) to prevent mass assignment |
| Versioning | Breaking changes require `/api/v2`; additive changes are non-breaking |

## 2. Errors

Every non-2xx response uses one envelope:

```json
{
  "error": {
    "code": "validation_error",
    "message": "Request validation failed.",
    "details": [{ "field": "title", "issue": "String should have at most 200 characters" }],
    "request_id": "01JABCDEF..."
  }
}
```

| HTTP | `code` | Meaning |
|---|---|---|
| 400 | `bad_request` | Malformed JSON, invalid cursor, bad header |
| 401 | `unauthenticated` | Missing/expired/invalid token |
| 403 | `forbidden` | Authenticated but lacking role (for example non-admin on `/admin`), unverified email where required, suspended account |
| 404 | `not_found` | Missing **or not owned** |
| 409 | `conflict`, `version_conflict`, `dependency_cycle`, `session_overlap`, `session_active`, `stale_inputs`, `incomplete_onboarding`, `integration_needs_reauth`, `already_exists` | State conflict |
| 413 | `payload_too_large` | Upload over limit |
| 415 | `unsupported_media_type` | File type not allowed |
| 422 | `validation_error` | Schema/business validation failed |
| 429 | `rate_limited`, `ai_quota_exceeded` | Throttled |
| 502/503 | `upstream_unavailable`, `ai_unavailable` | Provider or Google outage |
| 500 | `internal_error` | Unexpected; no internals leaked |

"Errors" columns below list only codes **beyond** the common ones (401 on U/A endpoints, 422 on bodies, 429, 500).

## 3. Schemas

Fields marked `*` are required on create. All response objects also include `id`, `created_at`, `updated_at` unless noted.

| Schema | Fields and validation |
|---|---|
| `Page<T>` | `items: T[]`, `next_cursor: string\|null`, `limit: int` |
| `User` | `email`, `email_verified: bool`, `role`, `status`, `timezone` (IANA), `locale` |
| `Profile` | `display_name*` 1–80, `institution_name` ≤120, `degree` ≤80, `major` ≤80, `year_of_study` 1–10, `graduation_year` 2000–2100, `onboarding_completed: bool` |
| `Settings` | `pomodoro: {focus_minutes 5–120, short_break_minutes 1–30, long_break_minutes 5–60, cycles_before_long_break 2–8}`, `week_starts_on 1–7`, `daily_study_target_minutes 0–960`, `quiet_hours: {start,end}\|null`, `ai_features_enabled`, `ai_personalization_enabled`, `theme` |
| `Goal` | `title*` 1–200, `category*`, `description` ≤2000, `target_date`, `priority` 1–4, `status` |
| `AvailabilityWindow` | `weekday* 1–7`, `start_time*`, `end_time*` (> start), `kind*` study\|blocked |
| `Term` | `name*` ≤80, `start_date*`, `end_date*` (> start) |
| `Subject` | `name*` 1–100, `term_id`, `code` ≤20, `color` `#RRGGBB`, `credits` 0–40, `instructor` ≤100, `archived: bool` |
| `TimetableEntry` | `title*` ≤100, `subject_id`, `kind*`, `weekday*`, `start_time*`, `end_time*`, `location` ≤100, `valid_from*`, `valid_until` |
| `Task` | `title*` 1–200, `description` ≤5000 (Markdown, sanitised on render), `status`, `priority` 1–4 (default 2), `due_at`, `estimated_minutes` 5–1440, `estimate_source`, `subject_id`, `project_id`, `hackathon_id`, `exam_id` (at most one of project/hackathon/exam), `source`, `is_blocked` (read-only), `subtask_counts {total, done}` (read-only), `completed_at`, `version` |
| `Subtask` | `title*` 1–200, `description` ≤2000, `status`, `estimated_minutes` 5–480, `position`, `due_at`, `ai_generated` (read-only), `depends_on: uuid[]` (same task, acyclic), `version` |
| `Roadmap` | `scope_type`, `scope_id`, `title`, `status`, `horizon_start`, `horizon_end`, `generated_by`, `explanation`, `steps: RoadmapStep[]`, `conflicts: Conflict[]`, `session_count` |
| `Conflict` | `code` (overload, deadline_infeasible, fixed_overlap, dependency_cycle, missing_estimate, past_due), `message`, `entity_refs`, `suggested_actions: string[]` |
| `PlannedSession` | `roadmap_id`, `task_id`, `subtask_id`, `exam_id`, `starts_at*`, `ends_at*` (> starts), `status`, `locked`, `reason_codes: string[]`, `explanation`, `version` |
| `Exam` | `title*` ≤120, `subject_id*`, `exam_type*`, `starts_at*`, `duration_minutes` 15–600, `location`, `weightage_percent` 0–100, `syllabus_notes` ≤20000 |
| `ExamTopic` | `title*` ≤120, `confidence` 1–5, `estimated_minutes` 5–1440, `status`, `position` |
| `Project` | `title*` ≤120, `description` ≤5000, `github_url` (must match `https://github.com/{owner}/{repo}`; never fetched server-side in MVP), `status`, `start_date`, `target_date`, `tech_stack` ≤20 items ≤40 chars, `subject_id` |
| `Hackathon` | `name*` ≤150, `organizer`, `url` (http/https), `mode`, `registration_deadline`, `starts_at`, `ends_at` (≥ starts), `submission_deadline`, `team_size` 1–20, `theme`, `problem_statement` ≤10000, `status`, `project_id`, `notes` |
| `TeamMember` | `name*` ≤80, `role` ≤60, `contact` ≤120 |
| `StudySession` | `task_id`, `subtask_id`, `subject_id`, `exam_id`, `planned_session_id`, `started_at*`, `ended_at`, `status`, `source`, `focus_seconds`, `rating` 1–5, `notes` ≤2000 |
| `PomodoroCycle` | `kind`, `planned_seconds`, `started_at`, `paused_total_seconds`, `ended_at`, `status`, derived `remaining_seconds` |
| `Recommendation` | `kind`, `title`, `body`, `reason`, `reason_codes`, `payload`, `score`, `status`, `expires_at` |
| `Resource` | `kind*`, `title*` ≤200, `url` (http/https), `file_id`, `subject_id`, `task_id`, `summary`, `body` ≤20000 |
| `FileMeta` | `original_filename`, `mime_type`, `size_bytes`, `status`, `purpose`, parent reference (exactly one) |
| `Notification` | `kind`, `title`, `body`, `payload`, `read_at` |
| `GoogleConnection` | `status`, `google_email`, `granted_scopes`, `capabilities {calendar, classroom, drive}`, `connected_at`, `last_error_code` |
| `AIRequest` | `feature`, `status`, `failure_reason`, `result` (feature-specific proposal, below), `review_status`, `usage {input_tokens, output_tokens}`, `completed_at` |
| `BreakdownProposal` | `subtasks: [{ref, title, description, estimated_minutes, depends_on: ref[]}]`, `assumptions: string[]`, `clarifying_questions: string[]` |

## 4. Endpoints

### 4.1 Authentication (`/auth`)

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| POST | `/auth/register` † | Create email account, send verification | P | `{email*, password*, display_name*, timezone}` → `201 User` | 409 `already_exists` is **not** revealed: always `202` with generic message; 422 weak password |
| POST | `/auth/login` | Email login | P | `{email*, password*}` → `200 {access_token, expires_in, user}` + refresh cookie | 401 `invalid_credentials`, 403 suspended, 429 |
| POST | `/auth/refresh` | Rotate refresh token, new access token | P (cookie + `X-Requested-With`) | none → `200 {access_token, expires_in}` | 401 `invalid_refresh` (reuse revokes family) |
| POST | `/auth/logout` | Revoke current session | U | none → `204` | none |
| POST | `/auth/logout-all` | Revoke all sessions | U | none → `204` | none |
| POST | `/auth/verify-email` | Consume verification token | P | `{token*}` → `204` | 400 `invalid_token` |
| POST | `/auth/verify-email/resend` | Resend verification | U | none → `202` | 429 |
| POST | `/auth/password/forgot` | Request reset (uniform response) | P | `{email*}` → `202` | 429 |
| POST | `/auth/password/reset` | Reset with token; revokes all sessions | P | `{token*, new_password*}` → `204` | 400 `invalid_token`, 422 |
| POST | `/auth/password/change` | Change password | U | `{current_password*, new_password*}` → `204` | 401 `invalid_credentials` |
| GET | `/auth/google/start` | Begin Google sign-in (redirect with PKCE/state/nonce) | P | query `redirect_to` (allowlisted path) → `302` | 400 |
| GET | `/auth/google/callback` | Complete Google sign-in | P | `code, state` → `302` to frontend + refresh cookie | 400 `invalid_state`, 403 `email_not_verified` |
| GET | `/auth/sessions` | List sessions | U | → `Page<Session{id, created_at, last_used_at, user_agent, current}>` | none |
| DELETE | `/auth/sessions/{id}` | Revoke a session | U Owner | → `204` | 404 |

**Example**

```http
POST /api/v1/auth/login
Content-Type: application/json

{ "email": "priya@example.edu", "password": "correct horse battery staple" }
```
```json
{ "access_token": "eyJhbGciOi...", "expires_in": 900,
  "user": { "id": "0192f2a0-...", "email": "priya@example.edu", "email_verified": true,
            "role": "student", "status": "active", "timezone": "Asia/Kolkata", "locale": "en" } }
```

### 4.2 Users and Privacy (`/users/me`)

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| GET | `/users/me` | Current user + profile | U | → `User` + `profile` | none |
| PATCH | `/users/me` | Update timezone, locale, profile | U | partial `User`/`Profile` → `User` | 422 invalid timezone |
| GET / PATCH | `/users/me/settings` | Read/update `Settings` | U | → `Settings` | 422 |
| GET / PUT | `/users/me/consents` | Read latest / record consent decisions | U | `{consents:[{type, version, granted}]}` → `200` | 422 |
| POST | `/users/me/export` † ⏳ | Request data export | U | none → `202 {id, status}` | 409 export already running |
| GET | `/users/me/export/{id}` | Export status and short-lived download URL | U Owner | → `{status, download_url?, expires_at?}` | 404 |
| DELETE | `/users/me` | Request account deletion (30-day grace) | U | `{password?, confirm: "DELETE"}` → `202` | 401 reauth required |
| POST | `/users/me/deletion/cancel` | Cancel pending deletion | U | none → `204` | 409 not pending |

### 4.3 Onboarding (`/onboarding`, `/interests`, `/goals`, `/availability`)

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| GET | `/onboarding` | Step completion state | U | → `{steps:{profile,interests,goals,availability}, completed}` | none |
| PUT | `/onboarding/profile` | Save profile and timezone | U | `Profile + timezone` → `Profile` | 422 |
| PUT | `/onboarding/interests` | Replace interests | U | `{interest_ids: uuid[] ≤30}` → `204` | 422 unknown id |
| PUT | `/onboarding/availability` | Replace weekly windows | U | `{windows: AvailabilityWindow[] ≤50}` → `200` | 422 overlap, end ≤ start |
| POST | `/onboarding/complete` | Mark complete (requires profile + availability) | U | none → `200 Profile` | 409 `incomplete_onboarding` |
| GET | `/interests` | Interest catalogue | U | → `Page<Interest>` | none |
| GET / POST | `/goals` | List / create goals | U | `Goal` → `Page<Goal>` / `201 Goal` | none |
| GET / PATCH / DELETE | `/goals/{id}` | Read / update / soft-delete | U Owner | `Goal` → `Goal` / `204` | 404 |

### 4.4 Subjects, Terms, Timetable

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| GET / POST | `/terms` | List / create terms | U | `Term` → `Page<Term>` / `201` | 422 |
| GET / PATCH / DELETE | `/terms/{id}` | Term CRUD | U Owner | `Term` → `Term` / `204` | 404 |
| GET | `/subjects` | List (`?term_id&archived`) | U | → `Page<Subject>` | none |
| POST | `/subjects` † | Create | U | `Subject` → `201 Subject` | 409 duplicate name in term |
| GET / PATCH | `/subjects/{id}` | Read / update | U Owner | `Subject` → `Subject` | 404, 409 |
| DELETE | `/subjects/{id}` | Soft-delete with children | U Owner | → `204` | 404 |
| GET | `/timetable` | Entries (`?weekday&subject_id`) | U | → `Page<TimetableEntry>` | none |
| POST | `/timetable` | Create | U | `TimetableEntry` → `201` | 422 end ≤ start |
| PATCH / DELETE | `/timetable/{id}` | Update / soft-delete | U Owner | `TimetableEntry` → `TimetableEntry` / `204` | 404 |

### 4.5 Tasks and Subtasks

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| GET | `/tasks` | List; filters `status, subject_id, project_id, hackathon_id, exam_id, due_before, due_after, is_blocked, q`; sort `due_at, priority, created_at` | U | → `Page<Task>` | 400 bad filter |
| POST | `/tasks` † | Create | U | `Task` → `201 Task` | 404 referenced parent not owned, 422 two parents |
| GET | `/tasks/{id}` | Read with subtasks and dependencies | U Owner | → `Task` + `subtasks` + `depends_on` | 404 |
| PATCH | `/tasks/{id}` | Update | U Owner | `Task + version*` → `Task` | 404, 409 `version_conflict` |
| DELETE | `/tasks/{id}` | Soft-delete (cascades to subtasks) | U Owner | → `204` | 404 |
| POST | `/tasks/{id}/complete` | Mark done (sets `completed_at`) | U Owner | none → `Task` | 409 blocked unless `force=true` |
| POST | `/tasks/{id}/reopen` | Reopen | U Owner | none → `Task` | 409 not done |
| PUT | `/tasks/{id}/dependencies` | Replace task-level dependencies | U Owner | `{depends_on: uuid[] ≤20}` → `200` | 409 `dependency_cycle`, 404 |
| GET | `/tasks/{task_id}/subtasks` | List | U Owner | → `Page<Subtask>` | 404 |
| POST | `/tasks/{task_id}/subtasks` † | Create | U Owner | `Subtask` → `201` | 422 |
| PATCH / DELETE | `/tasks/{task_id}/subtasks/{id}` | Update / soft-delete | U Owner | `Subtask + version*` → `Subtask` / `204` | 404, 409 |
| PUT | `/tasks/{task_id}/subtasks/order` | Reorder | U Owner | `{ids: uuid[]}` (must be exact set) → `200` | 422 |
| PUT | `/tasks/{task_id}/subtasks/{id}/dependencies` | Replace subtask dependencies | U Owner | `{depends_on: uuid[]}` → `200` | 409 `dependency_cycle` |

**Example**

```http
POST /api/v1/tasks
Idempotency-Key: 7a1f3c0e-5d0f-4a5e-9f4a-0d3b2a9c1e11

{ "title": "Operating Systems assignment 3", "subject_id": "0192f2a1-...",
  "due_at": "2026-10-14T18:29:00Z", "priority": 3, "estimated_minutes": 240 }
```
```json
{ "id": "0192f2b4-...", "title": "Operating Systems assignment 3", "status": "todo",
  "priority": 3, "due_at": "2026-10-14T18:29:00Z", "estimated_minutes": 240,
  "estimate_source": "user", "subject_id": "0192f2a1-...", "is_blocked": false,
  "subtask_counts": { "total": 0, "done": 0 }, "version": 1,
  "created_at": "2026-10-06T09:12:03Z", "updated_at": "2026-10-06T09:12:03Z" }
```

### 4.6 Planner and Calendar

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| GET | `/planner/now` | Next best action(s) with reason codes and explanation | U | `?limit=3` → `{items:[{task_ref, planned_session?, score, reason_codes, explanation}]}` | none (empty list if nothing actionable) |
| POST | `/planner/roadmaps` † ⏳ | Generate roadmap for a scope | U | `{scope_type*, scope_id*, horizon_end?}` → `202 AIRequest/Job` | 404 scope, 429 `ai_quota_exceeded`, 409 roadmap generation in progress |
| GET | `/planner/roadmaps` | List (`?status&scope_type`) | U | → `Page<Roadmap>` | none |
| GET | `/planner/roadmaps/{id}` | Read with steps, conflicts | U Owner | → `Roadmap` | 404 |
| POST | `/planner/roadmaps/{id}/accept` | Activate draft; creates planned sessions | U Owner | none → `200 Roadmap` | 409 not draft or stale inputs |
| POST | `/planner/roadmaps/{id}/regenerate` ⏳ | New draft from current data | U Owner | none → `202` | 429 |
| DELETE | `/planner/roadmaps/{id}` | Archive | U Owner | → `204` | 404 |
| GET | `/planner/sessions` | Planned sessions (`?from&to&status`, window ≤ 62 days) | U | → `Page<PlannedSession>` | 400 window too large |
| PATCH | `/planner/sessions/{id}` | Move, lock/unlock, skip | U Owner | `{starts_at?, ends_at?, locked?, status?, version*}` → `PlannedSession` | 409 `session_overlap`, `version_conflict` |
| POST | `/planner/replan` ⏳ | Re-run scheduling for unlocked future sessions | U | `{from?}` → `202` | 429 |
| GET | `/planner/availability` | Computed free slots (debug/preview) | U | `?from&to` → `{slots:[{start,end}]}` | 400 |
| GET | `/planner/conflicts` | Current conflicts | U | → `Conflict[]` | none |
| GET | `/calendar/events` | Unified read view of timetable, planned sessions, exams, deadlines | U | `?from&to` (≤ 62 days) → `{events:[{type, id, title, start, end, source_ref}]}` | 400 |

**Example**

```http
POST /api/v1/planner/roadmaps
{ "scope_type": "exam", "scope_id": "0192f2c0-..." }
```
```http
HTTP/1.1 202 Accepted
Location: /api/v1/ai/requests/0192f2d1-...
```
```json
{ "id": "0192f2d1-...", "feature": "roadmap", "status": "queued", "review_status": null }
```

### 4.7 Exams

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| GET / POST | `/exams` | List (`?subject_id&upcoming`) / create | U | `Exam` → `Page<Exam>` / `201` | 404 subject |
| GET / PATCH / DELETE | `/exams/{id}` | CRUD | U Owner | `Exam` → `Exam` / `204` | 404 |
| GET / POST | `/exams/{id}/topics` | List / create | U Owner | `ExamTopic` → `Page` / `201` | 422 |
| PATCH / DELETE | `/exams/{id}/topics/{topic_id}` | Update (confidence, status) / delete | U Owner | `ExamTopic` → `ExamTopic` / `204` | 404 |
| POST | `/exams/{id}/topics/extract` ⏳ | AI topic extraction from `syllabus_notes` | U Owner | none → `202 AIRequest` | 422 empty syllabus, 429 |
| POST | `/exams/{id}/roadmap` ⏳ | Generate exam roadmap | U Owner | none → `202` | 409 exam in past, 422 no topics |

### 4.8 Projects

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| GET / POST | `/projects` | List / create | U | `Project` → `Page` / `201` | 422 invalid GitHub URL |
| GET / PATCH / DELETE | `/projects/{id}` | CRUD | U Owner | `Project` → `Project` / `204` | 404 |
| GET | `/projects/{id}/tasks` | Project tasks (alias of `/tasks?project_id=`) | U Owner | → `Page<Task>` | 404 |
| POST | `/projects/{id}/breakdown` ⏳ | AI project task breakdown proposal | U Owner | `{goal_text?}` → `202 AIRequest` | 429 |

### 4.9 Hackathons

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| GET / POST | `/hackathons` | List (`?status`) / create | U | `Hackathon` → `Page` / `201` | 422 |
| GET / PATCH / DELETE | `/hackathons/{id}` | CRUD | U Owner | `Hackathon` → `Hackathon` / `204` | 404 |
| GET / POST | `/hackathons/{id}/team` | List / add member | U Owner | `TeamMember` → `Page` / `201` | 422 |
| PATCH / DELETE | `/hackathons/{id}/team/{member_id}` | Update / remove | U Owner | `TeamMember` → `TeamMember` / `204` | 404 |
| POST | `/hackathons/{id}/plan` ⏳ | AI milestone plan backward from submission deadline | U Owner | none → `202 AIRequest` | 422 no submission deadline, 429 |

### 4.10 Pomodoro and Study Sessions

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| GET | `/pomodoro/active` | Active session with current cycle | U | → `{session, cycle}` or `204` | none |
| POST | `/pomodoro/sessions` † | Start session and first focus cycle | U | `{task_id?, subtask_id?, subject_id?, exam_id?, planned_session_id?}` → `201 {session, cycle}` | 409 `session_active`, 404 |
| POST | `/pomodoro/sessions/{id}/pause` | Pause current cycle | U Owner | none → `{session, cycle}` | 409 not running |
| POST | `/pomodoro/sessions/{id}/resume` | Resume | U Owner | none → `{session, cycle}` | 409 not paused |
| POST | `/pomodoro/sessions/{id}/advance` | Complete or skip current cycle and start the next per settings | U Owner | `{action*: complete\|skip}` → `{session, cycle}` | 409 invalid transition |
| POST | `/pomodoro/sessions/{id}/finish` | End session | U Owner | `{outcome*: completed\|abandoned, rating?, notes?}` → `StudySession` | 409 already finished |
| GET / PUT | `/pomodoro/settings` | Read/update `Settings.pomodoro` | U | → `Settings.pomodoro` | 422 |
| GET | `/study-sessions` | List (`?from&to&subject_id&task_id`) | U | → `Page<StudySession>` | none |
| POST | `/study-sessions` † | Manual log | U | `StudySession` → `201` | 422 future or overlapping times |
| GET / PATCH / DELETE | `/study-sessions/{id}` | CRUD | U Owner | `StudySession` → `StudySession` / `204` | 404 |

**Example**

```http
POST /api/v1/pomodoro/sessions
{ "task_id": "0192f2b4-...", "planned_session_id": "0192f2e0-..." }
```
```json
{ "session": { "id": "0192f2f1-...", "status": "active", "source": "pomodoro", "task_id": "0192f2b4-..." },
  "cycle": { "id": "0192f2f2-...", "kind": "focus", "planned_seconds": 1500,
             "started_at": "2026-10-06T10:00:00Z", "status": "running", "remaining_seconds": 1500 } }
```

### 4.11 Productivity Analytics

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| GET | `/productivity/summary` | Period totals: focus time, tasks done, plan adherence | U | `?from&to` → `{focus_seconds, sessions, tasks_completed, adherence_ratio, deltas}` | 400 |
| GET | `/productivity/daily` | Daily series (≤ 366 days) | U | `?from&to` → `{days:[…]}` | 400 |
| GET | `/productivity/subjects` | Time per subject | U | `?from&to` → `{subjects:[…]}` | 400 |
| GET | `/productivity/estimates` | Estimate accuracy (actual ÷ estimated) | U | `?from&to` → `{ratio_median, samples, by_subject}` | 400 |

### 4.12 Recommendations

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| GET | `/recommendations` | Active recommendations | U | `?status&kind` → `Page<Recommendation>` | none |
| POST | `/recommendations/{id}/feedback` | Accept or dismiss | U Owner | `{action*: accept\|dismiss, reason?}` → `Recommendation` | 404, 409 expired |
| POST | `/recommendations/refresh` ⏳ | Force refresh (rate limited) | U | none → `202` | 429 |

### 4.13 Resources and Files

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| GET / POST | `/resources` | List (`?subject_id&kind&q`) / create | U | `Resource` → `Page` / `201` | 422 |
| GET / PATCH / DELETE | `/resources/{id}` | CRUD | U Owner | `Resource` → `Resource` / `204` | 404 |
| GET | `/resources/search` | Semantic search | U | `?q*` (≤ 300 chars), `limit ≤ 20` → `{items:[{resource, score}]}` | 429, 503 |
| POST | `/files` † | Upload (multipart: `file`, `purpose`, one parent id) | U | → `201 FileMeta` (`status` may be `pending_scan`) | 413, 415, 404 parent, 422 quota |
| GET | `/files` | List (`?parent…`) | U | → `Page<FileMeta>` | none |
| GET | `/files/{id}` | Metadata | U Owner | → `FileMeta` | 404 |
| GET | `/files/{id}/download-url` | Signed URL (5 min) | U Owner | → `{url, expires_at}` | 404, 409 not `available` |
| DELETE | `/files/{id}` | Soft-delete | U Owner | → `204` | 404 |

### 4.14 YouTube Assistant (post-MVP)

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| POST | `/youtube/analyze` † ⏳ | Analyse a video URL | U | `{url*}` (validated YouTube URL) → `202 AIRequest` | 422 invalid URL, 422 no transcript available, 429 |
| GET | `/youtube/analyses/{id}` | Read analysis | U Owner | → `{summary, key_points, chapters, resource_id}` | 404 |
| POST | `/youtube/analyses/{id}/save` | Save as resource | U Owner | `{subject_id?}` → `201 Resource` | 404 |

### 4.15 Notifications

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| GET | `/notifications` | List (`?unread`) | U | → `Page<Notification>` + `unread_count` | none |
| POST | `/notifications/{id}/read` | Mark read | U Owner | none → `204` | 404 |
| POST | `/notifications/read-all` | Mark all read | U | none → `204` | none |
| GET / PUT | `/notifications/preferences` | Per kind/channel toggles | U | `{items:[{kind, channel, enabled}]}` → same | 422 |

### 4.16 Gamification (post-MVP)

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| GET | `/gamification/profile` | XP total, level, streak | U | → `{xp, level, streak:{current, longest}}` | none |
| GET | `/gamification/achievements` | All with earned state | U | → `Page<Achievement>` | none |
| GET | `/gamification/xp-events` | History | U | → `Page<XpEvent>` | none |

### 4.17 Google Integrations (`/integrations/google`)

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| GET | `/integrations/google` | Connection status and capabilities | U | → `GoogleConnection` or `{status: "not_connected"}` | none |
| POST | `/integrations/google/connect` | Start incremental authorization for a capability | U | `{capability*: calendar\|classroom\|drive}` → `{authorization_url}` | 409 already granted |
| GET | `/integrations/google/callback` | OAuth callback | P (state-bound) | `code, state` → `302` | 400 `invalid_state`, 403 scope denied |
| DELETE | `/integrations/google` | Disconnect, revoke tokens, stop syncs | U | `{capability?}` → `204` | none |
| GET | `/integrations/google/calendar/settings` | Target calendar and sync options | U | → `{calendar_id, sync_sessions, sync_exams, sync_deadlines, use_freebusy}` | 409 not connected |
| PUT | `/integrations/google/calendar/settings` | Update | U | same → same | 409, 422 |
| POST | `/integrations/google/calendar/sync` ⏳ | Manual sync | U | none → `202 {sync_run_id}` | 409 not connected or running, 429 |
| GET | `/integrations/google/classroom/courses` | Imported courses and mappings | U | → `Page<Course{id, name, subject_id, sync_enabled}>` | 409 not connected |
| PUT | `/integrations/google/classroom/courses/{id}` | Map to subject, enable sync | U Owner | `{subject_id?, sync_enabled}` → `Course` | 404 |
| POST | `/integrations/google/classroom/sync` ⏳ | Manual sync | U | none → `202` | 409, 429 |
| GET | `/integrations/google/sync-runs` | History | U | `?provider` → `Page<SyncRun>` | none |

### 4.18 AI (`/ai`)

| Method | Path | Purpose | Auth | Request → Response | Errors |
|---|---|---|---|---|---|
| POST | `/ai/task-breakdown` † ⏳ | Propose subtasks for a task | U | `{task_id*, hints?: string ≤1000}` → `202 AIRequest` | 404 task, 403 `ai_disabled`, 429 `ai_quota_exceeded` |
| GET | `/ai/requests/{id}` | Poll status/result | U Owner | → `AIRequest` | 404 |
| POST | `/ai/requests/{id}/accept` | Accept (optionally edited) proposal; persists entities | U Owner | `{result_override?}` (re-validated) → `200 {created:[{type,id}]}` | 409 not pending, 410 expired, 422 edited proposal invalid |
| POST | `/ai/requests/{id}/reject` | Reject | U Owner | `{reason?}` → `204` | 409 not pending |
| GET | `/ai/usage` | Own quota and usage | U | → `{daily_limit, used_today, resets_at}` | none |

Feature endpoints that trigger AI elsewhere (`/planner/roadmaps`, `/exams/{id}/roadmap`, `/exams/{id}/topics/extract`, `/projects/{id}/breakdown`, `/hackathons/{id}/plan`, `/youtube/analyze`) also return `AIRequest` resources polled via `/ai/requests/{id}`.

**Example**

```http
POST /api/v1/ai/task-breakdown
{ "task_id": "0192f2b4-...", "hints": "I have not studied scheduling algorithms yet" }
```
```json
{ "id": "0192f301-...", "feature": "task_breakdown", "status": "queued" }
```
```http
GET /api/v1/ai/requests/0192f301-...
```
```json
{ "id": "0192f301-...", "feature": "task_breakdown", "status": "succeeded", "review_status": "pending",
  "result": { "subtasks": [
      { "ref": "S1", "title": "Review CPU scheduling algorithms", "description": "FCFS, SJF, RR, priority.", "estimated_minutes": 45, "depends_on": [] },
      { "ref": "S2", "title": "Implement round-robin simulator", "description": "Quantum configurable.", "estimated_minutes": 90, "depends_on": ["S1"] } ],
    "assumptions": ["Language is Python."], "clarifying_questions": [] },
  "usage": { "input_tokens": 910, "output_tokens": 420 } }
```

### 4.19 Admin (`/admin`)

All **A**: admin role, MFA (Phase 19), every call audited. Admin endpoints return aggregates and metadata only; no endpoint exposes tasks, notes, files, AI prompts or analytics of an individual student.

| Method | Path | Purpose | Request → Response | Errors |
|---|---|---|---|---|
| GET | `/admin/metrics` | Aggregate usage (signups, WAU, sessions, AI cost, sync health) | `?from&to` → `{series…}` | 400 |
| GET | `/admin/users` | Account metadata (id, masked email, status, role, created_at, last_login_at) | `?q&status` → `Page<AdminUser>` | 400 |
| PATCH | `/admin/users/{id}/status` | Suspend/reactivate | `{status*, reason*}` → `AdminUser` | 404, 409 self-suspend |
| GET | `/admin/jobs` | Queue depth, failed jobs | `?queue&status` → `Page<Job>` | none |
| POST | `/admin/jobs/{id}/retry` | Retry failed job | none → `202` | 404, 409 |
| GET | `/admin/ai/usage` | Aggregate AI cost and error rates | `?from&to` → `{by_feature, by_day}` | 400 |
| GET | `/admin/audit-logs` | Audit trail | `?actor&action&from&to` → `Page<AuditLog>` | 400 |
| POST | `/admin/access-grants` | Create break-glass grant (requires recorded user consent) | `{target_user_id*, reason*, user_consent_at*, expires_at*}` → `201` | 404, 422 consent missing or expiry > 7 days |
| DELETE | `/admin/access-grants/{id}` | Revoke grant | none → `204` | 404 |
| GET / PUT | `/admin/feature-flags` | Read/update flags | `{key, enabled, rollout_percent}` → same | 422 |

### 4.20 Webhooks and Health

| Method | Path | Purpose | Auth | Notes |
|---|---|---|---|---|
| GET | `/healthz` | Liveness | P | no dependencies checked |
| GET | `/readyz` | Readiness (DB, migrations at head) | P | 503 when not ready |
| POST | `/api/v1/webhooks/google/calendar` | Calendar push notifications (post-MVP) | token header verification | Verify channel token; confirm behaviour against Google docs before implementation |

## 5. Validation Summary

Pydantic models enforce types, lengths, ranges and enums above; services enforce cross-field and ownership rules (parent exists and is owned, dependency acyclicity, time ordering, quotas). Text is stored raw and escaped/sanitised at render. URLs accept only http/https and are never fetched server-side without the SSRF-safe fetcher ([Security](17-security.md#5-ssrf-and-outbound-requests)).

## 6. Failure Cases

| Situation | Result |
|---|---|
| Ownership mismatch on any `{id}` | 404 `not_found` |
| Duplicate POST with same idempotency key | Stored response replayed |
| AI provider down | AIRequest `failed` with `failure_reason`; HTTP 200 on poll; 503 only when the call itself cannot be queued |
| Google token revoked | 409 `integration_needs_reauth` on Google endpoints; connection `needs_reauth` |
| Stale roadmap accept | 409 `stale_inputs` with a hint to regenerate |
