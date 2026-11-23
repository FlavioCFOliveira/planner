---
title: Closing rules
status: draft
last-reviewed: 2026-11-23
related: [work-items.md, kanban-boards.md, sprints.md, releases.md, cli.md]
---

# Closing rules

## Overview

This file states when a Work Item may be closed, and which actions are refused because they would place an open Work Item under a closed parent or block a closed Work Item. The effect of closing and reopening on `closeddt` is in [work-items.md](work-items.md#dates). The Kanban moves that close or reopen a Work Item are in [kanban-boards.md](kanban-boards.md). Closing a Sprint and releasing a Release have their own rules in [sprints.md](sprints.md) and [releases.md](releases.md).

## Scope

The rules apply wherever the action is requested: the WEBSITE, Kanban boards and the CLI.

## Functional Requirements

- FR-CLS-001: IF a request to close a Work Item violates BR-CLS-001 or BR-CLS-002, THEN the system SHALL refuse it.
- FR-CLS-002: IF a request would create a Work Item whose parent is closed, or reopen a Work Item whose parent is closed, THEN the system SHALL refuse it.
- FR-CLS-003: IF a request would create or reopen an Issue that blocks a closed Work Item, THEN the system SHALL refuse it.
- FR-CLS-004: IF a request would add a blocking link from an open Issue to a closed Work Item, THEN the system SHALL refuse it.
- FR-CLS-005: IF a request would set the `parent` of an open Work Item to a closed Work Item, THEN the system SHALL refuse it.
- FR-CLS-006: IF a request would reopen a Work Item that is assigned to a closed Sprint or to a released Release, THEN the system SHALL refuse it ([sprints.md](sprints.md), [releases.md](releases.md)).

## Business Rules

- BR-CLS-001: A Work Item cannot be closed while it has open children in the hierarchy:
  - an Epic with open Features;
  - a Feature with open User Stories;
  - a User Story, Defect, Refactoring or Spike with open Tasks.
- BR-CLS-002: A Work Item cannot be closed while it is blocked by an open Issue. This applies to every type, including an Issue blocked by another Issue.
- BR-CLS-003: As a consequence of BR-CLS-001, a User Story, Defect, Refactoring or Spike is only done when all of its Tasks are finished.

## Use Cases

### UC-CLS-001: Close a Work Item

- Actor: user, through the WEBSITE or a Kanban board, or AI agent, through the CLI.
- Trigger: the actor requests that an open Work Item be closed.
- Preconditions: the Work Item is open.
- Main flow:
  1. The system checks BR-CLS-001 and BR-CLS-002.
  2. The system sets `closeddt` to the UTC date and time of the operation, persisted in UTC (FR-WI-007, [NFR-DATA-001](overview.md#non-functional-requirements)).
- Alternate flows:
  - 1a. A rule is violated: the system refuses the request and the Work Item stays open.
- Postconditions: the Work Item is closed, or unchanged if refused.

Acceptance criteria:

- Given a User Story with one open Task, when a user tries to close the User Story, then the request is refused.
- Given a Task blocked by an open Issue, when a user tries to close the Task, then the request is refused.
- Given an Issue blocked by another open Issue, when a user tries to close the first Issue, then the request is refused.
- Given a Feature whose User Stories are all closed and which is not blocked by an open Issue, when a user closes it, then its `closeddt` is set to the UTC date and time of the operation.
- Given a closed User Story, when a user creates a Task with that User Story as parent, then the request is refused.
- Given a closed User Story with a closed Task, when a user reopens the Task, then the request is refused.
- Given a closed Task and a closed Issue that blocks it, when a user reopens the Issue, then the request is refused.
- Given a closed Task and an open Issue, when a user adds a link by which the Issue blocks the Task, then the request is refused.
- Given an open Task and a closed User Story, when a user sets the User Story as the Task's parent, then the request is refused.
- Given a closed Task assigned to a closed Sprint, when a user reopens the Task, then the request is refused.
- Given a closed Task assigned to a released Release, when a user reopens the Task, then the request is refused.

## Dependencies

[work-items.md](work-items.md), [kanban-boards.md](kanban-boards.md).

## Glossary References

[Closed](glossary.md#closed), [Open](glossary.md#open), [Child](glossary.md#child), [Blocking](glossary.md#blocking), [Reopen](glossary.md#reopen).
