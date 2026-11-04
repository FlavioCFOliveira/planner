# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**planner** — Simple platform for software agile management, from planning to release.

Goal: manage the entire software development lifecycle, inspired by agile methodologies such as **Scrum** and **Kanban**.

## Domain

The central entity is the **Work Item (WI)**. Each WI has a **type** that distinguishes the nature of the work it addresses.

Work Item types: **Epic, Feature, User Story, Task, Defect, Refactoring, Spike, Issue**. All concepts below are WI types.

Base data structure shared by every WI, regardless of type:

- `id` — automatic sequential numeric identifier; unique across all WIs of a project, independently of their kind.
- `kind` — WI type (required): `epic`, `feature`, `userstory`, `task`, `defect`, `refactoring`, `spike`, `issue`.
- `title` — title that identifies the WI (required).
- `desc` — description of the work the WI is meant to address (required).
- `planstart` — planned start date.
- `planend` — planned end date.
- `starteddt` — start date (optional).
- `closeddt` — date on which the WI was completed; empty until completion, cleared on reopen.

**Strategic planning types** (high level): Epic and Feature. They group large volumes of work and align development with business goals.

**Epic** — a large business initiative that cannot be delivered within a single Sprint; too big to estimate accurately, so it must be broken down into smaller parts. An Epic is composed of several Features.

**Feature** — a software component that delivers clear value to the end user and usually fits within a single Release. A Feature is composed of several User Stories.

**User Story** — a user need described from that user's point of view (who, what, why); the basic unit of value in agile development. A User Story is decomposed into multiple Tasks.

**Task** — the technical breakdown of a User Story, Defect, Refactoring or Spike: the actual work developers, testers or designers must do.

Hierarchy: Epic → Feature → User Story → Task.

**Defect** (error/bug) — a problem or failure that makes the software behave differently from what is expected; requires immediate or planned correction. A Defect can be decomposed into multiple Tasks.

**Refactoring** (technical debt) — internal work to improve code quality, performance or architecture without changing user-visible behaviour; keeps the system from becoming slow or hard to maintain. A Refactoring can be decomposed into multiple Tasks.

**Spike** (investigation) — research or prototyping work, used when the team does not know how to implement something and must investigate before it can estimate the real work. A Spike can be decomposed into multiple Tasks.

Hierarchy: Defect | Refactoring | Spike → Task.

**Issue** — a work item that blocks the progress of the team's work. It can block any item in the hierarchy (Epic, Feature, User Story, Defect, Refactoring, Spike, Task). It is not part of the hierarchy and is not decomposed into Tasks.

**Closing rule** (applies everywhere: WEBSITE, Kanban, CLI): a WI cannot be closed while it has open children in the hierarchy (Epic → Features, Feature → User Stories, User Story/Defect/Refactoring/Spike → Tasks) or while it is blocked by an open Issue.

**Tag** — classifies WIs; tags are per project. Fields: `name` (required), `desc`, `color` (hexadecimal), `value` (sub-value); only `name` is required. GitLab-style:
- simple tag: name only, no categorisation (e.g. `frontend`, `backend`, `design`, `qa`);
- scoped tag: name + sub-value, written `name::value` (e.g. `prioridade::1-critica`, `prioridade::2-alta`, `prioridade::3-baixa`).

A WI can have many tags, including several scoped tags with the same name; the same tag cannot be applied twice to the same WI. Within a project, `name` + `value` is unique.

**Comments** — per project (stored in the project database), linked to the project's WIs. They form the project's historical progression line (a work log). Each comment belongs to exactly one WI; comments can be edited and deleted. Fields: `id` (automatic), `wiid` (WI id), `kind` (comment type), `text`, `author`, `createddt` (automatic) — all required.

Comment types:

