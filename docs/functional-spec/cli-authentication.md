---
title: CLI authentication and DEVICE-ID
status: draft
last-reviewed: 2026-11-23
related: [cli.md, actors-and-roles.md, users-and-identity.md, audit-log.md]
---

# CLI authentication and DEVICE-ID

## Overview

The CLI authenticates with a token issued on the WEBSITE on behalf of the authenticated user. Each token is bound to one device, identified by a `DEVICE-ID`, and grants per-project permissions. The `DEVICE-ID` also records the origin of every change in the audit log.

## Functional Requirements

### Tokens

- FR-AUTH-001: The system SHALL allow an authenticated user to issue, on the WEBSITE, a token that the CLI uses to act on that user's behalf.
- FR-AUTH-025: WHEN a token is issued, the system SHALL set its `issueddt` to the UTC date and time of issue, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)).
- FR-AUTH-002: Every token SHALL have an `expiresdt`, a date-time in UTC persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)), equal to `issueddt` plus the chosen expiry; the default expiry is 30 days. No token SHALL be issued without an `expiresdt`.
- FR-AUTH-003: The system SHALL allow the user to choose a different expiry when issuing the token, as a whole number of days from 1 to 365; `expiresdt` is then `issueddt` plus that number of days. IF the requested expiry is outside that range, THEN the system SHALL refuse to issue the token.
- FR-AUTH-026: A token SHALL be expired from the moment the current UTC date-time reaches its `expiresdt`. Neither `issueddt` nor `expiresdt` SHALL be editable.
- FR-AUTH-004: The system SHALL allow the user to revoke a token at any time.
- FR-AUTH-005: The system SHALL allow a user to have several active tokens at the same time and to revoke each one individually.
- FR-AUTH-017: Tokens SHALL be issued, listed and revoked only on the WEBSITE.

### Permissions

- FR-AUTH-006: WHEN a token is issued, the system SHALL require the user to choose, for each project the token covers, one permission level: `read` or `read+write`.
- FR-AUTH-007: A token SHALL have only the permissions granted to it at issue.
- FR-AUTH-018: The `read` level SHALL allow only operations that do not change state. The `read+write` level SHALL allow every operation in that project.
- FR-AUTH-008: WHEN a request is made with a token, the system SHALL limit the token's permissions in the project to what the user's role in that project allows at the time of the request. A user with the `viewer` role is limited to `read`; a user with no role in the project has no permission in it.
- FR-AUTH-022: WHERE the token's user is a platform administrator at the time of the request, the token's permissions in a project SHALL be exactly its grant for that project, whether or not the administrator holds a role in it.
- FR-AUTH-019: Tokens SHALL carry per-project permissions only. A token SHALL NOT carry platform-level permissions.
- FR-AUTH-020: WHEN a project is deleted, the system SHALL remove every token permission for that project ([multi-project.md](multi-project.md)).

### Other WEB API clients

- FR-AUTH-024: Clients of the WEB API other than the CLI SHALL be allowed, under exactly the rules this file states for the CLI: a token issued on the WEBSITE and bound to the client's own `DEVICE-ID` (FR-AUTH-023), per-project permissions, the `DEVICE-ID` on every request, and no access to the operations excluded by FR-CLI-006.

### Device binding

- FR-AUTH-009: WHEN a token is issued, the system SHALL require the `DEVICE-ID` the token is for.
- FR-AUTH-010: WHEN the server receives a request with a token, it SHALL check that the request's `DEVICE-ID` matches the `DEVICE-ID` the token was issued for.
- FR-AUTH-011: IF a token is expired, revoked, or presented with a non-matching `DEVICE-ID`, THEN the server SHALL reject the request.

### DEVICE-ID

