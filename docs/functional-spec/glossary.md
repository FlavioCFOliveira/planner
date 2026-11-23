---
title: Glossary
status: draft
last-reviewed: 2026-11-23
related: [README.md]
---

# Glossary

Terms are listed alphabetically. Each term is used with this meaning throughout the specification.

### AI agent

An automated agent, such as Claude Code, that operates the CLI. AI agents are the primary user of the CLI. See [cli.md](cli.md).

### Audit log

An append-only record of the state-changing operations performed by users. Each project has one; the central `planner` database has one for platform-level operations. See [audit-log.md](audit-log.md).

### Blocking

The many-to-many relationship by which an open Issue prevents another Work Item, including another Issue, from being closed. See [work-items.md](work-items.md#issue-blocking).

### Board

See Kanban board.

### Card

The representation of a Work Item on a Kanban board.

### Central database

The database named `planner` that orchestrates the platform: projects, users, per-project permissions and the platform audit log. See [multi-project.md](multi-project.md).

### Child

A Work Item directly below another in the hierarchy (for example, a Task of a User Story). The Work Item above is its parent, recorded in the child's `parent` field. See [work-items.md](work-items.md#hierarchy).

### CLI

The command-line client application of the WEB API, designed primarily for AI agents. See [cli.md](cli.md).

### Closed

The state of a Work Item whose `closeddt` is set. See [closing-rules.md](closing-rules.md).

### Comment

An entry in a project's work log, belonging to exactly one Work Item and having one comment type. See [comments.md](comments.md).

### Comment type

The classification of a comment, for example `DECISION` or `PROGRESS`. See [comments.md](comments.md#comment-types).

### Deactivated user

A user whose tokens and project roles have been removed by a platform administrator. Users are deactivated, never deleted. See [users-and-identity.md](users-and-identity.md#deactivation).

### Defect

A Work Item type: a problem or failure that makes the software behave differently from what is expected.

### `DEVICE-ID`

An identifier of the origin of a request. The CLI generates a random one on installation; the WEBSITE uses the reserved value `GUI`. See [cli-authentication.md](cli-authentication.md#device-id).

### Display name

The planner-only profile field `displayname`, distinct from the `name` copied from the Identity Provider. When set, it is shown instead of `name`. See [users-and-identity.md](users-and-identity.md#user-profile).

### Epic

A Work Item type: a large business initiative that cannot be delivered within a single Sprint.

### Feature

A Work Item type: a software component that delivers clear value to the end user and usually fits within a single Release.

### Identity Provider (IdP)

The external system that performs user registration and login through OpenID Connect. See [users-and-identity.md](users-and-identity.md).

### Inviolable requirement

A requirement marked `(INVIOLABLE)` after its identifier: it admits no exception and prevails over any conflicting requirement. See [README.md](README.md#inviolable-requirements).

### Issue

A Work Item type: an item that blocks the progress of the team's work. It is not part of the hierarchy.

### Kanban board

A project view that arranges Work Items in columns, behaving like GitLab issue boards. See [kanban-boards.md](kanban-boards.md).

### Open

The state of a Work Item whose `closeddt` is empty.

### Parent

See Child.

### Platform administrator

A user with full privileges over the whole platform. See [actors-and-roles.md](actors-and-roles.md#platform-administrator).

### Project

An isolated workspace with its own Work Items, tags, comments, boards and audit log, stored in its own database. See [multi-project.md](multi-project.md).

### Project settings

The part of a project that only users with the `owner` role (and platform administrators) may change: membership, Kanban board configuration and the project description. See [actors-and-roles.md](actors-and-roles.md).

### Project database

The database of one project, named `planner_<project>`.

### Refactoring

A Work Item type: internal work to improve code quality, performance or architecture without changing user-visible behaviour (technical debt).

### Release

A version of the software that a project delivers, to which Work Items other than Issues may be assigned. A Release is released when its `releaseddt` is set. See [releases.md](releases.md).

### Reopen

To return a closed Work Item to the open state by clearing its `closeddt`.

### Role

A fixed set of permissions a user holds in a project: `owner`, `member` or `viewer`. See [actors-and-roles.md](actors-and-roles.md#roles).

### Scoped tag

A tag with a name and a sub-value, written `name::value`.

### Simple tag

A tag with a name only.

### Spike

A Work Item type: research or prototyping work done before the real work can be estimated.

### Sprint

An iteration of a project, with a start and an end, in the state `planned`, `active` or `closed`. At most one Sprint per project is active. See [sprints.md](sprints.md).

### Tag

A per-project label that classifies Work Items. See [tags.md](tags.md).

### Task

A Work Item type: the technical breakdown of a User Story, Defect, Refactoring or Spike into the actual work to be done.

### Token

A credential issued on the WEBSITE that lets the CLI act on behalf of a user, bound to one `DEVICE-ID`. See [cli-authentication.md](cli-authentication.md).

### UTC

Coordinated Universal Time. Every date and date-time is persisted in UTC without exception (NFR-DATA-001), date-only fields are UTC calendar dates (FR-PLT-005), and every date-time is displayed in UTC (FR-PLT-007). See [overview.md](overview.md).

### User profile

The planner record of a user, keyed by the IdP user id and created at the user's first successful login. See [users-and-identity.md](users-and-identity.md#user-profile).

### User Story

A Work Item type: a user need described from that user's point of view (who, what, why); the basic unit of value in agile development.

### WEB API

The REST application integration interface exposed by the server.

### WEBSITE (GUI)

The graphical user interface exposed by the server.

### Work Item (WI)

The central entity of planner. Every Work Item has a type (`kind`) and the base data structure. See [work-items.md](work-items.md).

### Work log

The historical progression line of a project, formed by its comments.
