---
plan_id: 001-persistent-tasks-and-web-task-board
spec: 001-persistent-tasks-and-web-task-board
feature: Tasks stored in Postgres by the Task API, and a task board on the web site's start page
gitops_app: ika100-demo/taskboard20-gitops
status: completed
repos:
- id: taskboard20-gitops
  shape: gitops-app
  summary: Declare the Postgres database, wire the Task API to it with two dev replicas, and wire the
    web site to the Task API
  acs:
  - AC-001.1
  - AC-001.2
  - AC-001.3
  - AC-001.4
  - AC-001.5
  - AC-001.6
  - AC-001.7
  order: before-merge
  gitops:
  - resource: postgres
  - uses: postgres
    component: taskboard20-api
  - replicas: 2
    component: taskboard20-api
    instance: dev
  - env:
      TASK_API_URL: http://taskboard20-api
    component: taskboard20-web
  depends_on: []
  done: true
- id: taskboard20-api
  shape: service-python
  summary: Persistent tasks in Postgres with 503 and readiness on a database outage
  acs:
  - AC-001.1
  - AC-001.2
  criteria:
  - 'Given a task created with `POST /tasks` `{"title": "Buy milk", "due_date": "2026-12-01"}`, when the
    service is restarted against the same database (`DATABASE_URL`), then `GET /tasks` answers `200` and
    lists that task with the same `id`, `title` and `due_date` as returned by the `201` at creation (system
    AC-001.1) (replaces AC-001.7)'
  - Given two service instances using the same database, when tasks A, B and C are created in that order
    alternately through either instance, then `GET /tasks` on each instance lists A, B, C in creation
    order with exactly the fields `id`, `title`, `due_date`, and with `due_date` `null` for a task created
    without one (system AC-001.1)
  - Given a running service whose database is reachable, when a client sends `POST /tasks` with a `title`
    or `due_date` the contract rejects, then the response is still `422` with `loc` ending in that field,
    and no row is stored, so a following `GET /tasks` (also after a restart) does not contain it (system
    AC-001.1)
  - 'Given the database is unreachable (also when it already is at service start), when a client sends
    `POST /tasks` or `GET /tasks`, then within 6 seconds the response is `503` with the JSON body `{"detail":
    "Task store unavailable"}`, and no task is created (system AC-001.2)'
  - Given the database is unreachable, when the probes are called, then `GET /ready` answers `503` while
    `GET /health` answers `200` (system AC-001.2)
  - Given the database was unreachable and becomes reachable again, when a client then sends `POST /tasks`
    and `GET /tasks` to the same running process, then they answer `201` and `200` and `GET /ready` answers
    `200`, with no restart of the service (system AC-001.2)
  arguments: |
    Storage replaces the in-memory TaskStore behind the same models and router; the HTTP contract of the repo's spec 001
    (fields, validation, 422 body) stays unchanged, only its AC-001.7 is superseded. The service creates its table
    itself at start, idempotently (or with a migration tool run at start); with two replicas both may start at once.
    Creation order needs an ordering column filled by the database (e.g. a bigserial or insertion timestamp plus
    sequence), because `id` stays a UUID string. Connection config comes only from `DATABASE_URL` / `PG*` that the
    gitops entry injects; tests run against a real Postgres (e.g. a container) or an equivalent test double for the
    outage cases. Smoke tests must not assume an empty list and must tolerate two replicas behind the Service.
  depends_on: []
  done: true
