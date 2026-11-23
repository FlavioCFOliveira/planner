---
title: WEB API
status: draft
last-reviewed: 2026-11-23
related: [architecture.md, security.md, cli.md, data-integrity-and-concurrency.md, database-schema.md, ../functional-spec/cli.md, ../functional-spec/cli-authentication.md, ../functional-spec/actors-and-roles.md]
---

# WEB API

## Overview

REST over HTTPS (NFR-ARCH-003), JSON, under `/api/v1`. It serves the CLI and other token clients (FR-PLT-003, FR-AUTH-024). This file fixes the conventions and the resource map, not every field schema: request and response bodies carry the functional field names of each entity. Source ids `[Sn]` refer to the [source register](database-schema.md#sources). Decider key: [README.md](README.md#conventions).

## Decisions

| # | Decision | Rationale | By |
|---|----------|-----------|----|
| W1 | Base path `/api/v1`. | A breaking change gets a new prefix; clients pinned to `v1` keep working. | U |
| W2 | Errors are RFC 9457 problem details [S38] with two extension members: `code` (stable, machine-readable) and `requirement` (the ID the refusal enforces). | Agents branch on `code`, not on prose (NFR-USE-001). Extension members are allowed, and clients ignore those they do not know [S38 §3.2]. | U |
| W3 | Keyset pagination by id: `?after=<id>&limit=N`. | Stable under concurrent inserts; served by primary keys ([index coverage](#index-coverage)). | U |
| W4 | Date-times RFC 3339 UTC with `Z`; dates `YYYY-MM-DD`; colours `#RRGGBB` upper case on output. | FR-PLT-005, NFR-DATA-001, FR-TAG-013, FR-TAG-015; [S39]. | U |
| W5 | `/api/v1` contains exactly the operations FR-CLI-005 gives the CLI. WEBSITE-only operations (FR-CLI-006, FR-AUTH-017, FR-IDN-007) are WEBSITE routes, not API resources. | FR-AUTH-024 holds by construction: no token can reach an excluded operation. | B |
| W6 | Projects are addressed by name in paths. | Names are unique and immutable (FR-PRJ-006, FR-PRJ-007) and are what agents know. | B |

## Authentication

Every request carries `Authorization: Bearer <token>` and `X-Device-Id` ([security.md](security.md#cli-tokens)). Cookies are ignored. Every response carries `X-Request-Id` ([operations.md](operations.md#observability)).

## Representation

| Aspect | Rule | By |
|--------|------|----|
| Body | `application/json`, UTF-8. Errors `application/problem+json` [S38 §3]. | U |
| Field names | The functional names (`kind`, `title`, `desc`, `parent`, `planstart`, `closeddt`, `sprint`, `release`, `wiid`, `author`, …). Column differences are internal ([domain-model.md](domain-model.md)). | B |
| Ids | JSON numbers. | B |
| Date-time | `YYYY-MM-DDTHH:MM:SSZ`, whole seconds [S39 §5.6]. Input with another offset or with a fraction is refused (400), so no value is silently converted or truncated. | Format: U. Refusal: B |
| Date | `YYYY-MM-DD` (`full-date`) [S39 §5.6]. | U |
| Colour | Input `#RRGGBB` in either case; output upper case. | U |
| Absent optional value | `null`, never `""` (FR-PLT-006). | B |
| Update | `PATCH`: an omitted member is unchanged; `null` clears an optional member. Text is trimmed before validation (FR-PLT-006). System fields (`id`, `closeddt`, `author`, `createddt`, `editeddt`) are refused as input (FR-WI-013, FR-CMT-011). | B |
| Users | A user is designated by the user `id` (`users.id`, FR-IDN-025, FR-IDN-026): member `user`, comment `author`, audit `user`, search results. Wherever a user appears, the body gives `id` and `display` (FR-IDN-021). The IdP subject (`idp_subject`) is never returned by any `/api/v1` resource (FR-IDN-027). | U |

## Pagination

| Aspect | Rule | By |
|--------|------|----|
| Parameters | `after` (an id, exclusive), `limit`, `order` (`asc` default, `desc`). With `desc`, `after` returns ids lower than it. | Parameters: U. `order`: B |
| Limit | Default 50, maximum 200. A larger value is refused (400). | B |
| Response | `{"items": [...], "next": <id or null>}`. `next` is the last id of a full page, `null` on a shorter page. | B |
| Scope | Every list resource, except the Gantt view (the timeline needs every element) and user search (first 20 results and a `more` flag, FR-ACC-020). Board columns use their own cursor ([below](#board-column-pages)). | B |

### Board column pages

Cards are ordered by `(position, work_item_id)` (FR-KBN-015), not by id. (U)

- `GET /projects/{p}/boards/{id}` returns the board's columns, each with its first page of cards and its `next`.
- `GET /projects/{p}/boards/{id}/columns/{column_id}/cards?after=<wiid>&limit=N` returns the following page. The server reads the card `(column_id, after)` to obtain its `(position, work_item_id)` and returns the cards after it in that order (`ix_board_cards_order`). `limit` follows [Pagination](#pagination).
- If the card named by `after` is no longer in the column, the request is refused with 409 `BOARD_CURSOR_STALE` (B); the client restarts from the first page.
- Sprint and Release filters apply to every page.

## Errors

Problem members: `type` omitted (it then means `about:blank` [S38 §3.1]), `title` (HTTP reason phrase), `status`, `detail` (one human sentence), `code`, `requirement` (when a requirement applies), `request_id`, and for validation `errors`: `[{"field", "code", "requirement"}]`. The CLI maps statuses to exit codes ([cli.md](cli.md#exit-codes)). Mapping and codes: B, within the user's decision W2.

### Status mapping

| Situation | Status | `code` |
|-----------|--------|--------|
| Malformed JSON, unknown member, bad parameter | 400 | `INVALID_REQUEST` |
| Field rule (type-level **T**), CHECK violation 4025 or out-of-range 1264 [S26] | 400 | `VALIDATION_FAILED` |
| Missing or invalid token, device mismatch (FR-AUTH-011) | 401 | `TOKEN_MISSING`, `TOKEN_INVALID`, `TOKEN_EXPIRED`, `TOKEN_REVOKED`, `DEVICE_MISMATCH` |
| Insufficient level or role | 403 | `FORBIDDEN` |
| Token request not received over HTTPS | 403 | `HTTPS_REQUIRED` |
| Project not visible to the caller (no `read`), or entity absent | 404 | `NOT_FOUND`. A project the caller cannot read is indistinguishable from a missing one, so project names do not leak. |
| Business-rule refusal ([below](#business-rule-codes)) | 409 | rule code |
| Duplicate key 1062 [S26] | 409 | code of the unique index ([below](#business-rule-codes)), else `DUPLICATE` |
| FK `RESTRICT` 1451 [S26] | 409 | `SPR_HAS_ITEMS` / `REL_HAS_ITEMS` by FK name, else `IN_USE` |
| Referenced row missing 1452 [S26] | 422 | `REFERENCE_NOT_FOUND` |
| Rate limit | 429 | `RATE_LIMITED`, with `Retry-After` [S42] |
| Deadlock 1213, snapshot conflict 1020 or lock wait timeout 1205 [S26], after the L4 retries | 503 | `RETRY_EXHAUSTED`, with `Retry-After: 1` |
| No pool connection within 2 s (DA7) | 503 | `POOL_EXHAUSTED`, with `Retry-After: 1` (code: B) |
| Auto-increment exhausted: error 167, SQLSTATE 22003 [S48] ([database-schema.md](database-schema.md#decisions) D2) | 500 | `ID_SPACE_EXHAUSTED` |
| Any other failure | 500 | `INTERNAL` |

### Business-rule codes

| Requirement | `code` | Raised by |
|-------------|--------|-----------|
| BR-CLS-001 / FR-CLS-001 | `CLS_OPEN_CHILDREN` | service |
| BR-CLS-002 / FR-CLS-001 | `CLS_BLOCKED_BY_OPEN_ISSUE` | service |
| FR-CLS-002 | `CLS_PARENT_CLOSED` | service |
| FR-CLS-003 | `CLS_BLOCKS_CLOSED_ITEM` | service |
| FR-CLS-004 | `CLS_LINK_TO_CLOSED_ITEM` | service |
| FR-CLS-005 | `CLS_NEW_PARENT_CLOSED` | service |
| FR-CLS-006 | `CLS_REOPEN_FROZEN` | service |
| FR-WI-010 | `WI_KIND_IMMUTABLE` | service |
| FR-WI-016 | `WI_PARENT_NOT_ALLOWED` | type, CHECK |
| FR-WI-009 | `WI_BLOCKER_NOT_ISSUE`, `WI_SELF_BLOCK` | type, CHECK |
| FR-WI-017 (link already present) | `WI_LINK_EXISTS` | PK of `blockings` |
| FR-WI-023 | `WI_BLOCKING_CYCLE` | service |
| FR-WI-024 | `WI_DELETE_FROZEN` | service |
| FR-SPR-007 | `SPR_INVALID_TRANSITION` | service |
| FR-SPR-008 | `SPR_ALREADY_ACTIVE` | `uq_sprints_one_active` |
| FR-SPR-009 | `SPR_OPEN_ITEMS` | service |
| FR-SPR-011 | `SPR_KIND_NOT_ALLOWED` | type, CHECK |
| FR-SPR-016 | `SPR_NAME_TAKEN` | `uq_sprints_name` |
| FR-SPR-020 | `SPR_ASSIGNMENT_FROZEN` | service |
| FR-SPR-021 | `SPR_HAS_ITEMS` | FK `fk_work_items_sprint` |
| FR-SPR-022 | `SPR_FROZEN` | service |
| FR-SPR-023 | `SPR_DELETE_CLOSED` | service |
| FR-REL-005 | `REL_OPEN_ITEMS` | service |
| FR-REL-007 | `REL_KIND_NOT_ALLOWED` | type, CHECK |
| FR-REL-012 | `REL_NAME_TAKEN` | `uq_releases_name` |
| FR-REL-014 | `REL_UNRELEASE` | service |
| FR-REL-015 | `REL_ASSIGNMENT_FROZEN` | service |
| FR-REL-016 | `REL_HAS_ITEMS` | FK `fk_work_items_release` |
| FR-REL-018 | `REL_DATE_IN_FUTURE` | service |
| FR-REL-019 | `REL_FROZEN` | service |
| FR-REL-020 | `REL_DELETE_RELEASED` | service |
| FR-TAG-006 | `TAG_ALREADY_APPLIED` | PK of `work_item_tags` |
| FR-TAG-007 | `TAG_DUPLICATE` | `uq_tags_name_value` |
| FR-KBN-020 | `KBN_DUPLICATE_COLUMN` | `uq_board_columns_tag` |
| FR-KBN-022 | `KBN_NAME_TAKEN` | `uq_boards_name` |
| FR-KBN-013, FR-KBN-014 | the `CLS_*` code of the failed rule | service |
| FR-KBN-021 (position bound) | `KBN_POSITION_EXHAUSTED` | 1264 on `board_cards.position` ([database-schema.md](database-schema.md#project-database-planner_project)) |
| FR-ACC-002 (already a member) | `ACC_ALREADY_MEMBER` | `uq_project_roles` |
| FR-ACC-014 | `ACC_LAST_OWNER` | service |
| FR-ACC-018 | `ACC_NO_PROFILE` | service |
| FR-IDN-006 (decoder limits) | `IMAGE_TOO_LARGE` | `image::Limits` ([architecture.md](architecture.md#user-photo-storage)) |
| FR-KBN-015 (board column page cursor) | `BOARD_CURSOR_STALE` | service |

A rule that is both declarative and checked by the service reports the same code either way. Field rules that are not refusals of a state transition (for example FR-SPR-017, FR-WI-025) are `VALIDATION_FAILED` entries carrying their requirement.

## Resource map

Prefix `/api/v1`; `{p}` = project name. Level per FR-AUTH-018: **R** = `read`, **W** = `read+write`. Role: the extra role requirement, if any ([security.md](security.md#authorisation)). FR references are the functional operations served.

| Resource | Methods | Level | Role | Serves |
|----------|---------|-------|------|--------|
| `/projects` | GET (projects the token can read) | R | — | FR-PRJ-001 |
| `/projects/{p}` | GET | R | — | FR-PRJ-010 |
| `/projects/{p}` | PATCH (`desc`) | W | `owner` | FR-PRJ-010 |
| `/projects/{p}/members` | GET | R | — | FR-ACC-001 |
| `/projects/{p}/members` | POST (`user`, `role`) | W | `owner` | FR-ACC-009, FR-ACC-018 |
| `/projects/{p}/members/{user_id}` | PATCH (`role`), DELETE | W | `owner` | FR-ACC-009, FR-ACC-014 |
| `/projects/{p}/workitems` | GET (`sprint`, `release`, `parent`, `after`, `limit`, `order`) | R | — | FR-VIEW-002, FR-VIEW-006 |
| `/projects/{p}/workitems` | POST | W | — | FR-WI-002…005 |
| `/projects/{p}/workitems/{id}` | GET | R | — | |
| `/projects/{p}/workitems/{id}` | PATCH, DELETE | W | — | FR-WI-022, FR-WI-020 |
| `/projects/{p}/workitems/{id}/close`, `/reopen` | POST | W | — | FR-WI-007, FR-WI-008, closing rules |
| `/projects/{p}/workitems/{id}/tags/{tag_id}` | PUT, DELETE | W | — | FR-TAG-005, FR-TAG-006 |
| `/projects/{p}/workitems/{id}/blocks/{blocked_id}` | PUT, DELETE (`{id}` is the Issue) | W | — | FR-WI-009, FR-WI-017, FR-WI-023 |
| `/projects/{p}/workitems/{id}/comments` | GET | R | — | FR-CMT-002 |
| `/projects/{p}/workitems/{id}/comments` | POST | W | — | FR-CMT-004, FR-CMT-011 |
| `/projects/{p}/comments/{id}` | GET | R | — | |
| `/projects/{p}/comments/{id}` | PATCH, DELETE | W | author while `member`/`owner`, or `owner` | FR-CMT-006, FR-CMT-007, FR-CMT-010 |
| `/projects/{p}/tags` | GET, POST | R / W | — | FR-TAG-001…011 |
| `/projects/{p}/tags/{id}` | GET / PATCH, DELETE | R / W | — | FR-TAG-009, FR-TAG-010 |
| `/projects/{p}/sprints` | GET, POST | R / W | — | FR-SPR-002…006 |
| `/projects/{p}/sprints/{id}` | GET / PATCH, DELETE | R / W | — | FR-SPR-021…023 |
| `/projects/{p}/sprints/{id}/activate`, `/close` | POST | W | — | FR-SPR-007…009 |
| `/projects/{p}/releases` | GET, POST | R / W | — | FR-REL-002 |
| `/projects/{p}/releases/{id}` | GET / PATCH, DELETE | R / W | — | FR-REL-016, FR-REL-019, FR-REL-020 |
| `/projects/{p}/releases/{id}/release` | POST (`releaseddt`) | W | — | FR-REL-004, FR-REL-005, FR-REL-013 |
| `/projects/{p}/boards` | GET | R | — | FR-KBN-001 |
| `/projects/{p}/boards` | POST | W | `owner` | FR-KBN-016 |
| `/projects/{p}/boards/{id}` | GET (`sprint`, `release`): columns, each with its first page of cards | R | — | FR-KBN-004…006, FR-KBN-018 |
| `/projects/{p}/boards/{id}/columns/{column_id}/cards` | GET (`sprint`, `release`, `after`, `limit`) | R | — | FR-KBN-015, [board column pages](#board-column-pages) |
| `/projects/{p}/boards/{id}` | PATCH (`name`, ordered column tags), DELETE | W | `owner` | FR-KBN-003, FR-KBN-016, FR-KBN-022, FR-KBN-023 |
| `/projects/{p}/boards/{id}/moves` | POST (`wiid`, `from`, `to` column ids) | W | — | UC-KBN-001 |
| `/projects/{p}/boards/{id}/columns/{column_id}/order` | PUT (ordered `wiid` list) | W | — | FR-KBN-015 |
| `/projects/{p}/gantt` | GET (`sprint`, `release`) | R | — | FR-VIEW-003…010 |
| `/projects/{p}/audit` | GET (`after`, `limit`, `order`) | R | — | FR-AUD-010, FR-AUD-012 |
| `/projects/{p}/user-search` | GET (`q`) | R | `owner` | FR-ACC-019, FR-ACC-020 ([below](#user-search)) |
| `/users/{id}` | GET: `id` and `display` (FR-IDN-021) | any valid token | — | FR-AUTH-027 |
| `/users/{id}/photo` | GET | any valid token | — | FR-AUTH-027 ([architecture.md](architecture.md#user-photo-storage)) |

## User search

Requirements: FR-ACC-019 (who, active users only, result `id` and name to display), FR-ACC-020 (substring, case and diacritics ignored, trimmed term of ≥ 2 code points, order by name to display with a fixed tie order, first 20 results, indication of more). The identifier returned is the user `id` (`users.id`, FR-IDN-025), the one membership operations take (FR-IDN-026). `idp_subject` is never selected (FR-IDN-027). Response: `{"items": [{"id", "display"}, …], "more": true|false}`.

```sql
SELECT id, COALESCE(displayname, name) AS display
FROM `planner`.`users`
WHERE is_deactivated = 0
  AND (   name        COLLATE utf8mb4_uca1400_ai_ci LIKE ? ESCAPE '!'
       OR displayname COLLATE utf8mb4_uca1400_ai_ci LIKE ? ESCAPE '!'
       OR email       COLLATE utf8mb4_uca1400_ai_ci LIKE ? ESCAPE '!')
ORDER BY COALESCE(displayname, name, '') COLLATE utf8mb4_uca1400_ai_ci, id
LIMIT 21;
```

| Aspect | Rule |
|--------|------|
| Collation | The columns are `utf8mb4_nopad_bin` (D3), which compares exactly. The `COLLATE utf8mb4_uca1400_ai_ci` clause gives the case- and accent-insensitive comparison; that collation exists in 11.8, where it is the server default [S1]. Confirmed by local test [S58]: `'Versão'` matches `%versao%`, `'ANA SILVA'` matches `%ana s%`, `'José Ribeiro'` matches `%jose%` and `%ribe%`, full-width `'Ｊｏｓé'` matches `%jose%`, and `!_` matches only a literal `_`. `LIKE` compares character by character, unlike `=`: `'Straße'` does not match `%strasse%` but matches `%ße%` [S58]. FR-ACC-020 requires insensitivity to letter case and diacritics only; `ß` versus `ss` is neither, so the divergence is compatible. |
| Parameter | `%` + term + `%`, where `!`, `%` and `_` in the term are each prefixed with `!`. `!` is the escape character, so the backslash and `NO_BACKSLASH_ESCAPES` play no part. The term is a bound parameter, never spliced into SQL. (B) |
| Short term | Fewer than 2 code points after trimming: 400 `VALIDATION_FAILED` citing FR-ACC-020. |
| Order and `more` | Ordered by the name to display (`displayname`, else `name`, FR-IDN-021) under the same `ai_ci` collation, ties by ascending user `id`, which never changes (FR-ACC-020, FR-IDN-025). A profile with neither value sorts as the empty string. `LIMIT 21`: the service returns the first 20 rows and sets `more` when a 21st exists. |
| Display | The display name is computed in the query; `name` and `email` are never returned. |
| Cost | A leading `%` prevents index use, so each search scans `users`: ≥ 500 rows at NFR-PERF-001. Acceptable: the search is not a simple operation (NFR-PERF-004), runs only for owners and administrators, and is rate-limited ([security.md](security.md#rate-limiting)). |

## Index coverage

Checked against the DDL of [database-schema.md](database-schema.md) and the NFR-PERF-005 volume (10,000 Work Items, 100,000 comments, 500,000 audit entries per project). An InnoDB secondary index carries the primary key, so `(a)` also orders by `id` within equal `a`. (B)

| Read | Access path | Verdict |
|------|-------------|---------|
| Work Item by id, comment by id, Sprint/Release/tag/board by id (NFR-PERF-004) | Primary key | Covered |
| Work Item list, unfiltered, keyset | Primary key range | Covered |
| Work Item list by Sprint / Release (FR-VIEW-006) | `ix_work_items_sprint (sprint_id, closeddt)`, `ix_work_items_release (release_id, closeddt)`, then sort by `id` | Adequate: one Sprint or Release holds a small subset of the 10,000 rows. Lists have no latency target (NFR-PERF-004). No index added. |
| Children of a Work Item | `ix_work_items_parent` | Covered |
| Comments of a Work Item, by id | `ix_comments_work_item (work_item_id)` + `id` | Covered |
| Tags of a Work Item; Work Items of a tag | PK of `work_item_tags`; `ix_work_item_tags_tag` | Covered |
| Board column cards in order | `ix_board_cards_order (column_id, position)` + `work_item_id` (tie order) | Covered. Sprint/Release filter: PK lookups into `work_items`. |
| Gantt (FR-VIEW-003…010) | Full `work_items` scan (10,000 rows), or the Sprint/Release index when filtered; all `sprints`, `releases` | Adequate: a list operation. No index added. |
| Project audit log, keyset either direction | Primary key | Covered |
| Projects readable by a user / members of a project | `ix_project_roles_user`; `uq_project_roles` | Covered |
| Token by secret; session by id | `uq_tokens_secret_hash`; PK of `sessions` | Covered |
| Sessions of a user (deactivation) | `ix_sessions_user` | Covered (new with `sessions`) |
| Expired sessions (hourly purge) | `ix_sessions_lastseen`, `ix_sessions_created`, one `DELETE` each | Covered (new) |
| User search | Scan of `users` | Adequate ([above](#user-search)) |

No index was added to the project schema.
