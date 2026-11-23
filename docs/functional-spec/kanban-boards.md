---
title: Kanban boards
status: draft
last-reviewed: 2026-11-23
related: [tags.md, closing-rules.md, work-items.md, project-views.md, actors-and-roles.md, cli.md]
---

# Kanban boards

## Overview

Kanban boards arrange a project's Work Items in columns. They behave in every respect like GitLab issue boards. Columns between the first and the last are defined by [tags](tags.md); moving a card changes the Work Item's tags and, at the `closed` column, its open or closed state.

## Functional Requirements

### Boards and columns

- FR-KBN-001: The system SHALL allow a project to have multiple Kanban boards, each configured with its own columns.
- FR-KBN-002: Every board SHALL have the column `open` as its first column and the column `closed` as its last column.
- FR-KBN-003: The system SHALL allow the users named in FR-KBN-016 to choose the intermediate columns of a board, each defined by one tag (for example `open`, `tag1`, `tag2`, `closed`).
- FR-KBN-022: Every board SHALL have a `name` of 1 to 200 characters of any Unicode text. IF a board is created or edited without a `name`, or with the `name` of another board of the same project, THEN the system SHALL refuse the request.
- FR-KBN-023: Users with the `owner` role and platform administrators SHALL set the order of a board's intermediate columns.
- FR-KBN-016: Board configuration (creating, changing and deleting boards and their columns) is a project setting; only users with the `owner` role and platform administrators SHALL be able to perform it ([actors-and-roles.md](actors-and-roles.md)).
- FR-KBN-020: IF a request would give a board two columns defined by the same tag, THEN the system SHALL refuse it.
- FR-KBN-017: WHEN the tag that defines a column is deleted, the system SHALL remove that column from its board ([tags.md](tags.md)). WHEN that tag is edited, the column SHALL remain.

### Column contents

- FR-KBN-004: The `open` column SHALL show the open Work Items (empty `closeddt`) that have none of the board's column tags.
- FR-KBN-005: Each tag column SHALL show every open Work Item that has that column's tag. An open Work Item that has the tags of several columns of the same board SHALL appear in each of those columns.
- FR-KBN-006: The `closed` column SHALL show the closed Work Items.
- FR-KBN-018: A board SHALL be filterable by Sprint and by Release ([project-views.md](project-views.md)).

### Card order

- FR-KBN-015: The system SHALL let users with the `owner` or `member` role, and platform administrators, order cards manually within each column of each board, and SHALL persist that order.
- FR-KBN-021: WHEN a card enters a column, for any reason (a move, a new Work Item, a tag change or a change of state made outside the board), the system SHALL place it at the bottom of that column.

### Moving cards

- FR-KBN-007: The system SHALL allow a card to be moved from any column to any other column. The source column is the column the card is moved from.
- FR-KBN-008: WHEN a card is moved from a tag column to another tag column, the system SHALL remove the source column's tag and add the target column's tag.
- FR-KBN-009: WHEN a card is moved from `open` to a tag column, the system SHALL add the target column's tag.
- FR-KBN-010: WHEN a card is moved from a tag column to `open`, the system SHALL remove the source column's tag.
- FR-KBN-011: WHEN a card is moved from any column to `closed`, the system SHALL remove the source column's tag, if any, and close the Work Item, which sets `closeddt` to the UTC date and time of the move (FR-WI-007).
- FR-KBN-012: WHEN a card is moved from `closed` to any column, the system SHALL reopen the Work Item (clear its `closeddt`) and add the target column's tag, if any.
- FR-KBN-013: IF a card is moved to `closed` and the Work Item cannot be closed under [closing-rules.md](closing-rules.md), THEN the system SHALL refuse the move.
- FR-KBN-014: IF a card is moved out of `closed` and the Work Item cannot be reopened under FR-CLS-002, FR-CLS-003 or FR-CLS-006, THEN the system SHALL refuse the move.
- FR-KBN-019: A move SHALL change only the tags named in FR-KBN-008 to FR-KBN-012. Every other tag of the Work Item, including the tags of other columns of the same board, SHALL remain.

## Business Rules

- BR-KBN-001: The `open` column has no tag. Moving a card removes the tag of the column it was in and adds the tag of the column it moves to.
- BR-KBN-002: Board behaviour follows GitLab issue boards in every respect.
- BR-KBN-003: Because a closed Work Item keeps the tags of columns other than the source column (FR-KBN-019), moving it from `closed` to `open` places it in each tag column whose tag it still has, not in `open` (FR-KBN-004, FR-KBN-005).

## Use Cases

### UC-KBN-001: Move a card

- Actor: user with the `owner` or `member` role, through the WEBSITE, or AI agent, through the CLI.
- Trigger: the actor moves a card from a source column to a target column.
- Preconditions: the board exists and shows the card in the source column.
- Main flow:
  1. The system applies the transition rule for the source and target columns (FR-KBN-008 to FR-KBN-012).
  2. The card appears in the target column.
- Alternate flows:
  - 1a. The target is `closed` and the closing rules are not met: the system refuses the move and nothing changes (FR-KBN-013).
  - 1b. The source is `closed` and the Work Item cannot be reopened: the system refuses the move and nothing changes (FR-KBN-014).
- Postconditions: the Work Item's tags and state match the target column.

Acceptance criteria:

- Given a board `open`, `doing`, `review`, `closed` and an open Work Item tagged `doing`, when its card is moved to `review`, then the Work Item loses `doing` and gains `review`.
- Given an open Work Item with none of the board's column tags, when its card is moved to `doing`, then it gains `doing`.
- Given an open Work Item tagged `review`, when its card is moved to `open`, then it loses `review`.
- Given an open Work Item tagged `doing` that meets the closing rules, when its card is moved to `closed`, then it loses `doing` and its `closeddt` is set to the UTC date and time of the move.
- Given a closed Work Item, when its card is moved to `review`, then its `closeddt` is cleared and it gains `review`.
- Given an open User Story with an open Task, when its card is moved to `closed`, then the move is refused and nothing changes.
- Given an open Work Item tagged `doing` and `review`, then its card appears in both columns; when it is moved from `doing` to `closed`, then it loses `doing`, keeps `review` and is closed.
- Given a closed Task whose parent User Story is closed, when its card is moved to `open`, then the move is refused.
- Given a board with a column defined by `doing`, when the owner adds a second column defined by `doing`, then the request is refused.
- Given a column with three cards, when a fourth card is moved into it, then it is placed at the bottom.
- Given a user with the `member` role, when they add a column to a board, then the request is refused.
- Given a board named `Delivery`, when the owner creates another board named `Delivery` in the same project, then the request is refused.
- Given a board `open`, `doing`, `review`, `closed`, when the owner reorders the intermediate columns, then the board shows `open`, `review`, `doing`, `closed`.

## Inputs / Outputs

Board:

| Field | Description | Required |
|-------|-------------|----------|
| `name` | Name of the board, 1 to 200 characters, unique within the project. | Yes |
| Intermediate columns | Ordered list of tags, each defining one column; no tag twice. | No |

## Dependencies

[tags.md](tags.md), [closing-rules.md](closing-rules.md), [work-items.md](work-items.md#dates).

## Glossary References

[Kanban board](glossary.md#kanban-board), [Card](glossary.md#card), [Tag](glossary.md#tag), [Reopen](glossary.md#reopen).
