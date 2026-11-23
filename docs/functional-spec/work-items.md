---
title: Work Items
status: draft
last-reviewed: 2026-11-23
related: [closing-rules.md, tags.md, comments.md, sprints.md, releases.md, project-views.md, kanban-boards.md]
---

# Work Items

## Overview

The Work Item (WI) is the central entity of planner. Every Work Item has a type, which distinguishes the nature of the work it addresses, and shares one base data structure with every other Work Item.

## Functional Requirements

### Types

- FR-WI-001: The system SHALL support exactly these Work Item types, recorded in `kind`: `epic`, `feature`, `userstory`, `task`, `defect`, `refactoring`, `spike`, `issue`.
- FR-WI-002: Every Work Item, regardless of its type, SHALL have the base data structure defined in [Inputs / Outputs](#inputs--outputs).
- FR-WI-010: The system SHALL NOT allow the `kind` of a Work Item to be changed after creation.

### Identifier

- FR-WI-003: WHEN a Work Item is created, the system SHALL assign it an automatic, sequential, numeric `id`.
- FR-WI-004: The `id` SHALL be unique across all Work Items of a project, whatever their type.

### Mandatory fields

- FR-WI-005: IF a Work Item is submitted without `kind`, `title` or `desc`, THEN the system SHALL reject it.
- FR-WI-025: `title` SHALL be 1 to 200 characters of any Unicode text. `desc` SHALL have no functional length limit; its technical maximum is left to the technical specification.

### Dates

- FR-WI-011: `planstart` and `planend` SHALL be calendar dates without a time of day, interpreted as UTC dates (FR-PLT-005) and persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)).
- FR-WI-012: `starteddt` SHALL be a date and time entered by the user, in UTC and persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)); the user SHALL be able to set, change and clear it.
- FR-WI-006: The `closeddt` of a Work Item SHALL be empty while the Work Item is open.
- FR-WI-007: WHEN a Work Item is closed, the system SHALL set its `closeddt` to the UTC date and time of the close operation, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)).
- FR-WI-013: The system SHALL NOT allow a user to enter or edit `closeddt`.
- FR-WI-008: WHEN a Work Item is reopened, the system SHALL clear its `closeddt`.
- FR-WI-014: The system SHALL NOT validate the relative order of a Work Item's dates. For example, `planstart` may be later than `planend`, and `starteddt` may be later than `closeddt`.

### Hierarchy

- FR-WI-015: The system SHALL record a Work Item's parent in the optional field `parent`, which holds the `id` of the parent Work Item in the same project.
- FR-WI-016: IF a request would set a `parent` that violates BR-WI-014 or BR-WI-016, THEN the system SHALL refuse it.
- FR-WI-022: The system SHALL allow the `parent` of a Work Item to be changed or cleared after creation, subject to FR-WI-016 and FR-CLS-005.

### Issue blocking

- FR-WI-009: The system SHALL allow an Issue to block an Epic, Feature, User Story, Defect, Refactoring, Spike, Task or another Issue.
- FR-WI-017: Blocking SHALL be many-to-many: one Issue MAY block several Work Items, and one Work Item MAY be blocked by several Issues.
- FR-WI-023: IF a request would add a blocking link that creates a cycle, THEN the system SHALL refuse it. A cycle exists when an Issue would block itself, directly or through a chain of blocking links (for example, A blocks B and B blocks A, or A blocks B, B blocks C and C blocks A).

### Sprint and Release assignment

- FR-WI-018: A Work Item SHALL have an optional field `sprint`, which holds the `id` of the Sprint it is assigned to. Assignment rules are in [sprints.md](sprints.md).
- FR-WI-019: A Work Item SHALL have an optional field `release`, which holds the `id` of the Release it is assigned to. Assignment rules are in [releases.md](releases.md).

### Deletion

- FR-WI-020: The system SHALL allow a Work Item to be deleted, except as FR-WI-024 states.
- FR-WI-024: IF a request would delete a Work Item assigned to a closed Sprint or to a released Release, THEN the system SHALL refuse it.
- FR-WI-021: WHEN a Work Item is deleted, the system SHALL, in the same operation:
  - clear the `parent` field of each of its children, which become parentless;
  - delete its comments;
  - remove every blocking link in which it takes part, whether as the blocking Issue or as the blocked Work Item;
  - remove its tag assignments.

## Business Rules

### Type definitions

- BR-WI-001: An **Epic** is a large business initiative that cannot be delivered within a single iteration (Sprint). It is too big to be estimated accurately and must be broken down into smaller parts. Epics and Features are strategic planning types: they group large volumes of work and align development with business goals.
- BR-WI-002: A **Feature** is a software component that delivers clear value to the end user and usually fits within a single release cycle (Release).
- BR-WI-003: A **User Story** describes a user need from that user's point of view: who needs it, what they need, and why. It is the basic unit of value in agile development.
- BR-WI-004: A **Task** is the technical breakdown of a User Story, Defect, Refactoring or Spike. It represents the actual work that developers, testers or designers need to do.
- BR-WI-005: A **Defect** (error or bug) is a problem or failure in the software that causes it to behave differently from what is expected. It requires either immediate or planned correction.
- BR-WI-006: A **Refactoring** (technical debt) is internal work to improve code quality, performance or architecture without changing the behaviour visible to the user. It prevents the system from becoming slow or hard to maintain.
- BR-WI-007: A **Spike** (investigation) is research or the creation of a prototype, used when the team does not know how to implement something and must investigate before it can estimate the real work.
- BR-WI-008: An **Issue** is a Work Item that blocks the progress of the team's work.

