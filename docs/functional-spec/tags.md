---
title: Tags
status: draft
last-reviewed: 2026-11-23
related: [work-items.md, kanban-boards.md, actors-and-roles.md]
---

# Tags

## Overview

Tags classify Work Items. The model follows GitLab labels: simple tags and scoped tags.

## Functional Requirements

- FR-TAG-001: Each project SHALL have its own set of tags.
- FR-TAG-002: Every tag SHALL have a `name`; `desc`, `color` and `value` SHALL be optional.
- FR-TAG-003: The system SHALL support simple tags, which have only a name and no categorisation (for example `frontend`, `backend`, `design`, `qa`).
- FR-TAG-004: The system SHALL support scoped tags, which have a name and a sub-value and are written `name::value` (for example `prioridade::1-critica`, `prioridade::2-alta`, `prioridade::3-baixa`).
- FR-TAG-005: The system SHALL allow a Work Item to have several tags, including several scoped tags with the same name (for example `prioridade::1-critica` and `prioridade::2-alta`).
- FR-TAG-006: IF a tag is applied to a Work Item that already has it, THEN the system SHALL refuse the request.
- FR-TAG-007: IF a tag is created or edited so that its `name` and `value` equal those of another tag in the same project, THEN the system SHALL refuse the request.
- FR-TAG-012: A tag's `name` and `value` SHALL consist only of lowercase ASCII letters `a`–`z`, digits `0`–`9`, hyphen (`-`) and underscore (`_`). As a consequence, neither can contain `::`.
- FR-TAG-013: A tag's `color`, when set, SHALL have the form `#RRGGBB`, where each of `RR`, `GG` and `BB` is two hexadecimal digits.
- FR-TAG-014: A tag's `name` SHALL be 1 to 50 characters. A tag's `value` SHALL be either absent (simple tag) or 1 to 50 characters; it SHALL NOT be an empty string.
- FR-TAG-015: Hexadecimal digits in `color` SHALL be accepted in either letter case; values that differ only in case (for example `#ff0000` and `#FF0000`) denote the same colour.
- FR-TAG-008: The system SHALL allow a simple tag and one or more scoped tags to share the same `name` (for example `prioridade` and `prioridade::1-critica`).
- FR-TAG-009: The system SHALL allow every field of a tag to be edited. An edited tag remains applied to the same Work Items and keeps defining the same board columns.
- FR-TAG-010: WHEN a tag is deleted, the system SHALL remove it from every Work Item and SHALL remove every Kanban board column it defines.
- FR-TAG-011: Users with the `owner` or `member` role SHALL be able to create, edit, delete and apply tags. Users with the `viewer` role SHALL NOT. Tag management is not a project setting.

## Business Rules

- BR-TAG-001: The same tag cannot be applied twice to the same Work Item.
- BR-TAG-002: Within a project, the combination of `name` and `value` is unique. `prioridade::1-critica` and `prioridade::2-alta` are distinct tags; the simple tag `prioridade` is distinct from both; there cannot be two `prioridade::1-critica` tags, nor two simple `frontend` tags.

## Acceptance Criteria

- Given a Work Item tagged `frontend`, when a user applies `frontend` to it again, then the request is refused.
- Given a tag `prioridade::1-critica`, when a user creates another `prioridade::1-critica` in the same project, then the request is refused.
- Given a scoped tag `prioridade::1-critica`, when a user creates the simple tag `prioridade`, then it is accepted.
- Given a tag applied to two Work Items and defining a board column, when a user renames it, then both Work Items keep it and the column remains.
- Given a tag applied to two Work Items and defining a board column, when a user deletes it, then it is removed from both Work Items and the column is removed.
- Given a user with the `viewer` role, when they create a tag, then the request is refused.
- Given a request to create a tag named `Frontend` or `front end`, then it is refused.
- Given a request to create a tag with `color` `#F00`, then it is refused.
- Given a request to create a tag with `color` `#ff0000`, then it is accepted.
- Given a request to create a tag with a 51-character `name`, or with an empty-string `value`, then it is refused.

## Inputs / Outputs

| Field | Description | Required |
|-------|-------------|----------|
| `name` | Tag name. | Yes |
| `desc` | Description of the tag. | No |
| `color` | Display colour, `#RRGGBB` (for example `#FF0000`). | No |
| `value` | Sub-value; present only on scoped tags. | No |

## Dependencies

Kanban board columns are defined by tags ([kanban-boards.md](kanban-boards.md)).

## Glossary References

[Tag](glossary.md#tag), [Simple tag](glossary.md#simple-tag), [Scoped tag](glossary.md#scoped-tag).
