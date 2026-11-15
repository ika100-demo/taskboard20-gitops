---
spec_id: 001-persistent-tasks-and-web-task-board
title: Persistent tasks and web task board
status: approved
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks and web task board

## Problem

The Task API keeps tasks in the memory of one instance, so every restart or redeploy loses all tasks, and a second replica would show a different list. The only way to use the board is to call the API directly; the web site is a fresh template that shows no tasks. People need tasks that survive restarts and a page where they can see the board and add tasks without a tool. A Postgres database for the API and a task board in the web UI deliver both.

## Stories

As a team member, I want tasks I created to still be there after the service restarts or is redeployed, so that I can rely on the board as the record of what has to be done.
**Repos:** taskboard20-api, taskboard20-gitops

As a team member, I want to open the web task board and see every task with its due date, so that I know what is open without calling the API.
**Repos:** taskboard20-web, taskboard20-api, taskboard20-gitops

As a team member, I want to add a task with a title and an optional due date from the web task board, so that I can record work in the browser.
**Repos:** taskboard20-web, taskboard20-api, taskboard20-gitops

## Acceptance criteria

- **AC-001.1** Given a task created with `POST /tasks` `{"title": "Buy milk", "due_date": "2026-12-01"}`, when every instance of the Task API is restarted or replaced by a new deployment (also with two or more replicas running), then `GET /tasks` on any instance lists that task with the same `id`, `title` and `due_date` as returned at creation, and tasks keep their creation order.
- **AC-001.2** Given the database is unreachable, when a client sends `POST /tasks` or `GET /tasks`, then the response is `503` with a JSON error body and no task is created, `GET /ready` answers non-`200` while `GET /health` stays `200`; once the database is reachable again, both task requests succeed without a restart of the Task API.
- **AC-001.3** Given tasks A, B and C created in that order, when a person opens the task board page of the web site, then it lists A, B, C in creation order, each showing its title exactly as stored and its due date as `YYYY-MM-DD`, or `No due date` when it has none.
- **AC-001.4** Given no tasks exist, when a person opens the task board page, then it shows the text `No tasks yet` and the create form.
- **AC-001.5** Given the task board page is open, when a person enters the title `Call Bob`, optionally picks a due date, and submits the form, then the new task appears at the end of the list without a manual page reload, the form is emptied, and `GET /tasks` on the Task API contains a task with that title and due date (or `null`).
- **AC-001.6** Given the task board page is open, when a person submits the form with an empty or whitespace-only title, or a title longer than 200 characters, then the page shows `Title is required` or `Title must be at most 200 characters` respectively next to the title field, the typed input is kept, and no task is created.
- **AC-001.7** Given the Task API is unreachable or answers an error, when a person opens the task board page, then the page still loads and shows `Tasks could not be loaded. Try again later.` instead of the list; when a person submits a valid task in that state, the page shows `Task could not be saved. Try again.` and keeps the typed input.

## Non-goals

- Editing, completing or deleting tasks, in the API or the web UI; a page for a single task.
- Login, user accounts, per-user task lists or any access control: while the system runs only in `dev`, anyone who reaches the web site or the Task API may read and add tasks; login is a later spec.
- Pagination, search, filtering or sorting other than creation order.
- Moving tasks from the current in-memory store into the database.
- Database backups and restore, and running the system in staging or prod (the components run only in `dev`).
- Live updates of the board when another person adds a task (a page reload shows them).

## Assumptions

- **A1** The task shape and the API contract stay as today: `POST /tasks` and `GET /tasks` with exactly `id`, `title`, `due_date`, the same validation and `422` errors (other option: add `created_at` or `done` now; affects AC-001.1, AC-001.5).
- **A2** Tasks held in memory at the switch to the database are not carried over; they are lost as on any restart today (other option: a one-off copy; affects AC-001.1).
- **A3** A database outage answers `503` with a JSON error body and makes the API not ready, while liveness stays healthy so the pod is not restarted in a loop (other option: `500`; affects AC-001.2).
- **A4** The task board is the web site's start page `/` (other option: a separate `/tasks` page; affects AC-001.3–AC-001.7).
- **A5** The board lists tasks oldest first, the same order as `GET /tasks`, and a new task is appended at the end (other option: newest first; affects AC-001.3, AC-001.5).
- **A6** The due date is shown and entered as `YYYY-MM-DD` with the browser's date picker, and a task without one shows `No due date` (other option: a localised date format; affects AC-001.3, AC-001.5).
- **A7** The web form checks the title with the API's rules (required, not only whitespace, at most 200 characters) and shows the messages in AC-001.6; a `422` from the API is shown the same way (other option: show the API's raw error; affects AC-001.6).
- **A8** Error texts are in English and fixed as written in AC-001.4, AC-001.6 and AC-001.7 (other option: other wording; affects those criteria).
- **A9** One Postgres database is declared for the system and used only by the Task API, in `dev` only for now (other option: also staging and prod in this spec; affects AC-001.1, AC-001.2).

## Notes for the architect

- The API's spec `taskboard20-api:001-create-and-list-in-memory-tasks` has AC-001.7 ("tasks created before a restart are not listed after it") and assumption A9 (in-memory, single replica); AC-001.1 here reverses that behaviour, so the API's component spec derived from the plan should supersede that criterion (`replaces AC-001.7` in that repo).
- The browser need not reach the Task API directly: the web site can call it inside the environment, so only the web site's address has to be public.
- No resources are declared in `system.yaml` yet; the database needs to be added and the API wired to it, and the web site wired to the API.

## Open questions

- ~~Tasks become persistent data that anyone who can reach the web site or the API can read and add to, without login. Is that acceptable for this spec? (suggested: yes, no access control while the system runs in `dev` only; login is a later spec; affects AC-001.3, AC-001.5, new criterion if no)~~ Answered 2026-11-16: yes, no login for now; no access control while the system runs only in `dev`, login is a later spec (see Non-goals).

## Changelog

- 2026-11-16 created
- 2026-11-16 amended: open question on access answered (no login, no access control while dev-only; login a later spec), folded into Non-goals; criteria unchanged
- 2026-11-16 approved