- id: taskboard20-web
  shape: web-nextjs
  summary: Task board on the start page that lists and adds tasks through the Task API
  acs:
  - AC-001.3
  - AC-001.4
  - AC-001.5
  - AC-001.6
  - AC-001.7
  criteria:
  - Given the Task API's `GET /tasks` answers `200` with tasks A, B and C in that order, when a person
    opens the start page `/`, then it lists A, B, C in that order, each with its `title` exactly as returned
    and its `due_date` as `YYYY-MM-DD`, or the text `No due date` when `due_date` is `null` (system AC-001.3)
  - Given the Task API's `GET /tasks` answers `200` with `[]`, when a person opens `/`, then the page
    shows the text `No tasks yet` and the create form with a title field and a date-picker due-date field
    (system AC-001.4)
  - 'Given `/` is open, when a person enters the title `Call Bob`, optionally picks a due date, and submits,
    then the web site sends `POST {TASK_API_URL}/tasks` with `{"title": "Call Bob", "due_date": "<YYYY-MM-DD>"}`
    or `"due_date": null`, and on `201` the returned task appears at the end of the list without a page
    reload and the form is emptied (system AC-001.5)'
  - Given `/` is open, when a person submits an empty or whitespace-only title, or a title longer than
    200 characters, then `Title is required` or `Title must be at most 200 characters` respectively is
    shown next to the title field, the typed input is kept, and no `POST /tasks` is sent (system AC-001.6)
  - Given the Task API answers a `POST /tasks` with `422` whose `detail[].loc` ends in `title`, when a
    person submits, then the page shows `Title must be at most 200 characters` if the title is longer
    than 200 characters and `Title is required` otherwise, next to the title field, and keeps the typed
    input (system AC-001.6)
  - Given the Task API is unreachable, does not answer `GET /tasks` within 3 seconds, or answers anything
    but `200` with a JSON array, when a person opens `/`, then the page loads with `200` and shows `Tasks
    could not be loaded. Try again later.` instead of the list, with the create form still shown (system
    AC-001.7)
  - Given the Task API is unreachable, does not answer `POST /tasks` within 5 seconds, or answers anything
    but `201` or a `422` naming `title` (for example `503`), when a person submits a valid task, then
    the page shows `Task could not be saved. Try again.` and keeps the typed title and due date (system
    AC-001.7)
  arguments: |
    The browser never calls the Task API: the page reads tasks on the server at request time (no build-time
    prerendering, so the list is current on every load) and the form submits to the web site's own server
    (a Server Action or a route handler), which calls the Task API. `TASK_API_URL` (no `NEXT_PUBLIC_` prefix) is read
    at runtime and documented in docs/env-vars.md. `/api/ready` must not depend on the Task API. Tests stub the Task
    API at the HTTP boundary (`TASK_API_URL` pointing at a fake); the dev smoke test adds a `smoke-<uuid>` task
    (`@smoke @writes`) and a read-only one checks that `/` loads.
  depends_on: []
  done: true
gitops_pin: []
---

## Decomposition

Three entries, one level: all three have `depends_on: []`, and the gitops entry is `order: before-merge`, so `/system:ship` applies it before the components merge.

- Level 1 (in parallel): `taskboard20-gitops` (applied before the components merge), `taskboard20-api`, `taskboard20-web`.

The API and the web site build in parallel against the contract below. The API's HTTP contract does not change (spec A1); only its storage does. The web site starts with a stub of that contract, so it does not need the new API build.

## taskboard20-gitops — Declare the Postgres database, wire the Task API to it with two dev replicas, and wire the web site to the Task API

**Shape:** `gitops-app` · **Criteria:** AC-001.1 to AC-001.7 · **Depends on:** []

`system.yaml` has no resources yet. This entry does four things:

- `resource: postgres` declares the system's single database (A9). It runs in dev only, because the components run only in dev.
- `uses: postgres` on `taskboard20-api` injects `DATABASE_URL` and `PG*` from the CloudNativePG secret (AC-001.1, AC-001.2).
- `replicas: 2` for `taskboard20-api` in dev makes the "two or more replicas" case of AC-001.1 the normal dev state, so the dev smoke run and Argo actually test it.
- `env: TASK_API_URL=http://taskboard20-api` on `taskboard20-web` lets the web site reach the API inside the namespace (Service port 80). The board criteria AC-001.3 to AC-001.7 need this wiring.

The web site's exposure already exists. The API keeps its existing exposure because its smoke tests use it. Removing it is no operation of this plan, and the spec does not ask for it.

## taskboard20-api — Persistent tasks in Postgres with 503 and readiness on a database outage

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2 · **Depends on:** []

The API swaps its in-memory `TaskStore` for Postgres behind the unchanged `POST /tasks` / `GET /tasks` contract. It owns three parts:

- durability and shared state across restarts and replicas, with creation order kept (AC-001.1);
- the outage behaviour: `503`, `/ready` non-`200`, `/health` `200`, and recovery without a restart (AC-001.2);
- the schema.

Its first criterion replaces AC-001.7 of the repo's spec `001-create-and-list-in-memory-tasks` ("tasks created before a restart are not listed after it"), as the system spec's notes require. That spec's assumption A9 (in memory, single replica) no longer holds. Its other criteria (AC-001.1 to AC-001.6: fields, validation, `422`, order) stay active unchanged.

## taskboard20-web — Task board on the start page that lists and adds tasks through the Task API

**Shape:** `web-nextjs` · **Criteria:** AC-001.3 to AC-001.7 · **Depends on:** []

The template start page `/` becomes the task board (A4). It owns:

- the list in creation order with the due-date display (AC-001.3);
- the empty state (AC-001.4);
- the create form, which appends the new task without a reload (AC-001.5);
- client-side title validation and mapping a `422` to the same messages (AC-001.6, A7);
- the two failure texts (AC-001.7).

All calls to the Task API run on the web site's server, so only the web site needs to be public.

## Contract

Caller: the `taskboard20-web` server, never the browser. Callee: `taskboard20-api`, at the base URL `TASK_API_URL` (dev: `http://taskboard20-api`). Every body is `application/json`. There is no authentication (spec non-goal). The API contract is the one in the API repo's `docs/specs/001-create-and-list-in-memory-tasks/design.md`, unchanged, plus the `503` below.

### `POST /tasks`

Request body:

