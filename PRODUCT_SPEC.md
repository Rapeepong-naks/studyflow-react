# StudyFlow — Product requirements

**Status:** Planned. The repository currently contains a Vite + React starter, not the finished app.
**Goal:** Build a personal study planner for one person to organize courses, track tasks, and see what needs attention next.

This document defines the target app for contributors. Learning notes and course materials are kept locally and are not part of this repository.

## 1. Scope

The first release is a single-user web app that runs locally. A `json-server` REST API stores data in `db.json`, so tasks and courses survive a browser refresh or restart. Authentication, cloud sync, multi-user access, and public hosting are outside the first release.

The main user journey is: create a course → add a task with a due date and priority → find it in a filtered list → mark it complete → see the dashboard update → reopen the app and find the data intact.

## 2. Domain model

```ts
type Priority = 'low' | 'medium' | 'high'

type Course = {
  id: string
  name: string
  color: string // six-digit hex, for example #5b5bd6
}

type StudyTask = {
  id: string
  courseId: string
  title: string
  description: string
  dueDate: string | null // local calendar date: YYYY-MM-DD
  priority: Priority
  done: boolean
}
```

One task belongs to exactly one existing course. Course names and task titles cannot be blank after trimming whitespace. `priority` is restricted to the three listed values. `dueDate` may be absent. IDs are strings to match the selected `json-server` API.

**Open task:** `done === false`.

**Due soon:** open task due from today through the next seven local calendar days, inclusive.

**Overdue:** open task with a due date before today.
Completed tasks are excluded from due-soon and overdue counts. Date comparisons must use local calendar dates consistently; avoid mixing UTC timestamps with date-only form values.

Deleting a course with tasks is blocked until those tasks are moved or deleted. Deleting a task requires a confirmation step; canceling leaves it unchanged.

## 3. Pages and navigation

| Route | Required behavior |
| --- | --- |
| `/` | Dashboard with total, open, completed, due-soon, and overdue counts; next tasks; Focus Timer |
| `/tasks` | Task list with search, filters, sort, empty state, and links to details |
| `/tasks/new` | Create task form with validation |
| `/tasks/:taskId` | Task details, complete/reopen action, edit link, confirmed delete |
| `/tasks/:taskId/edit` | Edit form prefilled with current values |
| `/courses` | List, create, edit, and delete courses |

Direct navigation and refresh must work for every route. An unknown task ID displays a not-found message and a way back to the list instead of crashing. Browser Back/Forward should behave normally.

List view state lives in query parameters so a copied URL recreates the same view. Supported parameters:

- `q`: case-insensitive title search; trim leading/trailing whitespace
- `status`: `all` (default), `open`, `done`, or `overdue`
- `courseId`: existing course ID or no course filter
- `priority`: `low`, `medium`, `high`, or no priority filter
- `sort`: `dueDate` (default, earliest first, no date last) or `priority` (high first)

Invalid or unknown parameter values fall back to defaults without breaking the page. Changing one filter preserves the other supported parameters; clearing filters removes them.

## 4. Functional requirements

| ID | Requirement | Acceptance check |
| --- | --- | --- |
| FR-01 | Create, edit, and delete courses | Valid names/colors persist; deletion is blocked when tasks reference the course |
| FR-02 | Create tasks | Required values are validated; successful save opens the new task |
| FR-03 | Edit tasks | Current values appear in the form; unchanged fields remain intact after save |
| FR-04 | Complete and reopen tasks | Status, list, and dashboard counts update consistently |
| FR-05 | Delete tasks | Confirmation can be canceled; confirmed deletion persists after refresh |
| FR-06 | Search, filter, and sort | Results match the selected values; no matches show a clear empty state |
| FR-07 | Shareable list URL | Copying, reopening, or refreshing a filtered URL retains the view |
| FR-08 | Dashboard | Counts derive from current task data; due-soon/overdue rules match Section 2 |
| FR-09 | Persistence | Courses/tasks survive refresh and browser restart via the local API |
| FR-10 | Request feedback | Loading, error, empty, and pending states are distinct; failed actions can be retried |
| FR-11 | Focus Timer | Start, pause, and reset work across page navigation; only one interval runs |
| FR-12 | Validation | Errors identify the field and problem; invalid submissions do not write data |

## 5. Data access and API

Use `json-server` locally with `db.example.json` as the template for ignored `db.json`. Read the base URL from `VITE_API_URL` in `.env.local`, using `.env.example` as the template. Client-exposed `VITE_` variables must not contain secrets.

Expected resources are `/courses` and `/tasks`, each supporting `GET`, `POST`, `PATCH`, and `DELETE` as needed. UI components should call a small API module rather than duplicate request construction. Check HTTP response status; show useful feedback for network errors, 404, and unsuccessful writes. Prevent duplicate form submissions while a write is pending.

The app should progressively use React Router Data Mode loaders and actions for route data and writes. Task/course data comes from the API; query parameters own list filters; component state owns form input; shared UI state such as the Focus Timer may use reducer and context. Avoid independent copies of the same server data in multiple stores.

## 6. Quality requirements

| ID | Requirement | Verification |
| --- | --- | --- |
| QR-01 | Responsive layout | Main tasks work at roughly 375px width and desktop without horizontal overflow |
| QR-02 | Keyboard access | Main actions work with Tab, Shift+Tab, Enter, and Space; focus is visible |
| QR-03 | Accessible forms | Inputs have labels; errors are associated with fields; buttons state their action |
| QR-04 | Type safety | TypeScript strict mode passes without using `any` to hide errors |
| QR-05 | Maintainability | Components, API access, and shared UI state have clear responsibilities |
| QR-06 | Build health | Typecheck, lint, build, and preview pass when their scripts are installed |
| QR-07 | Local data hygiene | `db.json`, `.env.local`, build output, and dependencies remain out of Git |

## 7. Acceptance scenarios

1. Create a course and task, reload the page, and confirm both remain.
2. Edit a task's title, priority, and due date; reopen its direct URL and confirm the update.
3. Filter open tasks by course, copy the URL into a new tab, and get the same result.
4. Complete a task and confirm open/completed/dashboard counts change correctly.
5. Cancel deletion once and confirm the task remains; confirm deletion and verify it stays deleted after reload.
6. Attempt to delete a course with tasks and get a clear explanation without losing data.
7. Open an unknown task URL and get a usable not-found screen.
8. Stop the API, observe a recoverable error state, restart it, and retry successfully.
9. Start the Focus Timer, navigate between pages, pause/reset it, and verify it never counts twice as fast.

## 8. Implementation status and next work

The current `src/` is the Vite starter. Build the first static task list, migrate to TypeScript, then add state/forms, routing, local API integration, and Data Mode. `db.example.json` and `.env.example` are committed templates; create ignored local copies when integrating the API. Update this spec if product behavior changes, and keep the README's status accurate.
