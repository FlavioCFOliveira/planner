# planner
Simple platform for software agile management, from planning to release.

## Goal

planner aims to manage the entire software development lifecycle — from planning to release — drawing inspiration from agile methodologies such as **Scrum** and **Kanban**.

## Core concept: Work Items

The central element of planner is the **Work Item (WI)**. Work Items can have different **types**, which distinguish the nature of the work each one is meant to address.

| Type | Summary |
|------|---------|
| Epic | Large business initiative spanning multiple Sprints |
| Feature | Software component delivering clear value to the end user |
| User Story | User need told from the user's point of view |
| Task | Technical breakdown of a work item into actual work |
| Defect | Software behaving differently from what is expected |
| Refactoring | Internal quality improvement (technical debt) |
| Spike | Research or prototyping before estimating |
| Issue | Something blocking the team's progress |

### Base data structure

All Work Items, regardless of their type, share the same base data structure:

| Field | Description | Required |
|-------|-------------|----------|
| `id` | Automatic sequential numeric identifier. It is unique across all Work Items of a project, whatever their type. | Yes (automatic) |
| `kind` | Work Item type: `epic`, `feature`, `userstory`, `task`, `defect`, `refactoring`, `spike` or `issue`. | Yes |
| `title` | Title that identifies the Work Item. | Yes |
| `desc` | Description of the work the Work Item is meant to address. | Yes |
| `planstart` | Planned start date. | No |
| `planend` | Planned end date. | No |
| `starteddt` | Start date. | No |
| `closeddt` | Date on which the Work Item was completed. Empty until the Work Item is completed, and cleared again if it is reopened. | Set on completion |

### Strategic planning types

**Epics** and **Features** are high-level strategic planning items. They group large volumes of work and align development with business goals.

#### Epic

An **Epic** is a large business initiative that cannot be delivered within a single iteration (Sprint). It is too big to be estimated accurately and must be broken down into smaller parts. An Epic is composed of several **Features**.

#### Feature

A **Feature** is a software component that delivers clear value to the end user and usually fits within a single release cycle (Release). A Feature is composed of several **User Stories**.

### User Story

A **User Story** describes a user need told from that user's point of view — **who** needs it, **what** they need, and **why**. It is the basic unit of value in agile development. A User Story is decomposed into multiple **Tasks**.

### Task

