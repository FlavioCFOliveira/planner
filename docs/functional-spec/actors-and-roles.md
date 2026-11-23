---
title: Actors and roles
status: draft
last-reviewed: 2026-11-23
related: [users-and-identity.md, cli-authentication.md, multi-project.md, comments.md, audit-log.md]
---

# Actors and roles

## Overview

This file defines who uses planner and what each actor may do. How users are identified and deactivated is in [users-and-identity.md](users-and-identity.md); how the CLI acts on a user's behalf is in [cli-authentication.md](cli-authentication.md).

## Actors

| Actor | Description |
|-------|-------------|
| User | A person authenticated through the Identity Provider, holding a role in zero or more projects. |
| Platform administrator | A user with full privileges over the whole platform. There may be several. |
| AI agent | An automated agent that operates the CLI with a token issued to a user. It acts on that user's behalf and holds no identity of its own. |
| Identity Provider | The external system that performs registration and login. |

## Functional Requirements

### Roles

- FR-ACC-001: The system SHALL record, in the central `planner` database, the relationship between users and projects and each user's role in each project.
- FR-ACC-002: A user's permissions in a project SHALL be given by exactly one of the fixed roles `owner`, `member` or `viewer`.
- FR-ACC-003: A user with the `owner` role SHALL be able to do everything a `member` can, and also change the project's settings.
- FR-ACC-004: A user with the `member` role SHALL be able to use the project: maintain Work Items, comments, tags, Sprints and Releases, and move Kanban cards. A `member` SHALL NOT be able to change the project's settings.
- FR-ACC-005: A user with the `viewer` role SHALL be able to consult everything in the project, including its audit log, and SHALL NOT be able to modify anything.
- FR-ACC-008: The project settings SHALL be exactly: membership (which users belong to the project and their roles), Kanban board configuration, and the project description (`desc`, FR-PRJ-010).
- FR-ACC-009: A user with the `owner` role SHALL be able to add users to the project, remove them, and assign their roles.
- FR-ACC-018: Roles SHALL be granted only to users who have a profile, that is, users who have logged in successfully at least once (FR-IDN-017). This applies to FR-ACC-009 and FR-ACC-015.
- FR-ACC-019: Within a project, a user with the `owner` role SHALL be able to search the active (not deactivated) users by `name`, `displayname` or `email`, through the WEBSITE, the WEB API and the CLI, so that they can add users to the project under FR-ACC-009. For each user found, the result SHALL hold the user `id` (FR-IDN-025), which the membership operations of FR-ACC-009 take to designate the user (FR-IDN-026), and the name to display (FR-IDN-021), and nothing else. Deactivated users SHALL NOT appear in the result. Users with the `member` or `viewer` role SHALL NOT be able to search. This search is a project-membership operation, not user management (FR-ACC-007).
- FR-ACC-020: In the search of FR-ACC-019, a user SHALL match WHEN the search term occurs as a substring of the user's `name`, `displayname` or `email`, ignoring differences of letter case and of accents or other diacritics. The term SHALL be trimmed as FR-PLT-006 states; IF the trimmed term is shorter than 2 Unicode code points, THEN the system SHALL refuse the search. The matching users SHALL be ordered by their name to display (FR-IDN-021), ignoring differences of letter case and of accents or other diacritics, with ties broken by ascending user `id` (FR-IDN-025). A search SHALL return the first 20 users in that order, and the result SHALL indicate whether more users matched.
- FR-ACC-014: Every project SHALL have at least one user with the `owner` role. IF a request would remove or demote the last `owner` of a project, by anyone and including the `owner` themselves, THEN the system SHALL refuse it.

### Platform administrator

