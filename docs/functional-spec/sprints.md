---
title: Sprints
status: draft
last-reviewed: 2026-11-23
related: [work-items.md, releases.md, closing-rules.md, project-views.md, actors-and-roles.md, audit-log.md]
---

# Sprints

## Overview

A Sprint is an iteration of a project. Work Items are assigned to Sprints. Sprints are independent of [Releases](releases.md): the two have no direct link and relate only through the Work Items assigned to them. Sprint assignments are independent of the Work Item hierarchy (BR-WI-018).

## Functional Requirements

### Sprints

- FR-SPR-001: Each project SHALL have its own set of Sprints.
- FR-SPR-002: Every Sprint SHALL have the fields defined in [Inputs / Outputs](#inputs--outputs).
- FR-SPR-015: WHEN a Sprint is created, the system SHALL assign it an automatic, sequential, numeric `id`, unique among the Sprints of the project.
- FR-SPR-003: IF a Sprint is submitted without `name`, `startdt` or `enddt`, THEN the system SHALL reject it.
- FR-SPR-016: IF a Sprint is created or edited so that its `name` equals the `name` of another Sprint of the same project, THEN the system SHALL refuse the request.
- FR-SPR-017: IF a Sprint is created or edited so that its `enddt` is earlier than its `startdt`, THEN the system SHALL refuse the request.
- FR-SPR-024: `name` SHALL be 1 to 200 characters of any Unicode text. `goal` SHALL have no functional length limit. `startdt` and `enddt` SHALL be calendar dates without a time of day, interpreted as UTC dates (FR-PLT-005) and persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)).
- FR-SPR-018: The date ranges of different Sprints of the same project MAY overlap.
- FR-SPR-004: The system SHALL NOT link a Sprint directly to a Release.

### State

- FR-SPR-005: The `state` of a Sprint SHALL be one of `planned`, `active` or `closed`.
- FR-SPR-006: WHEN a Sprint is created, the system SHALL set its `state` to `planned`.
- FR-SPR-007: The `state` of a Sprint SHALL change only by explicit user action, and only from `planned` to `active` or from `active` to `closed`. The system SHALL refuse every other change of `state`, including `planned` to `closed` and any change out of `closed`.
- FR-SPR-008: IF a request would make a Sprint `active` while another Sprint of the same project is `active`, THEN the system SHALL refuse it.
- FR-SPR-009: IF a request would change a Sprint to `closed` while any Work Item assigned to it is open, THEN the system SHALL refuse it.

### Assignment

- FR-SPR-010: A Work Item SHALL be assigned to at most one Sprint, through its optional `sprint` field, which holds the Sprint's `id` ([work-items.md](work-items.md)).
- FR-SPR-011: Only Work Items of kind `userstory`, `defect`, `refactoring`, `spike`, `task` or `issue` MAY be assigned to a Sprint. IF a request would assign an Epic or a Feature to a Sprint, THEN the system SHALL refuse it.
- FR-SPR-019: The system SHALL allow both open and closed Work Items to be assigned to a `planned` or `active` Sprint.
- FR-SPR-020: The assignments of a closed Sprint SHALL be frozen. IF a request would assign a Work Item to a closed Sprint, or remove or move a Work Item out of a closed Sprint, THEN the system SHALL refuse it.
- FR-SPR-022: A closed Sprint SHALL be fully frozen. IF a request would edit any field of a closed Sprint, THEN the system SHALL refuse it.
- FR-SPR-012: Moving a Work Item from one Sprint to another SHALL consist of changing its `sprint` field. The audit log records only that the Work Item was modified, not which Sprints were involved (FR-AUD-004).

### Deletion

- FR-SPR-021: IF a request would delete a Sprint to which any Work Item is assigned, THEN the system SHALL refuse it.
- FR-SPR-023: IF a request would delete a closed Sprint, THEN the system SHALL refuse it, whether or not any Work Item is assigned to it.

### Permissions and audit

- FR-SPR-013: Users with the `owner` or `member` role SHALL be able to create, modify and delete Sprints, change their state, and assign Work Items to them. Users with the `viewer` role SHALL NOT. Sprint management is not a project setting.
- FR-SPR-014: The creation, modification and deletion of a Sprint SHALL be recorded in the project's audit log with `entity` `sprint` ([audit-log.md](audit-log.md)).

## Business Rules

- BR-SPR-001: At most one Sprint per project is `active` at any time.
- BR-SPR-002: A Sprint can be closed only when every Work Item assigned to it is closed. Because a closed Sprint's assignments are frozen (FR-SPR-020) and its Work Items cannot be reopened (FR-CLS-006), every Work Item of a closed Sprint stays closed.
- BR-SPR-003: A closed Sprint is final: it cannot be reopened, edited or deleted, and its assignments cannot change.

## Use Cases

### UC-SPR-001: Close a Sprint

- Actor: user with the `owner` or `member` role, through the WEBSITE, or AI agent, through the CLI.
- Trigger: the actor requests that the active Sprint be closed.
- Preconditions: the Sprint is `active`.
- Main flow:
  1. The system checks that every Work Item assigned to the Sprint is closed.
  2. The system sets the Sprint's `state` to `closed`.
- Alternate flows:
  - 1a. At least one assigned Work Item is open: the system refuses the request and the Sprint stays `active`.
- Postconditions: the Sprint is `closed`, or unchanged if refused.

Acceptance criteria:

- Given an active Sprint with one open assigned Task, when a user closes the Sprint, then the request is refused.
- Given an active Sprint whose assigned Work Items are all closed, when a user closes the Sprint, then its `state` becomes `closed`.
- Given an active Sprint, when a user activates a second Sprint of the same project, then the request is refused.
- Given a planned Sprint, when a user changes its state to `closed`, then the request is refused.
- Given a closed Sprint, when a user changes its state to `active`, then the request is refused.
- Given a Feature, when a user assigns it to a Sprint, then the request is refused.
- Given a closed Task, when a user assigns it to an active Sprint, then the assignment is accepted.
- Given a closed Sprint, when a user assigns a Work Item to it or removes one from it, then the request is refused.
- Given a closed Sprint, when a user edits its `goal`, then the request is refused.
- Given a closed Sprint with no assigned Work Items, when a user deletes it, then the request is refused.
- Given a Sprint with one assigned Work Item, when a user deletes the Sprint, then the request is refused.
- Given a Sprint named `S1`, when a user creates another Sprint named `S1` in the same project, then the request is refused.
- Given a new Sprint with `enddt` earlier than `startdt`, when it is submitted, then it is rejected.
- Given a Sprint from 2026-01-01 to 2026-01-14, when a user creates a Sprint from 2026-01-10 to 2026-01-24, then it is accepted.

## Inputs / Outputs

| Field | Description | Required |
|-------|-------------|----------|
| `id` | Automatic sequential numeric identifier, unique among the project's Sprints. | Yes (automatic) |
| `name` | Name of the Sprint, unique within the project. | Yes |
| `goal` | Goal of the Sprint. | No |
| `startdt` | Start date of the Sprint: calendar date, UTC, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)). | Yes |
| `enddt` | End date of the Sprint: calendar date, UTC, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)); not earlier than `startdt`. | Yes |
| `state` | `planned`, `active` or `closed`. | Yes (set to `planned` on creation) |

## Dependencies

[work-items.md](work-items.md), [closing-rules.md](closing-rules.md), [project-views.md](project-views.md) (filters and Gantt), [audit-log.md](audit-log.md).

## Glossary References

[Sprint](glossary.md#sprint), [Release](glossary.md#release), [Work Item](glossary.md#work-item-wi).
