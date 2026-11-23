---
title: Users and identity
status: draft
last-reviewed: 2026-11-23
related: [actors-and-roles.md, cli-authentication.md, multi-project.md]
---

# Users and identity

## Overview

planner delegates registration and login to an external Identity Provider and keeps its own user profile.

## Functional Requirements

### Delegated authentication

- FR-IDN-001: The system SHALL delegate user registration and login to an external Identity Provider using OpenID Connect (OIDC).
- FR-IDN-002: The system SHALL NOT perform user registration or login itself.

### User profile

- FR-IDN-003: The system SHALL keep a profile for each user, keyed by the user identifier provided by the Identity Provider.
- FR-IDN-017: WHEN a user logs in successfully for the first time, the system SHALL create the user's profile.
- FR-IDN-004: WHEN a user profile is created, the system SHALL copy the user's `name` and `email` from the Identity Provider.
- FR-IDN-023: The values copied from the Identity Provider under FR-IDN-004 SHALL be accepted without validation, even when they do not satisfy FR-IDN-022 or are absent, with one exception: a value longer than 64 characters SHALL be truncated to its first 64 characters, counted in Unicode code points. An absent value SHALL remain absent. FR-IDN-022 SHALL apply only to edits made in planner. Data supplied by the Identity Provider SHALL never block a login.
- FR-IDN-005: After the profile is created, the system SHALL NOT read the Identity Provider's profile data again.
- FR-IDN-006: The profile SHALL hold exactly two planner-only fields, both optional: `photo` and `displayname`. The `displayname` is a planner display name, distinct from the `name` copied from the Identity Provider.
- FR-IDN-021: Wherever the system shows a user (in the WEBSITE, in CLI output, and as a comment's author), it SHALL show the user's `displayname` when set, and the user's `name` when `displayname` is empty.
- FR-IDN-022: The profile `name`, when set, SHALL be 1 to 64 characters of any Unicode text. The profile `displayname`, when set, SHALL be 1 to 200 characters of any Unicode text. The `email` SHALL be 1 to 64 characters and SHALL be a valid e-mail address as defined by the WHATWG HTML Standard ("valid e-mail address").
- FR-IDN-007: The system SHALL allow a profile's fields to be edited by the user themself and by platform administrators, only through the WEBSITE.
- FR-IDN-008: WHEN a profile field is edited, the system SHALL change only the planner profile and SHALL NOT propagate the change to the Identity Provider.
- FR-IDN-009: The system SHALL manage users in the central `planner` database.

### Deactivation

- FR-IDN-010: The system SHALL allow a platform administrator to deactivate a user.
- FR-IDN-011: The system SHALL NOT allow users to be deleted.
- FR-IDN-018: IF a request would deactivate a user who is the last `owner` of any project, THEN the system SHALL refuse it until another `owner` exists in each such project (FR-ACC-014).
- FR-IDN-019: WHEN a platform administrator is deactivated, the system SHALL keep the user's platform administrator status.
- FR-IDN-020: IF a request would deactivate the last active (not deactivated) platform administrator, THEN the system SHALL refuse it.
- FR-IDN-012: WHEN a user is deactivated, the system SHALL revoke all of the user's tokens and remove all of the user's project roles.
- FR-IDN-013: WHEN a user is deactivated, the comments and audit log entries attributed to the user SHALL remain attributed to the user.
- FR-IDN-014: WHILE a user is deactivated, the system SHALL NOT allow the user to log in to the WEBSITE or to issue tokens.
- FR-IDN-015: The system SHALL allow a platform administrator to reactivate a deactivated user.
- FR-IDN-016: WHEN a user is reactivated, the system SHALL NOT restore the user's previous project roles or tokens.

## Business Rules

- BR-IDN-001: The only link between a planner profile and the Identity Provider is the user's IdP id.

## Inputs / Outputs

User profile:

| Field | Description | Source |
|-------|-------------|--------|
| IdP user id | Identifier provided by the Identity Provider; key of the profile. | IdP |
| `name` | User name. | Copied from the IdP once, at profile creation; editable in planner. |
| `email` | User email. | Copied from the IdP once, at profile creation; editable in planner. |
| `photo` | User photo. | planner only. |
| `displayname` | Optional planner display name, distinct from `name`; shown instead of `name` when set. | planner only. |

## Constraints

- The Identity Provider vendor is not yet defined ([OQ-013](open-questions.md#oq-013)).

## Acceptance Criteria

- Given a user with two tokens and roles in two projects, when an administrator deactivates the user, then both tokens are revoked and both roles are removed.
- Given a deactivated user, when they try to log in to the WEBSITE, then login is refused.
- Given a reactivated user who previously held the `member` role in a project, when they log in, then they hold no role in that project.
- Given a user whose `name` was copied from the IdP, when the name is edited in planner, then the IdP is not changed.
- Given a person who has never logged in, then no profile exists for them; when they log in successfully for the first time, then their profile is created.
- Given a user who is the only `owner` of a project, when an administrator deactivates the user, then the request is refused.
- Given a deactivated administrator, then the user still holds administrator status and cannot log in.
- Given a user, when an AI agent tries to edit the user's profile through the CLI, then the request is refused.
- Given a user with `name` `Ana Silva` and `displayname` `ana`, then comments by the user show `ana`; when `displayname` is cleared, then they show `Ana Silva`.
- Given a user, when they set a `displayname` of 201 characters, then the request is refused.
- Given a user, when they set a `name` of 65 characters, or a valid `email` of 65 characters, then the request is refused; when they set a `name` of 64 characters, then it is accepted.
- Given a person whose IdP `name` is 70 characters long and whose IdP `email` is absent, when they log in for the first time, then the login succeeds, the profile `name` holds the first 64 characters of the IdP `name`, and the profile `email` is absent.

## Glossary References

[Identity Provider](glossary.md#identity-provider-idp), [User profile](glossary.md#user-profile), [Deactivated user](glossary.md#deactivated-user).

## Open Questions

See [open-questions.md](open-questions.md): OQ-013 (IdP vendor).