| Field | Type | Rule |
|---|---|---|
| `title` | string | required; a JSON string, not coerced; 1 to 200 characters; at least one non-whitespace character; stored and returned as sent, not trimmed |
| `due_date` | string or null | optional, default `null`; exactly `YYYY-MM-DD` and a real calendar date; past dates are accepted |
| other fields | — | ignored (a client-sent `id` too) |

Responses:

- `201`: the created task `{"id": "<uuid4 string>", "title": "<as sent>", "due_date": "<YYYY-MM-DD>" | null}`, exactly these three fields, stored durably before the response.
- `422`: FastAPI's validation body `{"detail": [{"type", "loc", "msg", "input"}]}`. `loc[-1]` is `"title"` or `"due_date"`. Nothing is stored.
- `503`: the database is unreachable (see Errors). Nothing is stored.

The web site sends `due_date` as the date input's `YYYY-MM-DD` value, or `null` when it is empty.

### `GET /tasks`

- `200`: a JSON array of tasks, each exactly `{"id", "title", "due_date"}`, oldest first (creation order across every replica). The values equal those of the `201` that created each task.
- `503`: the database is unreachable.

### Probes (API)

- `GET /health`: `200` `{"status": "ok"}` while the process runs, whatever the database state.
- `GET /ready`: `200` `{"status": "ready"}` when a database check (`SELECT 1`) succeeds. Otherwise `503` `{"status": "not ready"}`.

### Web site (`taskboard20-web`)

- `/` is the task board.
- `TASK_API_URL` is a runtime server-side variable.
- `/api/health` and `/api/ready` do not depend on the Task API.

The fixed texts:

| Situation | Text |
|---|---|
| no tasks | `No tasks yet` |
| task without a due date | `No due date` |
| empty or whitespace-only title | `Title is required` |
| title over 200 characters | `Title must be at most 200 characters` |
| list failed | `Tasks could not be loaded. Try again later.` |
| save failed | `Task could not be saved. Try again.` |

### Errors

- **Validation:** `422` in FastAPI's standard body (above). Callers rely only on the status and `loc[-1]`, never on `type` or `msg`.
- **Database unavailable:** `503` with `{"detail": "Task store unavailable"}`. This covers connection refused, timeouts and a lost connection. The task is not created. The API does not crash or restart: each request opens or reuses a pooled connection, so the next request after recovery succeeds.
- **Unexpected API errors:** FastAPI's default `500`.
- **How the web site treats responses:**
  - `GET /tasks`: anything but `200` with a JSON array is "could not be loaded". That includes `503`, `500`, other statuses, a non-JSON body, a connection error and a timeout.
  - `POST /tasks`: anything but `201`, or a `422` whose `loc` ends in `title`, is "could not be saved". The page keeps the input.
  - A `422` naming `due_date` is also "could not be saved": the date picker cannot normally produce one.
  - The web site never shows the API's raw error text to the person.

### Timeouts

| Call | Timeout | When it expires or the other side is down |
|---|---|---|
| web → `GET /tasks` | 3 s | the page renders with `Tasks could not be loaded. Try again later.` and the form; no retry |
| web → `POST /tasks` | 5 s | `Task could not be saved. Try again.`, input kept; no automatic retry, so no duplicate task is created |
| API → Postgres, connect | 2 s | `503` |
| API → Postgres, statement | 3 s | `503` |
| API `/ready` database check | 1 s | `/ready` `503` |

A task request to the API therefore answers within about 6 s, even while the database hangs.

The kubelet's readiness probe takes a failing API replica out of the Service. With both replicas not ready, the web site's call fails fast (connection refused) and shows the failure text.

## gitops-app PR

**Before the components merge**, `/system:ship` applies the gitops entry with `cplat`:

1. `resource add postgres`
2. `uses` for `taskboard20-api`
3. `replicas` 2 for `taskboard20-api` in dev
4. `env` `TASK_API_URL` for `taskboard20-web`

It then renders, and the change lands in this repo's ship PR. The first resource adds the CloudNativePG operator under `operators/`; `/system:ship` applies `operators/root.yaml` once on the local cluster. The two component builds are pinned in dev on the `compose/<spec>` branch of the system ship (spec 144).

**No staging or prod pins** (`gitops_pin: []`): the spec keeps the system in `dev` only (non-goal). Staging and prod come through `/instance:promote` in a later spec.

**After Argo reconciles dev, verify:**

- The Postgres cluster is healthy.
- Two `taskboard20-api` pods are ready and have `DATABASE_URL` set.
- `taskboard20-web` has `TASK_API_URL=http://taskboard20-api`.
- `GET http://taskboard20-api.taskboard20-dev.localhost:8088/tasks` keeps a created task across `kubectl rollout restart`. A person does this restart check, or `/instance:verify dev`.
- `http://taskboard20-web.taskboard20-dev.localhost:8088/` shows the board.
- The dev smoke tests of both components pass.
