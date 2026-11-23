---
title: Comments
status: draft
last-reviewed: 2026-11-23
related: [work-items.md, audit-log.md, actors-and-roles.md]
---

# Comments

## Overview

Each project has a comments component linked to its Work Items. Comments form the project's historical progression line: a work log.

## Functional Requirements

- FR-CMT-001: The system SHALL store a project's comments in that project's database.
- FR-CMT-002: Every comment SHALL belong to exactly one Work Item of the same project.
- FR-CMT-003: Every comment SHALL have exactly one of the comment types listed in [Comment types](#comment-types).
- FR-CMT-004: WHEN a comment is created, the system SHALL assign it an automatic, sequential, numeric `id` and set its `createddt` to the UTC date and time of creation, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)).
- FR-CMT-005: IF a comment is submitted without `wiid`, `kind` or `text`, THEN the system SHALL reject it.
- FR-CMT-012: `text` SHALL have no functional length limit; its technical maximum is left to the technical specification.
- FR-CMT-013: Wherever a comment's author is shown, the system SHALL show it as FR-IDN-021 states.
- FR-CMT-011: WHEN a comment is created, the system SHALL set its `author` to the acting user. The `author` SHALL NOT be supplied by the request and SHALL NOT be changed afterwards.
- FR-CMT-006: The system SHALL allow a comment to be edited by its author, by a user with the `owner` role in the project, and by a platform administrator.
- FR-CMT-007: The system SHALL allow a comment to be deleted by its author, by a user with the `owner` role in the project, and by a platform administrator.
- FR-CMT-010: The author's rights under FR-CMT-006 and FR-CMT-007 SHALL apply only while the author holds the `member` or `owner` role in the project.
- FR-CMT-008: WHEN a comment is edited, the system SHALL set its `editeddt` to the UTC date and time of the edit, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)).
- FR-CMT-009: WHEN a Work Item is deleted, the system SHALL delete its comments ([work-items.md](work-items.md#deletion)).

## Business Rules

### Comment types

| Type | Definition |
|------|------------|
| `HYPOTHESIS` | Something to verify or evaluate: a probable explanation, or a proposed approach not yet validated. It always requires a decision, even if that decision is to do nothing. |
| `TEST` | Result of tests that were run: what happened and how (for example, all unit tests passed with no failures, or 7 of 100 tests passed, with the errors detailed). |
| `FINDING` | A fact observed or discovered during the work, backed by evidence (from a TEST, investigation, reading code, and so on). |
| `DECISION` | A choice that was made, with the alternatives considered and the justification. |
| `PROGRESS` | Concrete progress on the Work Item's work: what has been done since the last entry. |
| `UPDATE` | A change to the Work Item's scope, requirements, priority or context. It is not work done; that is PROGRESS. |
| `NOTE` | Free context or reference information (links, observations) that does not fit any other type. |
| `QUESTION` | An open question that needs an answer from someone for the work to proceed correctly. The answer goes in another comment (for example DECISION or NOTE). |
| `FOLLOW-UP` | Work identified for later, outside the Work Item's current scope; it may give rise to a new Work Item. |
| `BLOCKER` | Something currently preventing progress on this Work Item, and why. If the blocker needs to be managed as work in its own right, it should give rise to an Issue. |
| `HANDOFF` | Handing the Work Item over to another person, with the current state, context and next steps. |
| `ASSUMPTION` | A premise accepted as true without verification so the work can proceed. Unlike HYPOTHESIS, it is not verified; if it turns out to be false, the work may need to be revisited. |
| `RISK` | A problem that has not happened yet but may happen, with its impact and, if possible, its mitigation. Unlike BLOCKER, it is not yet preventing the work. |
| `REVIEW` | Result of a review of the Work Item's work (code, design, documentation): what was reviewed, observations, and verdict (approved or changes requested). |
| `DEBT` | Existing technical debt identified while working on the Work Item; it may give rise to a Refactoring. |

- BR-CMT-001: The comment types are exactly the fifteen listed above.
- BR-CMT-002: Comment types are a classification only. The system does not track or enforce the follow-ups the definitions mention (for example, that a `HYPOTHESIS` receives a decision or that a `QUESTION` is answered).
- BR-CMT-003: A comment remains attributed to its author after the author is deactivated ([users-and-identity.md](users-and-identity.md#deactivation)).

## Acceptance Criteria

- Given a comment by a `member`, when that member edits it, then `text` changes and `editeddt` is set to the UTC date and time of the edit.
- Given a comment by a user whose role is now `viewer`, when that user tries to edit or delete it, then the request is refused.
- Given a comment by another user, when a `member` tries to delete it, then the request is refused.
- Given a comment by another user, when an `owner` deletes it, then it is deleted.
- Given a comment without `kind`, when it is submitted, then it is rejected.
- Given a `member` creating a comment, then its `author` is that member; when an `owner` later edits the comment, then its `author` is unchanged.

## Inputs / Outputs

| Field | Description | Required |
|-------|-------------|----------|
| `id` | Automatic sequential numeric identifier. | Yes (automatic) |
| `wiid` | `id` of the Work Item the comment belongs to. | Yes |
| `kind` | Comment type. | Yes |
| `text` | Comment content. | Yes |
| `author` | User who created the comment; set by the system, never changed. | Yes (automatic) |
| `createddt` | Date-time of creation, UTC, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)). | Yes (automatic) |
| `editeddt` | Date-time of the last edit, UTC, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)). Empty until the first edit. | Set by the system on edit |

## Glossary References

[Comment](glossary.md#comment), [Comment type](glossary.md#comment-type), [Work log](glossary.md#work-log).
