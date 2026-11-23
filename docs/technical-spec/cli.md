---
title: CLI
status: draft
last-reviewed: 2026-11-23
related: [web-api.md, security.md, architecture.md, ../functional-spec/cli.md, ../functional-spec/cli-authentication.md]
---

# CLI

## Overview

The `planner` binary is a client of [`/api/v1`](web-api.md) whose primary user is an AI agent (NFR-USE-001, NFR-USE-002). It runs on Linux, macOS and Windows (FR-CLI-007). Every design choice below favours predictable, machine-readable behaviour over interactive convenience. Source ids `[Sn]` refer to the [source register](database-schema.md#sources). Decider key: [README.md](README.md#conventions).

## Decisions

| # | Decision | Rationale | By |
|---|----------|-----------|----|
| C1 | Argument parsing with clap 4.6.9 [S28, S32]. | Nested subcommands, generated `--help`, and runtime introspection of the command tree (`get_subcommands`, `get_arguments`, …) [S32], which produces `planner schema` from the same definition as the parser. | B |
| C2 | Commands are noun–verb: `planner <noun> <verb> [args]`. | One fixed shape lets an agent derive commands from the resource map ([web-api.md](web-api.md#resource-map)). | U |
| C3 | Help has two forms: concise text `--help`, and `planner schema`, a JSON description of every command, argument, type and error code. | Agents discover the whole surface in one call (NFR-USE-001). | U (command name: B) |
| C4 | Output is JSON (FR-CLI-002). Errors are JSON too and carry the problem `code`. | Agents parse one format. | U |
| C5 | Exit codes 0, 2–8 ([below](#exit-codes)). | Agents branch on the class of failure without parsing. | U |
| C6 | Credentials in a config file in the platform's per-user configuration directory, readable by the user only; `PLANNER_TOKEN` and `PLANNER_SERVER_URL` override it. | [Below](#configuration-and-credentials). FR-CLI-007. | U |
| C7 | No prompts, no colour, no pager, no progress output. | An agent cannot answer a prompt; decoration corrupts JSON. | B |
| C8 | HTTP client: `reqwest` 0.13.5 with default features disabled and the `rustls` and `json` features [S28, S53]. | One pure-Rust TLS stack on all three platforms (A8). | U (features: B) |
| C9 | Configuration directory from `directories` 6.0.0, `ProjectDirs::from("", "", "planner")` [S54]. | It maps one call to the platform conventions below. | B |
| C10 | Windows ACL through `windows-sys` 0.61.2, Microsoft's raw Windows API bindings [S28, S59]. | `windows-acl` 0.3.0 was last released on 2021-01-11 [S28]; the official bindings are maintained and expose the needed calls [S59]. | B |

## Command map

Each project-scoped command takes `--project <name>`. The set equals FR-CLI-005 minus FR-CLI-006. (Command and option names: B, under C2.)

| Noun | Verbs | API resource |
|------|-------|--------------|
| `project` | `list`, `show`, `update` (`--desc`) | `/projects`, `/projects/{p}` |
| `member` | `list`, `search` (`--q`; prints `{"items", "more"}`), `add`, `set-role`, `remove` | `/projects/{p}/members`, `/projects/{p}/user-search` (FR-ACC-019, FR-ACC-020) |
| `wi` | `list` (`--sprint`, `--release`, `--parent`), `show`, `create`, `update`, `delete`, `close`, `reopen`, `tag`, `untag`, `block`, `unblock` | `/projects/{p}/workitems…` |
| `comment` | `list` (`--wi`), `show`, `add`, `edit`, `delete` | `…/comments` |
| `tag` | `list`, `show`, `create`, `update`, `delete` | `/projects/{p}/tags` |
| `sprint` | `list`, `show`, `create`, `update`, `delete`, `activate`, `close` | `/projects/{p}/sprints` |
| `release` | `list`, `show`, `create`, `update`, `delete`, `release` | `/projects/{p}/releases` |
| `board` | `list`, `show` (`--sprint`, `--release`), `cards` (`--column`, `--after`, `--limit`), `create`, `update`, `delete`, `move`, `reorder` | `/projects/{p}/boards…` ([board column pages](web-api.md#board-column-pages)) |
| `gantt` | `show` (`--sprint`, `--release`) | `/projects/{p}/gantt` |
| `audit` | `list` | `/projects/{p}/audit` |
| `user` | `show` (`id` and `display`; never the IdP subject, FR-IDN-027), `photo` (`--out <path>`: writes the WebP file and prints `{"path": …}`) | `/users/{id}`, `/users/{id}/photo` (FR-AUTH-027) (B) |
| `schema` | (none) | local |
| `device-id` | `show` | local: prints the `DEVICE-ID` to supply when issuing a token on the WEBSITE (FR-AUTH-009) |
| `config` | `set-server`, `set-token` | local |

Absent by FR-CLI-006: token issue/list/revoke, project create/delete, user (de)activation, administrator status, profile edits, central audit log. Membership search is present (FR-CLI-006 as amended, FR-ACC-019).

Users are designated by the user `id` in arguments and output (FR-IDN-026).

List commands accept `--after`, `--limit` and `--order` and return the API's `{"items", "next"}` unchanged ([web-api.md](web-api.md#pagination)).

## Help and schema

- `--help` on any command: text, one line of purpose, then arguments with types and defaults. No examples longer than one line.
- `planner schema` prints one JSON document: for each command, its path (`["wi","create"]`), purpose, arguments (name, type, required, default, allowed values, repeatable), the output shape, and the problem `code` values it can return ([web-api.md](web-api.md#business-rule-codes)). It needs no server and no credentials. It is generated from the clap definition [S32] plus a static table of codes, so it cannot drift from the parser. (Generation method: B.)

## Output

| Stream | Content | By |
|--------|---------|----|
| stdout | On success, exactly one JSON value: the API response body, unchanged. | U (stream: B) |
| stderr | On failure, exactly one JSON object: the problem details from the server, unchanged, or for local failures an object of the same shape with a CLI `code` (`USAGE`, `NETWORK`, `CONFIG_UNREADABLE`, `CONFIG_PERMISSIONS`, `IO_ERROR`, `INSECURE_URL`). | U (stream: B; local codes: B) |

Date-times are printed as the API returns them, in UTC with `Z` (FR-PLT-007).

## Exit codes

| Code | Meaning | From |
|------|---------|------|
| 0 | Success | 2xx |
| 2 | Usage: invalid arguments, or the server refused the input | parse error; 400, 422 |
| 3 | Authentication | 401; no token configured |
| 4 | Permission | 403 |
| 5 | Not found | 404 |
| 6 | Business rule or conflict | 409 |
| 7 | Network or server | connection failure, timeout, 429, 5xx |
| 8 | Local failure: configuration unreadable or invalid, file permissions too open or not settable, local I/O error | CLI |

Mapping of 400/422 to 2 and of 429 to 7: B. Exit code 8: U. A non-`https`, non-loopback server URL is a usage error (2, `INSECURE_URL`) (B).

## Configuration and credentials

| Item | Rule | By |
|------|------|----|
| File | `config.toml` in `ProjectDirs::config_dir()` [S54]: Linux `$XDG_CONFIG_HOME/planner` or `$HOME/.config/planner` [S45]; macOS `$HOME/Library/Application Support/<project path>`; Windows `{FOLDERID_RoamingAppData}\<project path>\config` (`%APPDATA%`). If no home directory can be determined (`from` returns `None` [S54]), the CLI exits with 8. | Per-platform location: U. Format: B. Crate: B |
| Protection | Linux and macOS: directory mode 0700, file mode 0600; the CLI refuses to read a token from a file readable by group or others. Windows: the CLI sets on the directory and the file a protected DACL (inheritance off) whose only entry grants the current user access, using `windows-sys` 0.61.2 (`SetEntriesInAclW`, `SetNamedSecurityInfoW` with `SE_FILE_OBJECT`, `GetNamedSecurityInfoW`) [S59]; it refuses to read a token from a file whose DACL grants access to anyone else. Refusal on any platform: exit 8, `CONFIG_PERMISSIONS`. | U (Unix modes, Windows ACL rule); crate: B |
| Contents | `server_url`, `device_id`, `token`. | U |
| `DEVICE-ID` | 24 characters from `[A-Za-z0-9]`, drawn with the operating system's CSPRNG (FR-AUTH-013), generated when the CLI runs and finds none, and written to the file (FR-AUTH-012); an existing value is reused (FR-AUTH-021). Never `GUI` (FR-AUTH-016; the pattern excludes it). | U |
| Overrides | `PLANNER_TOKEN` replaces `token`; `PLANNER_SERVER_URL` replaces `server_url`. The `DEVICE-ID` has no override. | U |
| Transport | Every request sends `Authorization: Bearer` and `X-Device-Id` ([security.md](security.md#cli-tokens)). The server URL MUST be `https`, except that `http` is accepted when the host is `127.0.0.1`, `::1` or `localhost`. | U |
| Secrets | The token never appears in output, errors or `planner schema`. | B |
