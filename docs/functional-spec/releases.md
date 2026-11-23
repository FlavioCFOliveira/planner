---
title: Releases
status: draft
last-reviewed: 2026-11-23
related: [work-items.md, sprints.md, closing-rules.md, project-views.md, actors-and-roles.md, audit-log.md]
---

# Releases

## Overview

A Release is a version of the software that a project delivers. Work Items are assigned to Releases. Releases are independent of [Sprints](sprints.md): the two have no direct link and relate only through the Work Items assigned to them. Release assignments are independent of the Work Item hierarchy (BR-WI-018).

## Functional Requirements

### Releases

- FR-REL-001: Each project SHALL have its own set of Releases.
- FR-REL-002: Every Release SHALL have the fields defined in [Inputs / Outputs](#inputs--outputs).
- FR-REL-010: WHEN a Release is created, the system SHALL assign it an automatic, increasing, numeric `id`, unique among the Releases of the project. Ids SHALL never be reused and MAY have gaps.
- FR-REL-011: IF a Release is submitted without `name`, THEN the system SHALL reject it.
- FR-REL-012: IF a Release is created or edited so that its `name` equals the `name` of another Release of the same project, THEN the system SHALL refuse the request. Two names that differ only in letter case, or only in accents or other diacritics, SHALL be treated as equal (for example, `v1.0` and `V1.0`, or `Versão 1` and `Versao 1`).
- FR-REL-021: `name` SHALL be 1 to 200 characters of any Unicode text. `desc` SHALL have no functional length limit. `plandt` and `releaseddt` SHALL be interpreted as UTC dates (FR-PLT-005) and persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)).
- FR-REL-003: The system SHALL NOT link a Release directly to a Sprint.

### Released state

- FR-REL-004: A Release SHALL be released when its `releaseddt` is set, and unreleased while `releaseddt` is empty.
- FR-REL-013: The `releaseddt` SHALL be entered by the user. Releasing a Release consists of the user setting its `releaseddt`.
- FR-REL-005: IF a request would set `releaseddt` while any Work Item assigned to the Release is open, THEN the system SHALL refuse it.
- FR-REL-014: IF a request would clear the `releaseddt` of a released Release, THEN the system SHALL refuse it.
- FR-REL-017: `plandt` and `releaseddt` SHALL be calendar dates without a time of day.
- FR-REL-018: IF a request would set `releaseddt` to a date later than the current UTC date, THEN the system SHALL refuse it.
- FR-REL-019: A released Release SHALL be fully frozen. IF a request would edit any field of a released Release, including changing `releaseddt` to another date, THEN the system SHALL refuse it.

### Assignment

- FR-REL-006: A Work Item SHALL be assigned to at most one Release, through its optional `release` field, which holds the Release's `id` ([work-items.md](work-items.md)).
- FR-REL-007: Work Items of every kind except `issue` MAY be assigned to a Release. IF a request would assign an Issue to a Release, THEN the system SHALL refuse it.
- FR-REL-015: The assignments of a released Release SHALL be frozen. IF a request would assign a Work Item to a released Release, or remove or move a Work Item out of a released Release, THEN the system SHALL refuse it.

### Deletion

- FR-REL-016: IF a request would delete a Release to which any Work Item is assigned, THEN the system SHALL refuse it.
- FR-REL-020: IF a request would delete a released Release, THEN the system SHALL refuse it, whether or not any Work Item is assigned to it.

### Permissions and audit

- FR-REL-008: Users with the `owner` or `member` role SHALL be able to create, modify and delete Releases, release them, and assign Work Items to them. Users with the `viewer` role SHALL NOT. Release management is not a project setting.
- FR-REL-009: The creation, modification and deletion of a Release SHALL be recorded in the project's audit log with `entity` `release` ([audit-log.md](audit-log.md)).

## Business Rules

- BR-REL-001: A Release can be released only when every Work Item assigned to it is closed.
- BR-REL-002: A released Release is final: it cannot be un-released, edited or deleted, and Work Items cannot be added to it or removed from it. Because its Work Items cannot be reopened (FR-CLS-006), every Work Item of a released Release stays closed.

## Use Cases

### UC-REL-001: Release a Release

- Actor: user with the `owner` or `member` role, through the WEBSITE, or AI agent, through the CLI.
- Trigger: the actor enters a `releaseddt` for an unreleased Release.
- Preconditions: the Release's `releaseddt` is empty.
- Main flow:
  1. The system checks that every Work Item assigned to the Release is closed.
  2. The system stores the entered `releaseddt`.
- Alternate flows:
  - 1a. At least one assigned Work Item is open: the system refuses the request and the Release stays unreleased.
- Postconditions: the Release is released, or unchanged if refused.

Acceptance criteria:

- Given a Release with one open assigned User Story, when a user sets its `releaseddt`, then the request is refused.
- Given a Release whose assigned Work Items are all closed, when a user sets its `releaseddt`, then the Release is released.
- Given a released Release, when a user clears its `releaseddt`, then the request is refused.
- Given a released Release, when a user edits its `desc` or changes its `releaseddt`, then the request is refused.
- Given a released Release with no assigned Work Items, when a user deletes it, then the request is refused.
- Given an unreleased Release whose Work Items are all closed, when a user sets its `releaseddt` to the day after the current UTC date, then the request is refused.
- Given a released Release, when a user assigns a Work Item to it or removes one from it, then the request is refused.
- Given an Issue, when a user assigns it to a Release, then the request is refused.
- Given a Release with one assigned Work Item, when a user deletes the Release, then the request is refused.
- Given a Release named `1.0.0`, when a user creates another Release named `1.0.0` in the same project, then the request is refused.
- Given a Release named `v1.0`, when a user creates another Release named `V1.0` in the same project, or renames another Release to `V1.0`, then the request is refused.
- Given a Release named `Versão 1`, when a user creates another Release named `Versao 1` in the same project, or renames another Release to `Versao 1`, then the request is refused.

## Inputs / Outputs

| Field | Description | Required |
|-------|-------------|----------|
| `id` | Automatic, increasing numeric identifier; never reused; may have gaps; unique among the project's Releases. | Yes (automatic) |
| `name` | Name of the Release, which is its version; unique within the project, ignoring letter case and diacritics (FR-REL-012). | Yes |
| `desc` | Description of the Release. | No |
| `plandt` | Planned release date: calendar date, UTC, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)). | No |
| `releaseddt` | Release date: calendar date, UTC, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)); entered by the user; not later than the current UTC date. Empty while unreleased. | Set by the user on release |

## Dependencies

[work-items.md](work-items.md), [closing-rules.md](closing-rules.md), [project-views.md](project-views.md) (filters and Gantt), [audit-log.md](audit-log.md).

## Glossary References

[Release](glossary.md#release), [Sprint](glossary.md#sprint), [Work Item](glossary.md#work-item-wi).
