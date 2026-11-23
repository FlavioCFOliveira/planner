---
title: Operations
status: draft
last-reviewed: 2026-11-23
related: [architecture.md, security.md, database-schema.md, testing.md]
---

# Operations

## Overview

Build artefacts, deployment, configuration, observability and backup of the server. Source ids `[Sn]` refer to the [source register](database-schema.md#sources). Decider key: [README.md](README.md#conventions).

## Decisions

| # | Decision | Rationale | By |
|---|----------|-----------|----|
| O1 | Two artefacts: an OCI image, and a Linux binary with an example systemd unit. | Container and bare-host deployments. Static assets and templates are inside the binary (A4, A5), so each artefact is self-contained. | U |
| O2 | Logs: structured JSON to stdout via `tracing`, with a request id. No metrics endpoint. | The container runtime or journald collects stdout. tracing-subscriber's `Json` formatter emits newline-delimited JSON (crate features `fmt`, `json`, `std`) [S33]. | U |
| O3 | `/health` and `/ready` endpoints. | Supervisor and proxy checks. | U |
| O4 | Backup: physical instance-wide backup with mariadb-backup plus a copy of the photo directory, run by the operator. | One consistent point in time for `planner` and every `planner_<project>` ([below](#backup-and-restore)). | U |
| O5 | Configuration from environment variables. A secret may instead be read from a file named by the same variable with suffix `_FILE`. | Works unchanged in OCI (env, mounted secrets) and systemd (`EnvironmentFile`). | B |

## Packaging

| Artefact | Content |
|----------|---------|
| OCI image | The server binary; runs as a non-root user; exposes the listen port; the photo directory is a mounted volume. |
| Linux binary | The server binary, plus an example systemd unit that sets `Restart=on-failure`, which restarts on a non-zero exit [S47] (the startup sequence relies on it, [architecture.md](architecture.md#startup-sequence)), reads an `EnvironmentFile` [S47], and runs as a dedicated user owning the photo directory. |
| CLI | The `planner` binary ([cli.md](cli.md)). Distribution channels are not specified. |

## Deployment

1. MariaDB ≥ 11.8 with the [required server settings](database-schema.md#required-server-settings).
2. The operator creates the database `planner` and the three accounts with the account-level grants of [database-schema.md](database-schema.md#privileges). The migration account then creates the tables and grants the runtime account its table privileges at first start (D19).
3. A reverse proxy terminates TLS and forwards to `PLANNER_LISTEN_ADDR`, appending `X-Forwarded-For` and setting `X-Forwarded-Proto` ([security.md](security.md#reverse-proxy-and-client-address)). The proxy does not expose `/health` and `/ready` publicly.
4. An OIDC client registered at the provider with redirect URI `PLANNER_PUBLIC_URL/auth/callback`.
5. Start the server. Migrations and recovery run before it listens.

## Configuration

Every value the server reads. "Secret" values accept the `_FILE` form (O5). Defaults marked B are proposals accepted by the user.

| Variable | Meaning | Default | Source |
|----------|---------|---------|--------|
| `PLANNER_LISTEN_ADDR` | Internal HTTP listen address | `127.0.0.1:8080` (B) | SE6 |
| `PLANNER_PUBLIC_URL` | Public `https` origin; OIDC redirect base and `Origin` check | required | SE1, SE3 |
| `PLANNER_TRUSTED_PROXIES` | CIDR list | `127.0.0.1/32,::1/128` (B) | SE6 |
| `PLANNER_DB_HOST`, `PLANNER_DB_PORT` | MariaDB address | required; `3306` | NFR-ARCH-004 |
| `PLANNER_DB_RUNTIME_USER`, `PLANNER_DB_RUNTIME_PASSWORD` (secret) | Runtime account | required | D17 |
| `PLANNER_DB_PROVISIONING_USER`, `PLANNER_DB_PROVISIONING_PASSWORD` (secret) | Provisioning account | required | D17 |
| `PLANNER_DB_MIGRATION_USER`, `PLANNER_DB_MIGRATION_PASSWORD` (secret) | Migration account | required | D17 |
| `PLANNER_DB_POOL_MAX` | Pool maximum | `32` (B) | DA5 |
| `PLANNER_DB_SSL_MODE` | `disabled`, `preferred`, `required`, `verify_ca` or `verify_identity` | `disabled` (U) | DA9 |
| `PLANNER_DB_SSL_CA` | CA certificate file for `verify_ca` / `verify_identity` | none (B) | DA9 |
| `PLANNER_OIDC_ISSUER` | Issuer URL | required | SE1, FR-IDN-024 |
| `PLANNER_OIDC_CLIENT_ID` | Client id | required | SE1 |
| `PLANNER_OIDC_CLIENT_SECRET` (secret) | Client secret | required | SE1 |
| `PLANNER_OIDC_AUTH_ENDPOINT`, `PLANNER_OIDC_TOKEN_ENDPOINT`, `PLANNER_OIDC_JWKS_URI`, `PLANNER_OIDC_USERINFO_ENDPOINT` | Explicit endpoints for a provider without discovery. Setting the first switches discovery off; the first three are then required, UserInfo optional. | unset (U; names: B) | SE1 |
| `PLANNER_COOKIE_KEY` (secret), `PLANNER_COOKIE_KEY_PREVIOUS` (secret) | Key of the pre-login cookie, and the previous key accepted during rotation: standard base64 of ≥ 64 bytes. | required; unset | [security.md](security.md#pre-login-cookie) |
| `PLANNER_FIRST_ADMIN_SUB` | IdP `sub` of the first administrator | required | FR-ACC-012 |
| `PLANNER_PHOTO_DIR` | Photo directory | required | [architecture.md](architecture.md#user-photo-storage) |
| `PLANNER_PHOTO_DELETE_DELAY` | Minimum age of an unreferenced photo file before the sweep deletes it. MUST exceed the interval between backups. | `48h` (U) | [architecture.md](architecture.md#periodic-tasks) |
| `PLANNER_SESSION_IDLE_TIMEOUT` | Session idle timeout | `8h` | SE2 (U) |
| `PLANNER_SESSION_ABSOLUTE_TIMEOUT` | Session absolute timeout | `7d` | SE2 (U) |
| `PLANNER_RATE_IP_RPS`, `PLANNER_RATE_IP_BURST` | General per-IP bucket | `200`, `400` (B) | SE5 |
| `PLANNER_RATE_TOKEN_RPS`, `PLANNER_RATE_TOKEN_BURST` | Per-token bucket | `20`, `40` (B) | SE5 |
| `PLANNER_RATE_LOGIN_PER_MIN`, `PLANNER_RATE_LOGIN_BURST` | Login bucket, per IP | `10`, `10` (B) | SE5 |
| `PLANNER_RATE_AUTHFAIL_PER_MIN`, `PLANNER_RATE_AUTHFAIL_BURST` | Failed-token bucket, per IP | `10`, `10` (B) | SE5 |
| `PLANNER_LOG` | `tracing` level filter | `info` (B) | O2 |

Fixed values, not configuration: pool acquire timeout 2 s (DA7); session `innodb_lock_wait_timeout` 5 s (DA8); periodic task interval 1 h.

## Observability

| Aspect | Rule | By |
|--------|------|----|
| Format | One JSON object per line on stdout [S33]. | U |
| Request id | Taken from `X-Request-Id` when sent by a trusted proxy, otherwise generated. Returned in the `X-Request-Id` response header and in problem details (`request_id`); attached to every log line of the request. | U (header handling: B) |
| Request log | One line per request: request id, method, route template, status, latency in ms from receipt to response, client IP, user id when known, project when applicable. The latency field is the NFR-PERF-003 measurement point (measured at the server, excluding network transit). | B |
| Never logged | Token secrets, `Authorization` headers, session ids, `Cookie` and `Set-Cookie` headers, CSRF tokens, OIDC codes, `state`, `nonce` and PKCE verifiers, the OIDC client secret, database passwords. Request and response bodies are not logged. | U (list beyond the decision: B) |
| `/health` | 200 while the process runs. No authentication. | U |
| `/ready` | 200 after the startup sequence completed and while a pool connection answers `SELECT 1`; 503 otherwise. No authentication. | U (condition: B) |
| Metrics | None. | U |

## Backup and restore

Executed by the operator (O4).

Backup:

1. `mariadb-backup --backup --target-dir=<dir>` on the running server, then `mariadb-backup --prepare --target-dir=<dir>` [S27]. InnoDB hot backups are supported. After prepare, InnoDB recovery makes the copy consistent at one point in time, and the default global read lock makes all tables consistent at that point regardless of engine; `--no-lock` MUST NOT be used, because without the lock only InnoDB tables are consistent [S27]. The backup user needs at least `RELOAD, PROCESS, LOCK TABLES, BINLOG MONITOR` [S27].
2. Copy `PLANNER_PHOTO_DIR`, after step 1 and within `PLANNER_PHOTO_DELETE_DELAY` of it.

The backup covers every database and the `mysql` system tables, so accounts and grants are restored with the data.

Photo coherence: a replaced or removed photo file is deleted no earlier than `PLANNER_PHOTO_DELETE_DELAY` after the change ([architecture.md](architecture.md#user-photo-storage)). When that delay exceeds the interval between backups, step 2 still finds every file referenced by the step-1 snapshot. Files created after the snapshot are copied too and are unreferenced after restore; the sweep removes them once older than the delay.

Restore order:

1. Stop planner, then stop MariaDB.
2. Empty the MariaDB `datadir`; `mariadb-backup --copy-back --target-dir=<dir>`; restore ownership of the `datadir` to the MariaDB user; start MariaDB [S27].
3. Restore the photo directory.
4. Start planner. Its startup sequence migrates and recovers as after any restart.
