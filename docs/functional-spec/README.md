---
title: planner functional specification
status: draft
last-reviewed: 2026-11-23
related: []
---

# planner functional specification

## Overview

This folder is the single source of truth for what planner does: its requirements, business rules, actors, use cases and vocabulary. It states *what* the system does, not *how* it is built. Technical design, architecture decisions and code are out of scope.

The specification was bootstrapped on 2026-11-23 from the functional description then held in the repository's `README.md` and `CLAUDE.md`. Every requirement here traces either to that description or to a decision the user made on 2026-11-23 in answer to the open questions raised during and after the bootstrap. Remaining gaps are recorded in [open-questions.md](open-questions.md), not resolved.

## Index

| File | Purpose |
|------|---------|
| [overview.md](overview.md) | Product goal, scope, interfaces and technology constraints. |
| [glossary.md](glossary.md) | Definitions of every domain term used in the specification. |
| [actors-and-roles.md](actors-and-roles.md) | Actors, per-project roles and the platform administrator. |
| [users-and-identity.md](users-and-identity.md) | OIDC delegation and the planner user profile. |
| [work-items.md](work-items.md) | Work Item types, base data structure, hierarchy and Issue blocking. |
| [closing-rules.md](closing-rules.md) | When a Work Item may be closed, and what closing and reopening do. |
| [tags.md](tags.md) | Simple and scoped tags, their fields and uniqueness rules. |
| [comments.md](comments.md) | The per-project work log: comment fields and comment types. |
| [sprints.md](sprints.md) | Sprints: fields, states, Work Item assignment, closing rule. |
| [releases.md](releases.md) | Releases: fields, Work Item assignment, release rule. |
| [project-views.md](project-views.md) | Work Item list and Gantt chart views, Sprint and Release filters. |
| [kanban-boards.md](kanban-boards.md) | Boards, columns and card-move semantics. |
| [audit-log.md](audit-log.md) | Project and platform audit logs. |
| [cli.md](cli.md) | The AI-agent-oriented command-line interface. |
| [cli-authentication.md](cli-authentication.md) | CLI tokens and the `DEVICE-ID`. |
| [multi-project.md](multi-project.md) | Multiple projects and the central platform database. |
| [open-questions.md](open-questions.md) | Unresolved gaps and decisions pending. |

## Conventions

### Requirement language

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY and OPTIONAL are to be interpreted as described in RFC 2119 and RFC 8174 when, and only when, they appear in all capitals.

Functional requirements use the EARS patterns: "The system SHALL ..."; "WHEN <event>, the system SHALL ..."; "WHILE <state>, ..."; "IF <condition>, THEN ..."; "WHERE <feature>, ...". Acceptance criteria use Given/When/Then.

### Identifiers

Each requirement, rule, use case and open question has a stable identifier. Identifiers are never renumbered or reused; a withdrawn item keeps its identifier and is marked deprecated.

| Prefix | Meaning | Format |
|--------|---------|--------|
| `FR` | Functional requirement | `FR-<MODULE>-<NNN>` |
| `NFR` | Non-functional requirement or constraint | `NFR-<CATEGORY>-<NNN>` |
| `BR` | Business rule | `BR-<MODULE>-<NNN>` |
| `UC` | Use case | `UC-<MODULE>-<NNN>` |
| `OQ` | Open question | `OQ-<NNN>` |

Module codes:

| Code | Area | File |
|------|------|------|
| `PLT` | Platform interfaces | [overview.md](overview.md) |
| `ACC` | Access control (roles, administrator) | [actors-and-roles.md](actors-and-roles.md) |
| `IDN` | Identity and user profile | [users-and-identity.md](users-and-identity.md) |
| `WI` | Work Items | [work-items.md](work-items.md) |
| `CLS` | Closing and reopening | [closing-rules.md](closing-rules.md) |
| `TAG` | Tags | [tags.md](tags.md) |
| `CMT` | Comments | [comments.md](comments.md) |
| `SPR` | Sprints | [sprints.md](sprints.md) |
| `REL` | Releases | [releases.md](releases.md) |
| `VIEW` | Project views (list, Gantt) | [project-views.md](project-views.md) |
| `KBN` | Kanban boards | [kanban-boards.md](kanban-boards.md) |
| `AUD` | Audit log | [audit-log.md](audit-log.md) |
| `CLI` | Command-line interface | [cli.md](cli.md) |
| `AUTH` | CLI authentication and `DEVICE-ID` | [cli-authentication.md](cli-authentication.md) |
| `PRJ` | Projects and tenancy | [multi-project.md](multi-project.md) |

NFR categories: `ARCH` (architecture and technology constraints, in [overview.md](overview.md)), `DATA` (data persistence constraints, in [overview.md](overview.md)), `SEC` (security, in [cli-authentication.md](cli-authentication.md)), `USE` (usability, in [cli.md](cli.md)).

### Inviolable requirements

A requirement marked `(INVIOLABLE)` immediately after its identifier (for example `NFR-DATA-001 (INVIOLABLE)`) uses MUST, admits no exception, and prevails over every other requirement that conflicts with it. It can be changed or withdrawn only by an explicit decision of the user.

### File structure

Each file carries YAML frontmatter with `title`, `status` (`draft` | `approved` | `deprecated`), `last-reviewed` and `related`. Sections follow this order and are omitted when empty: Overview, Scope, Actors, Functional Requirements, Non-Functional Requirements, Business Rules, Use Cases, Acceptance Criteria (for rules not covered by a use case), Inputs / Outputs, Dependencies, Constraints, Glossary References, Open Questions.

Field names of data structures (for example `planstart`, `closeddt`) are written in code format and are part of the specification. Example values in the source description (for example `prioridade::1-critica`) are kept verbatim.

## Status

All files are `draft`. Open questions are listed in [open-questions.md](open-questions.md).