- FR-ACC-006: The platform administrator SHALL have full privileges on the platform, including in every project.
- FR-ACC-007: The platform administrator SHALL be able to create and delete projects, manage users, assign roles in any project and access every project. Managing users consists of deactivating and reactivating users ([users-and-identity.md](users-and-identity.md#deactivation)) and granting and removing platform administrator status.
- FR-ACC-010: Only platform administrators SHALL be able to create projects.
- FR-ACC-015: WHEN a platform administrator creates a project, the system SHALL require the administrator to name at least one user as `owner` of that project.
- FR-ACC-011: The system SHALL allow several users to be platform administrators at the same time.
- FR-ACC-012: The first platform administrator SHALL be designated by a server configuration value that holds that user's IdP user id.
- FR-ACC-013: A platform administrator SHALL be able to grant platform administrator status to another user and to remove it from any administrator, including themselves.
- FR-ACC-016: IF a request would remove platform administrator status from the last active (not deactivated) administrator, including by self-removal, THEN the system SHALL refuse it.
- FR-ACC-017: After the first administrator is created, the server configuration value of FR-ACC-012 SHALL give that user no special treatment: the user is an administrator like any other.

### Permission summary

| Action | `viewer` | `member` | `owner` | Administrator |
|--------|----------|----------|---------|---------------|
| Consult the project | Yes | Yes | Yes | Yes |
| Create, modify, delete, close and reopen Work Items | No | Yes | Yes | Yes |
| Create comments | No | Yes | Yes | Yes |
| Edit or delete a comment ([comments.md](comments.md)) | No | Own comments | Any | Any |
| Read the project audit log | Yes | Yes | Yes | Yes |
| Manage and apply tags | No | Yes | Yes | Yes |
| Manage Sprints and Releases | No | Yes | Yes | Yes |
| Move Kanban cards | No | Yes | Yes | Yes |
| Configure Kanban boards (setting) | No | No | Yes | Yes |
| Manage membership and roles (setting), keeping at least one `owner` | No | No | Yes | Yes |
| Search active users to add as members (FR-ACC-019) | No | No | Yes | Yes |
| Edit the project description (setting) | No | No | Yes | Yes |
| Edit one's own profile (WEBSITE only) | Yes | Yes | Yes | Yes |
| Edit another user's profile (WEBSITE only) | No | No | No | Yes |
| Create and delete projects | No | No | No | Yes |
| Manage users and administrator status; read the central audit log | No | No | No | Yes |

## Business Rules

- BR-ACC-001: Roles are fixed; no other per-project role exists.
- BR-ACC-002: Every change a user makes, through the WEBSITE or the CLI, is attributed to that user.
- BR-ACC-003: Wherever this specification grants a permission to a project role, platform administrators also hold it in every project (FR-ACC-006). Through the CLI, an administrator's permissions are bounded by the token grant (FR-AUTH-022).

## Acceptance Criteria

- Given a project with one `owner`, when that owner demotes themselves to `member`, then the request is refused.
- Given a project with two owners, when one demotes themselves to `member`, then the change is accepted.
- Given a project with one `owner`, when an administrator removes that owner from the project, then the request is refused.
- Given an administrator creating a project without naming an `owner`, when the request is submitted, then it is refused.
- Given a platform with one administrator, when that administrator removes their own status, then the request is refused.
- Given one active administrator and one deactivated administrator, when the active one removes their own status, then the request is refused.
- Given a user with the `member` role, when they change a Kanban board's columns, then the request is refused.
- Given a person who has never logged in, when an `owner` tries to give them a role, then the request is refused.
- Given an active user with `displayname` `ana` and a deactivated user with `name` `Ana Costa`, when an `owner` of a project searches for `ana` through the CLI, then the result holds the active user's user `id` and `ana`, and does not hold the deactivated user.
- Given a user with the `member` role in a project, when they search users through the WEB API, then the request is refused.
- Given an active user with `name` `José Ribeiro`, when an `owner` searches for `JOSE` or for `ribe`, then the result holds that user.
- Given an `owner`, when they search for ` a ` (one letter between spaces), then the search is refused.
- Given 30 active users whose `email` contains `example`, when an `owner` searches for `example`, then the result holds the first 20 of them in order of name to display and indicates that more users matched; given 20 such users, the result holds all 20 and indicates that no more users matched.
- Given active users whose names to display are `bruno`, `Álvaro` and `ana`, all matching the term, then the result lists them in the order `Álvaro`, `ana`, `bruno`.

## Glossary References

[Role](glossary.md#role), [Platform administrator](glossary.md#platform-administrator), [Project settings](glossary.md#project-settings), [AI agent](glossary.md#ai-agent), [Deactivated user](glossary.md#deactivated-user), [Display name](glossary.md#display-name).