A **Task** is the technical breakdown of a User Story, Defect, Refactoring or Spike. It represents the actual work that developers, testers or designers need to do. A User Story, Defect, Refactoring or Spike is only done when all of its Tasks are finished (see [Closing rules](#closing-rules)).

### Defect

A **Defect** (error or bug) is a problem or failure in the software that causes it to behave differently from what is expected. It requires either immediate or planned correction. Like a User Story, a Defect can be decomposed into multiple **Tasks**.

### Refactoring

A **Refactoring** (technical debt) is internal work to improve code quality, performance or architecture without changing the behaviour visible to the user. It prevents the system from becoming slow or hard to maintain in the future. A Refactoring can be decomposed into multiple **Tasks**.

### Spike

A **Spike** (investigation) is a research-oriented piece of work, or the creation of a prototype. It is used when the team does not know how to implement something and needs to investigate before it can estimate the real work. A Spike can be decomposed into multiple **Tasks**.

### Issue

An **Issue** is a work item that blocks the progress of the team's work. An Issue can block any other work item in the hierarchy (Epic, Feature, User Story, Defect, Refactoring, Spike or Task). Unlike those items, an Issue is not decomposed into Tasks.

### Hierarchy

```
Epic
└── Feature
    └── User Story
        └── Task

Defect | Refactoring | Spike
└── Task

Issue ──blocks──▶ any work item above
```

### Closing rules

A Work Item **cannot be closed** while:

- it has **open children** in the hierarchy (e.g. an Epic with open Features, a Feature with open User Stories, a User Story with open Tasks); or
- it is **blocked by an open Issue**.

These rules apply wherever a Work Item is closed — WEBSITE, Kanban boards or CLI.

## Tags

**Tags** classify Work Items. Each project has its own set of tags. Each tag has:

| Field | Description | Required |
|-------|-------------|----------|
| `name` | Tag name. | Yes |
| `desc` | Description of the tag. | No |
| `color` | Display colour, in hexadecimal (e.g. `#FF0000`). | No |
| `value` | Sub-value. | No |

Following GitLab's model, there are two kinds of tags:

- **Simple tags** — only a name, with no categorisation. Example: `frontend`, `backend`, `design`, `qa`.
- **Scoped tags** — a name and a sub-value, written `name::value`. Example (priority): `prioridade::1-critica`, `prioridade::2-alta`, `prioridade::3-baixa`.

A Work Item can have several tags, including several scoped tags with the same name (e.g. `prioridade::1-critica` and `prioridade::2-alta`), but the same tag cannot be applied twice to the same Work Item.

Within a project, the combination of `name` + `value` is unique: `prioridade::1-critica` and `prioridade::2-alta` are distinct tags, but there cannot be two `prioridade::1-critica` tags, nor two simple `frontend` tags.

## Comments

Each project (that is, each project database) has a **comments** component linked to the project's Work Items. Comments form the project's historical progression line — a kind of **work log**.

Every comment has one of the following types:

| Type | Definition |
|------|------------|
| `HYPOTHESIS` | Something to verify or evaluate: a probable explanation, or a proposed approach not yet validated. It always requires a decision, even if that decision is to do nothing. |
| `TEST` | Result of tests that were run: what happened and how (e.g. all unit tests passed with no failures, or 7 of 100 tests passed, with the errors detailed). |
| `FINDING` | A fact observed or discovered during the work, backed by evidence (from a TEST, investigation, reading code, etc.). |
| `DECISION` | A choice that was made, with the alternatives considered and the justification. |
| `PROGRESS` | Concrete progress on the Work Item's work: what has been done since the last entry. |
| `UPDATE` | A change to the Work Item's scope, requirements, priority or context. It is not work done — that is PROGRESS. |
| `NOTE` | Free context or reference information (links, observations) that does not fit any other type. |
| `QUESTION` | An open question that needs an answer from someone for the work to proceed correctly. The answer goes in another comment (e.g. DECISION or NOTE). |
| `FOLLOW-UP` | Work identified for later, outside the Work Item's current scope; it may give rise to a new Work Item. |
| `BLOCKER` | Something currently preventing progress on this Work Item, and why. If the blocker needs to be managed as work in its own right, it should give rise to an Issue. |
| `HANDOFF` | Handing the Work Item over to another person, with the current state, context and next steps. |
| `ASSUMPTION` | A premise accepted as true without verification so the work can proceed. Unlike HYPOTHESIS, it is not verified; if it turns out to be false, the work may need to be revisited. |
| `RISK` | A problem that has not happened yet but may happen, with its impact and, if possible, its mitigation. Unlike BLOCKER, it is not yet preventing the work. |
| `REVIEW` | Result of a review of the Work Item's work (code, design, documentation): what was reviewed, observations, and verdict (approved / changes requested). |
| `DEBT` | Existing technical debt identified while working on the Work Item; it may give rise to a Refactoring. |

Every comment belongs to exactly one Work Item. Comments can be edited and deleted.

| Field | Description | Required |
|-------|-------------|----------|
| `id` | Automatic sequential numeric identifier. | Yes (automatic) |
| `wiid` | `id` of the Work Item the comment belongs to. | Yes |
| `kind` | Comment type (one of the types above). | Yes |
| `text` | Comment content. | Yes |
| `author` | User who wrote the comment. | Yes |
| `createddt` | Date the comment was created. | Yes (automatic) |

## Project views

The work-management interface (**WEBSITE (GUI)**) offers multiple ways of viewing a project:

1. **Work Item list** — a listing of the project's Work Items.
2. **Kanban boards** — Work Items organised in Kanban-style columns (see below).
3. **Gantt chart** — a timeline of the project, covering both **planning** (`planstart` / `planend`) and **history** (`starteddt` / `closeddt`).

### Kanban boards

Kanban boards behave in every respect like **GitLab issue boards**:

- A project can have **multiple boards**, each configured with its own columns.
- Every board has the columns `open` (first) and `closed` (last). In between, the user chooses the intermediate columns, each one matching a **tag** — e.g. `open`, `tag1`, `tag2`, `closed`.
- `open` shows the open Work Items (empty `closeddt`) that have none of the board's column tags; each tag column shows **only** the open Work Items that have that column's tag; `closed` shows the closed Work Items.
- Work Items can move freely from any column to any other column. Moving a card **removes the tag of the column it was in and adds the tag of the new column**:
  - tag column → tag column: the source tag is removed and the target tag is added;
  - `open` → tag column: the target tag is added;
  - tag column → `open`: the source tag is removed;
  - any column → `closed`: the source tag (if any) is removed and the Work Item is closed;
  - `closed` → any column: the Work Item is reopened (its `closeddt` is cleared) and the target tag (if any) is added.
- Moving a Work Item to `closed` is refused if it cannot be closed (see [Closing rules](#closing-rules)).

The **CLI** provides the same views (list, Kanban boards, Gantt) for AI agents, presenting the information as JSON or another format better suited to agents.

## Audit log

Each project has an **audit log**: a table in the project database recording the operations each user performs, whether through the **WEBSITE (GUI)** or the **CLI**.

- It records **only operations that create, modify or delete state** (objects). Read operations are not recorded.
- It does **not** store the changed data (no before/after values). It keeps a history of *which* changes a user made, for example: `created work item #123`, `modified WI #44`, `created comment #555`.
- It is the only place where the request's **`DEVICE-ID`** is stored (the CLI's `DEVICE-ID`, or `GUI` for the WEBSITE).
- It is **append-only**: entries can never be edited or deleted.
- The central `planner` database has an audit log with the **same structure**, recording platform-level operations (projects, users, permissions, tokens, etc.).

| Field | Description |
|-------|-------------|
| `id` | Automatic sequential numeric identifier. |
| `createddt` | Date and time of the operation. |
| `user` | User who performed the operation. |
| `deviceid` | `DEVICE-ID` the request came from (`GUI` for the WEBSITE). |
| `action` | `create`, `modify` or `delete`. |
| `entity` | Type of object affected (e.g. `workitem`, `comment`, `tag`). |
| `entityid` | `id` of the object affected. |

Readable descriptions such as `created work item #123` are derived from these fields.

## Architecture

planner follows a **client/server architecture**, with both components developed in **Rust**:

- **Server** — the web application, which exposes two interfaces:
  - **WEB API** — the application integration interface (REST);
  - **WEBSITE (GUI)** — the graphical user interface.
- **Client** — a command-line interface (CLI) that is the client application of the **WEB API**.

Data is persisted in **MariaDB** (see [Database and multiple projects](#database-and-multiple-projects)).

### AI-agent-oriented CLI

The CLI is **entirely oriented towards AI agents**. Its whole architecture — from how commands are structured to how the help is organised — is designed and optimised to be operated by AI agents such as **Claude Code**.

### CLI authentication

The CLI authenticates with a **token** issued on the platform's **WEBSITE (GUI)** on behalf of the authenticated user. With this token, the CLI operates on that user's behalf — even when it is driven by an AI agent.

- **Expiry** — each token has an expiry date: 30 days by default, configurable when the token is issued.
- **Revocation** — a token can be revoked at any time. Validity (expiry and revocation) is always checked **server-side**.
- **Multiple tokens** — a user can have several active tokens at the same time, and revoke each one individually.
- **Per-project permissions** — each token can have different permissions in each project. A token has only the permissions granted to it, and those can never exceed the user's own permissions in that project.

### Device ID

When installed, the CLI generates a unique **`DEVICE-ID`**, which it sends with every request to the **WEB API**. The `DEVICE-ID` is a random string of **24 characters**, made of uppercase letters, lowercase letters and digits (`[A-Za-z0-9]{24}`).

Each token is **bound to a device**: issuing a token on the WEBSITE **requires** specifying the `DEVICE-ID` it is for. When the server validates the token on each request, it also checks that the request's `DEVICE-ID` matches the one the token was issued for.

The `DEVICE-ID` also identifies the request's origin. **Every change made through the CLI** (Work Items, comments, etc.) remains attributed to the user, and the `DEVICE-ID` it came from is recorded **only in the project's audit log**.

Changes made through the **WEBSITE (GUI)** use a fixed, reserved `DEVICE-ID`, **`GUI`**, which is the same for all users. The CLI can never use `GUI` as its `DEVICE-ID`. As a result, every change — whether it comes from the CLI or the WEBSITE — has a `DEVICE-ID`.

### Database and multiple projects

The server persists its data in **MariaDB**.

planner supports **multiple projects**. Each project is stored in its own, separate MariaDB database, named `planner_<project>`.

A **central database named `planner`** orchestrates the whole platform. It holds platform-wide data, such as:

- which projects exist;
- the platform's users;
- the permissions each user has in each project.