- `HYPOTHESIS` — Something to verify or evaluate: a probable explanation, or a proposed approach not yet validated. It always requires a decision, even if that decision is to do nothing.
- `TEST` — Result of tests that were run: what happened and how (e.g. all unit tests passed with no failures, or 7 of 100 tests passed, with the errors detailed).
- `FINDING` — A fact observed or discovered during the work, backed by evidence (from a TEST, investigation, reading code, etc.).
- `DECISION` — A choice that was made, with the alternatives considered and the justification.
- `PROGRESS` — Concrete progress on the WI's work: what has been done since the last entry.
- `UPDATE` — A change to the WI's scope, requirements, priority or context. It is not work done — that is PROGRESS.
- `NOTE` — Free context or reference information (links, observations) that does not fit any other type.
- `QUESTION` — An open question that needs an answer from someone for the work to proceed correctly. The answer goes in another comment (e.g. DECISION or NOTE).
- `FOLLOW-UP` — Work identified for later, outside the WI's current scope; it may give rise to a new WI.
- `BLOCKER` — Something currently preventing progress on this WI, and why. If the blocker needs to be managed as work in its own right, it should give rise to an Issue.
- `HANDOFF` — Handing the WI over to another person, with the current state, context and next steps.
- `ASSUMPTION` — A premise accepted as true without verification so the work can proceed. Unlike HYPOTHESIS, it is not verified; if it turns out to be false, the work may need to be revisited.
- `RISK` — A problem that has not happened yet but may happen, with its impact and, if possible, its mitigation. Unlike BLOCKER, it is not yet preventing the work.
- `REVIEW` — Result of a review of the WI's work (code, design, documentation): what was reviewed, observations, and verdict (approved / changes requested).
- `DEBT` — Existing technical debt identified while working on the WI; it may give rise to a Refactoring.

**Project views** (WEBSITE/GUI): (1) Work Item list; (2) Kanban boards; (3) Gantt chart — planning from `planstart`/`planend`, history from `starteddt`/`closeddt`.

**Kanban boards** — behave in every respect like GitLab issue boards. A project can have multiple boards with different columns. Columns: `open` (first), user-chosen intermediate columns each matching a tag, `closed` (last) — e.g. `open`, `tag1`, `tag2`, `closed`. `open` = open WIs (empty `closeddt`) with none of the board's column tags; a tag column = only the open WIs that have that column's tag; `closed` = closed WIs. WIs can move freely between any columns: moving a card removes the tag of the column it was in and adds the tag of the new column (`open` has no tag); moving to `closed` also closes the WI, moving out of `closed` reopens it (clears `closeddt`). Moving to `closed` is refused if the closing rule is not met.

The CLI provides the same views as the WEBSITE (list, Kanban, Gantt) for AI agents, as JSON or another agent-appropriate format.

**Audit log** — one table per project (in the project database) recording the operations performed by users, through the WEBSITE or the CLI. Only operations that create, modify or delete state are logged; reads are not. It does not store changed data (no before/after values), only which change was made (e.g. `created work item #123`, `modified WI #44`, `created comment #555`). It is the only place the `DEVICE-ID` is stored. Append-only: entries are never edited or deleted. Fields: `id`, `createddt` (date/time), `user`, `deviceid`, `action` (`create` | `modify` | `delete`), `entity` (e.g. `workitem`, `comment`, `tag`), `entityid`; readable descriptions are derived from them. The central `planner` database has an audit log with the same structure for platform-level operations (projects, users, permissions, tokens, etc.).

## Architecture

Client/server architecture, both components written in **Rust**:

- **Server** — web application with two interfaces:
  - **WEB API** — application integration interface (REST);
  - **WEBSITE (GUI)** — graphical user interface.
- **Client** — CLI, the client application of the **WEB API**.

**The CLI is COMPLETELY AI-agent-oriented.** Its entire architecture — from command structure to how the help is organised — must be designed and optimised to be operated by AI agents such as Claude Code. Treat AI agents as the primary user of the CLI in every design decision.

CLI authentication: the CLI uses a **token** issued on the platform's WEBSITE (GUI) on behalf of the authenticated user, so the CLI acts on that user's behalf (even when operated by an AI agent).
- Tokens expire (default 30 days, configurable at issue time) and can be revoked; validity is always checked server-side.
- A user can have several active tokens at once and revoke each individually.
- Each token can have different permissions per project; a token has only the permissions granted to it, never more than the user's own permissions in that project.
- On installation the CLI generates a unique `DEVICE-ID`, sent with every WEB API request: a random 24-character string of uppercase letters, lowercase letters and digits (`[A-Za-z0-9]{24}`).
- Tokens are bound to a device: issuing a token **requires** its `DEVICE-ID`; on each request the server validates the token together with the `DEVICE-ID` (they must match).
- The `DEVICE-ID` also identifies the request's origin: every change made through the CLI (WIs, comments, etc.) stays attributed to the user; the `DEVICE-ID` is recorded **only in the project's audit log** (never as a field of WIs, comments or other domain entities).
- Changes made through the WEBSITE (GUI) use the fixed, reserved `DEVICE-ID` `GUI`, the same for all users; the CLI can never use `GUI`. So every change has a `DEVICE-ID`.

Database: **MariaDB**, used by the server for persistence.

Multi-project: the platform supports multiple projects; each project is a separate MariaDB database named `planner_<project>`. A central database named `planner` orchestrates the whole platform: it records which projects exist, the users, and each user's permissions per project (among other platform-wide data).
