---
title: Audit log
status: draft
last-reviewed: 2026-11-23
related: [cli-authentication.md, multi-project.md, actors-and-roles.md]
---

# Audit log

## Overview

The audit log records which state changes users make, through the WEBSITE or the CLI. Each project has one audit log; the central `planner` database has another, with the same structure, for platform-level operations.

## Functional Requirements

- FR-AUD-001: Each project SHALL have an audit log, stored as a table in the project's database.
- FR-AUD-002: WHEN a user performs an operation that creates, modifies or deletes state in a project, through the WEBSITE or the CLI, the system SHALL append an entry to that project's audit log.
- FR-AUD-003: The system SHALL NOT record read operations in the audit log.
- FR-AUD-004: The audit log SHALL NOT store the changed data (no before or after values); it SHALL record only which change was made.
- FR-AUD-005: Each entry SHALL record the `DEVICE-ID` of the request: the CLI's `DEVICE-ID`, or `GUI` for the WEBSITE.
- FR-AUD-006: The system SHALL NOT allow audit log entries to be edited or deleted.
- FR-AUD-007: The central `planner` database SHALL have an audit log with the same structure (including `createddt` in UTC), recording every operation that creates, modifies or deletes platform-level data: projects (their creation and deletion, and any other platform-level project data, but not the project description `desc`, which is project data recorded in that project's audit log under FR-AUD-015), users (including deactivation and reactivation), platform administrator status, project roles and tokens.
- FR-AUD-008: The system SHALL derive readable descriptions of entries from their fields (for example `created work item #123`, `modified WI #44`, `created comment #555`).
- FR-AUD-009: The creation, modification and deletion of Sprints and Releases SHALL be recorded, with `entity` `sprint` and `release` respectively.
- FR-AUD-010: A project's audit log SHALL be readable by every user who holds a role in that project (`owner`, `member` or `viewer`) and by platform administrators.
- FR-AUD-011: The central audit log SHALL be readable only by platform administrators.
- FR-AUD-012: A project's audit log SHALL be readable through the WEBSITE and through the CLI. The central audit log SHALL be readable only through the WEBSITE, because platform operations are excluded from the CLI (FR-CLI-006).
- FR-AUD-014: The system SHALL record one entry for each object that an operation creates, modifies or deletes, including objects affected as a consequence of another object's change. This rule SHALL apply to project audit logs and to the central audit log alike. In particular:
  - deleting a Work Item records a `delete` entry for the Work Item, a `delete` entry for each of its comments, and a `modify` entry for each child whose `parent` is cleared and for each Work Item at the other end of a removed blocking link;
  - deleting a tag records a `delete` entry for the tag, a `modify` entry with `entity` `workitem` for each Work Item it is removed from, and a `modify` entry with `entity` `board` for each board that loses a column;
  - deleting a project records, in the central audit log, a `delete` entry for the project and one entry for each role and each token permission removed (FR-PRJ-009);
  - deactivating a user records, in the central audit log, a `modify` entry for the user, one entry for each token revoked and one entry for each project role removed (FR-IDN-012);
  - only a manual reordering of cards (FR-KBN-015) records a `modify` entry with `entity` `board`; the automatic placement of a card at the bottom of a column (FR-KBN-021) records no entry with `entity` `board`, because the change that caused it is already recorded as a `modify` entry with `entity` `workitem`.
- FR-AUD-015: WHEN a user edits a project's description (`desc`, FR-PRJ-010), the system SHALL append an entry with `entity` `project` to that project's audit log, and SHALL NOT record the edit in the central audit log.
- FR-AUD-013: WHEN a project is deleted, the system SHALL record the deletion in the central audit log. The project's own audit log is removed with the project's database ([multi-project.md](multi-project.md)).

## Business Rules

- BR-AUD-001: The audit log is append-only. The sole exception is the deletion of a project, which removes that project's audit log with its database (FR-AUD-013).
- BR-AUD-002: The audit log is the only place where the `DEVICE-ID` is stored. See [cli-authentication.md](cli-authentication.md#device-id).

## Inputs / Outputs

| Field | Description |
|-------|-------------|
| `id` | Automatic, increasing numeric identifier; never reused; may have gaps. |
| `createddt` | Date-time of the operation, UTC, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)); set by the system. |
| `user` | User who performed the operation, designated by the user `id` ([FR-IDN-026](users-and-identity.md#user-profile)). |
| `deviceid` | `DEVICE-ID` the request came from (`GUI` for the WEBSITE). |
| `action` | `create`, `modify` or `delete`. |
| `entity` | Type of object affected (for example `workitem`, `comment`, `tag`, `board`, `sprint`, `release`, `project`). |
| `entityid` | `id` of the object affected. |

## Acceptance Criteria

- Given a user reading a Work Item, then no audit entry is written.
- Given a user modifying Work Item #44 through the WEBSITE, then an entry with `action` `modify`, `entity` `workitem`, `entityid` 44 and `deviceid` `GUI` is appended.
- Given a Work Item with two comments, when it is deleted, then three `delete` entries are appended: one for the Work Item and one for each comment.
- Given a project with two roles and one token permission referring to it, when an administrator deletes it, then four entries are appended to the central audit log: one for the project, one for each role and one for the token permission.
- Given a user with two tokens and roles in two projects, when an administrator deactivates the user, then five entries are appended to the central audit log: one for the user, one for each token and one for each role.
- Given a project, when an `owner` edits its `desc`, then an entry with `action` `modify` and `entity` `project` is appended to that project's audit log, and no entry is appended to the central audit log.
- Given an existing audit entry, when any user tries to edit or delete it, then the request is refused.
- Given a user with the `viewer` role, when they read the project audit log, then it is shown.
- Given a user who is not an administrator, when they try to read the central audit log, then the request is refused.

## Glossary References

[Audit log](glossary.md#audit-log), [`DEVICE-ID`](glossary.md#device-id), [Central database](glossary.md#central-database).
