---
title: Overview
status: draft
last-reviewed: 2026-11-23
related: [glossary.md, multi-project.md, cli.md, sprints.md, releases.md]
---

# Overview

## Overview

planner is a simple platform for software agile management, from planning to release. Its goal is to manage the entire software development lifecycle, drawing inspiration from agile methodologies such as Scrum and Kanban.

The central element is the [Work Item](work-items.md). Work Items are classified with [tags](tags.md), documented through a per-project work log of [comments](comments.md), and presented through [project views](project-views.md) and [Kanban boards](kanban-boards.md). Every state change is recorded in an [audit log](audit-log.md).

## Scope

In scope:

- multiple, isolated projects ([multi-project.md](multi-project.md));
- Work Items of eight types, their hierarchy, blocking and closing rules;
- Sprints and Releases ([sprints.md](sprints.md), [releases.md](releases.md));
- tags, comments, project views (list, Kanban boards, Gantt chart);
- users authenticated through an external Identity Provider, per-project roles and a platform administrator;
- a command-line interface operated primarily by AI agents, authenticated by device-bound tokens;
- audit logs at project and platform level.

## Functional Requirements

- FR-PLT-001: The system SHALL offer three interfaces: the WEBSITE (GUI), the WEB API and the CLI.
- FR-PLT-002: The WEBSITE SHALL be the graphical user interface of the platform.
- FR-PLT-003: The WEB API SHALL be the application integration interface of the platform. Clients other than the CLI MAY use it under the rules of [cli-authentication.md](cli-authentication.md#other-web-api-clients).
- FR-PLT-004: The CLI SHALL be the client application of the WEB API.

- FR-PLT-005: All date-times SHALL be in UTC. Date-only fields SHALL be interpreted as UTC calendar dates, and "the current date" SHALL be the current UTC date. Persistence in UTC is governed separately by NFR-DATA-001.
- FR-PLT-006: Before validating and storing a text value entered by a user, the system SHALL remove its leading and trailing whitespace. A required text field that is empty after this removal SHALL be treated as missing. An optional text field that is empty after this removal SHALL be treated as absent (cleared) and SHALL NOT be stored as an empty string. Length limits SHALL be counted in Unicode code points on the trimmed value. Values copied from the Identity Provider are exempt (FR-IDN-023).
- FR-PLT-007: The WEBSITE and the CLI SHALL display every date-time in UTC and SHALL mark it as UTC. This is a presentation rule; it does not affect NFR-DATA-001.

## Non-Functional Requirements

- NFR-ARCH-001: The system SHALL follow a client/server architecture. The server is a web application that exposes the WEB API and the WEBSITE; the client is the CLI.
- NFR-ARCH-002: The server and the CLI SHALL be written in Rust.
- NFR-ARCH-003: The WEB API SHALL be a REST interface.
- NFR-ARCH-004: The server SHALL persist its data in MariaDB.
- NFR-DATA-001 (INVIOLABLE): Every date and every date-time MUST be persisted in UTC, regardless of the time zone in which it is presented, with no exception. This applies to every date and date-time field of every entity, present or future, including: Work Item `planstart`, `planend`, `starteddt` and `closeddt`; comment `createddt` and `editeddt`; Sprint `startdt` and `enddt`; Release `plandt` and `releaseddt`; token `issueddt` and `expiresdt`; audit log `createddt` in every project audit log and in the central audit log; and any date or date-time held in user profiles or projects. This requirement holds even if the presentation rules (FR-PLT-007) change.
- NFR-PERF-001: The platform SHALL support at least 500 registered users, at least 100 of them active at the same time, and at least 500 projects.
- NFR-PERF-002: The platform SHALL sustain a peak load of at least 200 requests per second, counting the requests of every client, AI agents included, while NFR-PERF-001 holds.
- NFR-PERF-003: Under the load of NFR-PERF-002, with the users and projects of NFR-PERF-001 and the data volume of NFR-PERF-005, the WEB API SHALL answer every simple operation (NFR-PERF-004) with a 95th-percentile latency below 200 ms. Latency SHALL be measured at the server, from the receipt of a request to the sending of its response, and SHALL exclude network transit to and from the client.
- NFR-PERF-004: A simple operation is any WEB API operation that acts on a single object: reading, creating, modifying, closing, reopening or deleting a single Work Item, comment, tag, Sprint or Release, where that object supports the operation; activating or closing a single Sprint; releasing a single Release; and moving a single card on a Kanban board. List and filter operations are not simple operations; NFR-PERF-003 sets no latency target for them.
- NFR-PERF-005: NFR-PERF-003 SHALL be verified with every project holding 10,000 Work Items, 100,000 comments and 500,000 project audit log entries.

## Glossary References

[Work Item](glossary.md#work-item-wi), [WEBSITE](glossary.md#website-gui), [WEB API](glossary.md#web-api), [CLI](glossary.md#cli), [AI agent](glossary.md#ai-agent), [Kanban board](glossary.md#kanban-board), [Audit log](glossary.md#audit-log).