### Hierarchy

- BR-WI-009: An Epic is composed of several Features.
- BR-WI-010: A Feature is composed of several User Stories.
- BR-WI-011: A User Story is decomposed into multiple Tasks.
- BR-WI-012: A Defect, a Refactoring and a Spike can each be decomposed into multiple Tasks.
- BR-WI-013: An Issue is not part of the hierarchy and is not decomposed into Tasks.
- BR-WI-019: BR-WI-009 to BR-WI-012 describe the intended decomposition and set no minimum. A Work Item may have any number of children, including none.
- BR-WI-014: The allowed parent and child combinations are exactly these; no other combination is allowed:

  | Parent `kind` | Child `kind` |
  |---------------|--------------|
  | `epic` | `feature` |
  | `feature` | `userstory` |
  | `userstory` | `task` |
  | `defect` | `task` |
  | `refactoring` | `task` |
  | `spike` | `task` |

- BR-WI-015: A Work Item has at most one parent. The hierarchy is a strict tree.
- BR-WI-016: An Epic and an Issue never have a parent. As a consequence of BR-WI-014, a Defect, a Refactoring and a Spike never have a parent either. A Feature, a User Story and a Task may have a parent or be parentless.
- BR-WI-017: Because a Feature, a User Story and a Task may be parentless (BR-WI-016), the children of a deleted Work Item remain valid after FR-WI-021 clears their `parent`.
- BR-WI-018: Sprint and Release assignments are independent of the hierarchy. For example, a Task may be in a different Sprint or Release from its User Story.

The resulting hierarchy:

```
Epic
└── Feature
    └── User Story
        └── Task

Defect | Refactoring | Spike
└── Task

Issue ──blocks──▶ Epic, Feature, User Story, Defect, Refactoring, Spike, Task, Issue
```

## Inputs / Outputs

Base data structure, shared by every Work Item:

| Field | Description | Required |
|-------|-------------|----------|
| `id` | Automatic sequential numeric identifier, unique across all Work Items of a project, whatever their type. | Yes (automatic) |
| `kind` | Work Item type: `epic`, `feature`, `userstory`, `task`, `defect`, `refactoring`, `spike` or `issue`. Cannot be changed after creation. | Yes |
| `title` | Title that identifies the Work Item. | Yes |
| `desc` | Description of the work the Work Item is meant to address. | Yes |
| `parent` | `id` of the parent Work Item (BR-WI-014 to BR-WI-016). | No |
| `planstart` | Planned start date: calendar date, UTC, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)). | No |
| `planend` | Planned end date: calendar date, UTC, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)). | No |
| `starteddt` | Start date-time, UTC, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)); entered by the user. | No |
| `closeddt` | Date-time of the close operation, UTC, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)); set by the system. Empty while open; cleared on reopen. | Set by the system on close |
| `sprint` | `id` of the Sprint the Work Item is assigned to ([sprints.md](sprints.md)). | No |
| `release` | `id` of the Release the Work Item is assigned to ([releases.md](releases.md)). | No |

## Dependencies

- Closing, reopening and the constraints on closed Work Items: [closing-rules.md](closing-rules.md).
- Classification: [tags.md](tags.md). Work log: [comments.md](comments.md).
- Iterations and delivery: [sprints.md](sprints.md), [releases.md](releases.md).
- Presentation: [project-views.md](project-views.md), [kanban-boards.md](kanban-boards.md).

## Glossary References

[Work Item](glossary.md#work-item-wi), [Child](glossary.md#child), [Blocking](glossary.md#blocking), [Sprint](glossary.md#sprint), [Release](glossary.md#release).

## Acceptance Criteria

- Given a Work Item of kind `task`, when a user tries to change its `kind` to `defect`, then the request is refused.
- Given an open Epic, when a user creates a Task with that Epic as parent, then the request is refused (BR-WI-014).
- Given a Defect, when a user tries to set a Feature as its parent, then the request is refused (BR-WI-016).
- Given a parentless open Task and an open User Story, when a user sets the User Story as the Task's parent, then the change is accepted.
- Given Issue A blocking Issue B, when a user adds a link by which B blocks A, then the request is refused.
- Given Issue A, when a user adds a link by which A blocks A, then the request is refused.
- Given a User Story with two Tasks, three comments, one tag and one blocking Issue, when the User Story is deleted, then the Tasks become parentless, the comments are deleted, and the tag assignment and the blocking link are removed.
- Given a Work Item with `planstart` later than `planend`, when it is saved, then it is accepted.
- Given a closed Task assigned to a closed Sprint, when a user deletes it, then the request is refused.
- Given an Epic with no Features that is not blocked by an open Issue, when a user closes it, then it is closed.
