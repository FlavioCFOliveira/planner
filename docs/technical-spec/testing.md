---
title: Testing
status: draft
last-reviewed: 2026-11-23
related: [architecture.md, data-integrity-and-concurrency.md, database-schema.md, security.md, web-api.md, cli.md, ../functional-spec/overview.md]
---

# Testing

## Overview

Four levels, each against what it can prove. Acceptance criteria of the functional specification are the primary test cases; tests cite the requirement IDs they cover. Source ids `[Sn]` refer to the [source register](database-schema.md#sources). Decider key: [README.md](README.md#conventions).

## Decisions

| # | Decision | Rationale | By |
|---|----------|-----------|----|
| T1 | Unit tests for the domain. | Type-level invariants (**T** in [domain-model.md](domain-model.md)) need no database. | U |
| T2 | Integration tests against a real MariaDB 11.8 in a container, including concurrency, locking and retry tests. | Declarative rules (**D**), locking (L1–L6) and privileges exist only in the server; a substitute database would not prove them. Project SQL is not compile-time checked (DA3), so these tests are its check. | U |
| T3 | End-to-end CLI tests against a test server. | Proves the CLI contract: commands, JSON output, exit codes ([cli.md](cli.md)). | U |
| T4 | A performance test verifies NFR-PERF-001…005. | The targets are measured, not assumed; DA5, DA6 and the table-cache settings depend on it. | B (from NFR-PERF-003, NFR-PERF-005) |
| T5 | Integration tests start MariaDB with `testcontainers` 0.28.0, `GenericImage::new("mariadb", "11.8")` and `AsyncRunner::start` [S56]; a Docker-compatible engine is required on the test host. | `testcontainers-modules` 0.15.0 depends on `testcontainers` ^0.27 [S56], so the generic image of the current release is used. | U (generic image: B) |
| T6 | Load generator: k6 2.4.0 [S57]. A threshold `p(95)<200` on the simple-operation requests is the pass criterion: a failed threshold ends k6 with a non-zero exit code [S57]. | Automated pass or fail in CI. | U |


## Unit tests

Every value object constructor (valid, boundary, invalid: for example `Title` of 200 and 201 code points, `TagName` with upper case), `WorkItemKind` rule functions against BR-WI-014 and FR-SPR-011/FR-REL-007, `Sprint` transitions (FR-SPR-007), token secret format, `Color` parsing and formatting (FR-TAG-015), and the effective-permission table ([security.md](security.md#authorisation)).

## Integration tests

Each test runs against a fresh database set on MariaDB 11.8 with the [required settings](database-schema.md#required-server-settings) and the three accounts.

| Area | Proves |
|------|--------|
| Schema | Both DDL scripts load; each CHECK, UNIQUE and FK rejects its counter-example (the cases of [S22] become regression tests). |
| Privileges | The runtime account cannot `UPDATE` or `DELETE` `audit_log`, nor run DDL (FR-AUD-006). Provisioning creates, grants, revokes and drops a project database. |
| Rules | Every row of the [enforcement table](data-integrity-and-concurrency.md#rule-enforcement) and every problem `code` of [web-api.md](web-api.md#business-rule-codes). |
| Concurrency | Two connections race on: last owner removal (FR-ACC-014); last administrator (FR-ACC-016); second active Sprint (FR-SPR-008); reverse blocking links (FR-WI-023); close vs. child create (BR-CLS-001, FR-CLS-002); card append in one column (FR-KBN-021); role removal vs. write ([permission check](data-integrity-and-concurrency.md#permission-check-in-writes)); deactivation vs. write with the user's token. Each race ends with exactly the outcome one serial order allows. |
| Retry | A forced deadlock (1213) and a lock wait timeout (1205) are retried and succeed; four consecutive conflicts end in 503 `RETRY_EXHAUSTED` (L4). Pool exhaustion ends in 503 `POOL_EXHAUSTED` after 2 s. |
| Migrations | A run interrupted after each step resumes and ends at the same schema as an uninterrupted run (D16). A database at a newer version stops startup. |
| Recovery | Projects left in `provisioning` and `deleting` at each step are rolled back or completed at startup. |
| Instance guard | A second server process against the same database instance refuses to start (A1). |
| Sessions | Idle and absolute timeouts; logout and deactivation delete sessions; CSRF token and `Origin` refusals. |
| OIDC | Login against Keycloak 26.8.2 [S46], with discovery and with explicit endpoints: first-login profile creation, truncation of long IdP values (FR-IDN-023), deactivated user refused, first administrator at first login and at a later login of an existing profile (FR-ACC-012). |
| Photos | Format guessing accepts PNG, JPEG and WebP only; size limit; decoder limits (an image over 8,192 pixels in either dimension refused with `IMAGE_TOO_LARGE`); first frame of animated input; centre crop; WebP output; replacement order (a crash simulated between steps leaves only unreferenced files); the sweep deletes only unreferenced files older than the delay. |
| Periodic tasks | Session purge by each timeout; photo sweep at startup and hourly. |
| Search | FR-ACC-019, FR-ACC-020: case and accent insensitivity (the [S58] cases, including `Straße` not found by `strasse`), order by name to display with ties by id, `more` with 21 matches, escaping of `%`, `_` and `!`, minimum length, 20-result cap, deactivated users absent, `member` refused. |
| Pre-login cookie | Tampered, expired and reused cookies refused; a cookie sealed with the previous key accepted during rotation. |
| Board pages | Cursor pages follow card order; `BOARD_CURSOR_STALE` after the cursor card leaves the column. |

## End-to-end CLI tests

The CLI binary runs against a test server whose tokens are inserted directly in the test database (token issue is WEBSITE-only, FR-AUTH-017). Cases: every command of the [command map](cli.md#command-map) once; `planner schema` matches the parser; each exit code class; JSON on stdout and stderr only; device mismatch refused (FR-AUTH-010); `PLANNER_TOKEN` override; exit code 8 for an unreadable configuration and for a configuration file readable by others (mode bits on Linux and macOS, DACL on Windows). The test server listens on loopback and the CLI connects over `http://127.0.0.1`, which both sides accept for loopback ([cli.md](cli.md#configuration-and-credentials), [security.md](security.md#cli-tokens)); a non-loopback `http` URL is refused. The suite runs on Linux, macOS and Windows (FR-CLI-007).

## Performance test

Verifies NFR-PERF-003 under the conditions of NFR-PERF-001, NFR-PERF-002 and NFR-PERF-005.

| Parameter | Value | Source |
|-----------|-------|--------|
| Data | 500 projects, each with 10,000 Work Items, 100,000 comments and 500,000 project audit entries; 500 user profiles | NFR-PERF-001, NFR-PERF-005 |
| Concurrency | ≥ 100 distinct users active at the same time | NFR-PERF-001 |
| Load | ≥ 200 requests per second sustained, all clients | NFR-PERF-002 |
| Mix | Simple operations (NFR-PERF-004, including Sprint activation and close and Release release) across projects, plus list operations, which count towards the load but have no latency target | NFR-PERF-002, NFR-PERF-004 |
| Measure | p95 of the request-log latency of simple operations < 200 ms, measured at the server ([operations.md](operations.md#observability)) | NFR-PERF-003 |
| Pass criterion | k6 threshold `http_req_duration{kind:simple}: ['p(95)<200']` on requests tagged `kind: simple` [S57]. k6 observes latency at the client, which includes what the server measures, so passing it implies the server-side target; the request log confirms the server-side value. | T6 |
| Also recorded | Pool saturation, statement-cache misses, table-cache misses, L4 retries | DA5, DA6, [required settings](database-schema.md#required-server-settings) |

The topology is the production one: one server instance (A1) behind a reverse proxy.
