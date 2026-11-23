---
title: Command-line interface
status: draft
last-reviewed: 2026-11-23
related: [cli-authentication.md, project-views.md, kanban-boards.md, overview.md]
---

# Command-line interface

## Overview

The CLI is the client application of the WEB API ([overview.md](overview.md)). It is entirely oriented towards AI agents, such as Claude Code, which are its primary user. Authentication is specified in [cli-authentication.md](cli-authentication.md).

## Actors

AI agents (primary), operating on behalf of the user to whom the token was issued.

## Functional Requirements

- FR-CLI-001: The CLI SHALL provide the same project views as the WEBSITE: the Work Item list, Kanban boards and the Gantt chart, with the same Sprint and Release filters ([project-views.md](project-views.md)).
- FR-CLI-002: The CLI SHALL present its output as JSON.
- FR-CLI-003: The CLI SHALL act on behalf of the user to whom its token was issued, including when an AI agent drives it.
- FR-CLI-004: Every request the CLI sends to the WEB API SHALL carry its `DEVICE-ID` ([cli-authentication.md](cli-authentication.md#device-id)).
- FR-CLI-005: The CLI SHALL offer every operation the WEBSITE offers, within the permissions of its token, except the operations excluded by FR-CLI-006.
- FR-CLI-006: The CLI SHALL NOT offer: issuing, listing or revoking tokens; creating or deleting projects; deactivating or reactivating users; granting or removing platform administrator status; editing user profiles; reading the central audit log.

## Non-Functional Requirements

- NFR-USE-001: The whole design of the CLI, from command structure to the organisation of its help, SHALL be optimised for operation by AI agents.
- NFR-USE-002: Every CLI design decision SHALL treat AI agents as the primary user.

## Business Rules

- BR-CLI-001: The closing rules apply to actions through the CLI ([closing-rules.md](closing-rules.md)).

## Glossary References

[CLI](glossary.md#cli), [AI agent](glossary.md#ai-agent), [Token](glossary.md#token).
