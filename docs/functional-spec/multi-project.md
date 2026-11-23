---
title: Multiple projects
status: draft
last-reviewed: 2026-11-23
related: [overview.md, actors-and-roles.md, users-and-identity.md, audit-log.md]
---

# Multiple projects

## Overview

The platform supports multiple projects. Each project is isolated in its own database; a central database orchestrates the platform.

## Functional Requirements

- FR-PRJ-001: The system SHALL support multiple projects.
- FR-PRJ-002: The system SHALL store each project in its own, separate database, named `planner_<project>`, where `<project>` is the project name.
- FR-PRJ-003: The system SHALL keep a central database, named `planner`, holding platform-wide data, including: which projects exist; the platform's users; the permissions each user has in each project; the platform audit log.
- FR-PRJ-004: The system SHALL store a project's comments and audit log in that project's database.
- FR-PRJ-005: A project name SHALL consist of 1 to 50 characters, drawn from lowercase letters `a`–`z`, digits `0`–`9` and underscore, and SHALL start with a letter; that is, it SHALL match `^[a-z][a-z0-9_]{0,49}$`.
- FR-PRJ-006: A project name SHALL be unique on the platform.
- FR-PRJ-007: The system SHALL NOT allow a project to be renamed.
- FR-PRJ-010: A project SHALL have an optional description, `desc`, in addition to its name. `desc` SHALL have no functional length limit. It is a project setting (FR-ACC-008): users with the `owner` role and platform administrators SHALL be able to edit it.
- FR-PRJ-008: Only platform administrators SHALL be able to create and delete projects. Creating a project requires naming at least one `owner` (FR-ACC-015).
- FR-PRJ-009: WHEN a project is deleted, the system SHALL drop the project's database, remove every role and every token permission that refers to the project, and record the deletion in the central audit log.

## Business Rules

- BR-PRJ-001: Tags, comments, boards, Sprints, Releases and Work Item identifiers are per project ([tags.md](tags.md), [comments.md](comments.md), [kanban-boards.md](kanban-boards.md), [sprints.md](sprints.md), [releases.md](releases.md), [work-items.md](work-items.md)).

## Constraints

- Databases are MariaDB databases (NFR-ARCH-004 in [overview.md](overview.md)).

## Acceptance Criteria

- Given a request to create a project named `web_app2`, then it is accepted.
- Given a request to create a project named `2web`, `Web` or `web-app`, then it is refused.
- Given a project named `web`, when an administrator creates another project named `web`, then the request is refused.
- Given an existing project, when anyone tries to rename it, then the request is refused.
- Given a user with the `member` role, when they edit the project `desc`, then the request is refused; when an `owner` edits it, then the change is accepted.
- Given a project with members and tokens granting access to it, when an administrator deletes it, then its database is dropped, the roles and token permissions are removed, and the deletion is recorded in the central audit log.

## Glossary References

[Project](glossary.md#project), [Project database](glossary.md#project-database), [Central database](glossary.md#central-database).