- FR-AUTH-012: WHEN the CLI is installed and no `DEVICE-ID` exists on the device, the CLI SHALL generate one.
- FR-AUTH-021: WHEN the CLI is installed and a `DEVICE-ID` already exists on the device, the CLI SHALL reuse it, so that tokens issued for that `DEVICE-ID` remain valid.
- FR-AUTH-023: IF a `DEVICE-ID` supplied when issuing a token does not match `[A-Za-z0-9]{24}`, THEN the system SHALL refuse to issue the token. As a consequence, no token can be bound to `GUI`.
- FR-AUTH-013: The CLI-generated `DEVICE-ID` SHALL be a random string of 24 characters drawn from uppercase letters, lowercase letters and digits, matching `[A-Za-z0-9]{24}`.
- FR-AUTH-014: The CLI SHALL send its `DEVICE-ID` with every request to the WEB API.
- FR-AUTH-015: Changes made through the WEBSITE SHALL use the fixed, reserved `DEVICE-ID` `GUI`, the same for all users.
- FR-AUTH-016: The CLI SHALL NOT use `GUI` as its `DEVICE-ID`.

## Non-Functional Requirements

- NFR-SEC-001: Token validity (expiry and revocation) and token permissions SHALL always be checked server-side.

## Business Rules

- BR-AUTH-001: Every change, whether from the CLI or the WEBSITE, has a `DEVICE-ID`.
- BR-AUTH-002: Every change made through the CLI remains attributed to the user to whom the token was issued.
- BR-AUTH-003: The `DEVICE-ID` is recorded only in the project's audit log, never as a field of Work Items, comments or other domain entities ([audit-log.md](audit-log.md)).
- BR-AUTH-004: Deactivating a user revokes all of the user's tokens ([users-and-identity.md](users-and-identity.md#deactivation)).

## Use Cases

### UC-AUTH-001: Issue a CLI token

- Actor: user, on the WEBSITE.
- Trigger: the user wants the CLI on a device to act on their behalf.
- Preconditions: the user is authenticated; the CLI is installed on the device and has a `DEVICE-ID`.
- Main flow:
  1. The user supplies the device's `DEVICE-ID`.
  2. For each project the token is to cover, the user chooses `read` or `read+write`.
  3. The user accepts the default expiry of 30 days or sets another.
  4. The system issues the token.
- Alternate flows:
  - 1a. No `DEVICE-ID` is supplied: the system does not issue the token.
- Postconditions: an active token exists, bound to the `DEVICE-ID`, with the chosen permission levels and expiry.

Acceptance criteria:

- Given a token issued for device `A`, when a request presents that token with device `B`, then the server rejects it.
- Given a token issued with the default expiry, then its `expiresdt` is its `issueddt` plus 30 days; when it is presented after `expiresdt`, then the server rejects it as expired.
- Given a revoked token, when it is presented, then the server rejects it.
- Given a `read+write` token for a project where the user's role was later changed to `viewer`, when the token is used to modify a Work Item, then the request is rejected.
- Given a `read+write` token held by a `member`, when it is used to configure a Kanban board, then the request is rejected, because board configuration requires the `owner` role.
- Given a device with an existing `DEVICE-ID` and a valid token, when the CLI is reinstalled, then the token remains valid.
- Given an administrator with no role in project `p` and a `read+write` token for `p`, when the token is used to modify a Work Item in `p`, then the request is accepted.
- Given a deactivated user, when they try to issue a token, then the request is refused.
- Given a user issuing a token, when they request an expiry of 366 days or of 1.5 days, then the token is not issued.
- Given a third-party client with its own `DEVICE-ID` and a token bound to it, when it calls the WEB API to create a Work Item within the token's permissions, then the request is accepted.
- Given a third-party client, when it tries to create a project, then the request is refused.

## Inputs / Outputs

Token:

| Field | Description | Required |
|-------|-------------|----------|
| `issueddt` | Date-time of issue, UTC, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)); set by the system. | Yes (automatic) |
| `expiresdt` | Expiry date-time, UTC, persisted in UTC ([NFR-DATA-001](overview.md#non-functional-requirements)); `issueddt` plus a whole number of days from 1 to 365 (default 30). | Yes |
| `DEVICE-ID` | The device the token is bound to (FR-AUTH-009). | Yes |
| Permissions | One level, `read` or `read+write`, per project covered (FR-AUTH-006). | Yes |
| Revocation state | Whether the token has been revoked (FR-AUTH-004). | Yes |

## Glossary References

[Token](glossary.md#token), [`DEVICE-ID`](glossary.md#device-id), [Role](glossary.md#role).
