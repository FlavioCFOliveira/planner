---
title: Project views
status: draft
last-reviewed: 2026-11-23
related: [work-items.md, kanban-boards.md, sprints.md, releases.md, cli.md]
---

# Project views

## Overview

The WEBSITE offers several ways of viewing a project. The CLI offers the same views for AI agents ([cli.md](cli.md)). Kanban boards are specified in [kanban-boards.md](kanban-boards.md).

## Functional Requirements

- FR-VIEW-001: The WEBSITE SHALL offer three views of a project: the Work Item list, Kanban boards and the Gantt chart.
- FR-VIEW-002: The Work Item list SHALL list the project's Work Items.
- FR-VIEW-003: The Gantt chart SHALL show a timeline of the project.
- FR-VIEW-004: The Gantt chart SHALL show planning from each Work Item's `planstart` and `planend`.
- FR-VIEW-005: The Gantt chart SHALL show history from each Work Item's `starteddt` and `closeddt`.
- FR-VIEW-006: The Work Item list, Kanban boards and the Gantt chart SHALL each be filterable by Sprint and by Release.
- FR-VIEW-007: The Gantt chart SHALL show each Sprint as an interval from its `startdt` to its `enddt`.
- FR-VIEW-008: The Gantt chart SHALL show each Release as a milestone at its `plandt`, when `plandt` is set.
- FR-VIEW-010: The Gantt chart SHALL position every element from the stored UTC values of the fields above ([NFR-DATA-001](overview.md#non-functional-requirements), FR-PLT-005) and SHALL display them in UTC (FR-PLT-007).
- FR-VIEW-009: The Gantt chart SHALL show a second marker for each Release at its `releaseddt`, when `releaseddt` is set.

## Dependencies

[work-items.md](work-items.md) (date fields), [sprints.md](sprints.md), [releases.md](releases.md), [kanban-boards.md](kanban-boards.md), [cli.md](cli.md).

## Glossary References

[Work Item](glossary.md#work-item-wi), [Kanban board](glossary.md#kanban-board), [Sprint](glossary.md#sprint), [Release](glossary.md#release).
