---
title: Data integrity and concurrency
status: draft
last-reviewed: 2026-11-23
related: [database-schema.md, domain-model.md, open-questions.md, ../functional-spec/closing-rules.md, ../functional-spec/kanban-boards.md, ../functional-spec/audit-log.md]
---

# Data integrity and concurrency

## Overview

Row-local rules are declarative ([database-schema.md](database-schema.md)). A rule that spans rows is enforced by the application, in the same transaction as the write, under explicit row locks. Two concurrent requests therefore cannot both pass a check that only one of them may pass. Source ids `[Sn]` refer to the [source register](database-schema.md#sources).

## Locking protocol

| # | Rule |
|---|------|
| L1 | Every check that decides whether a write is allowed reads with a locking read: `SELECT … FOR UPDATE` (X) or `SELECT … LOCK IN SHARE MODE` (S) [S15]. The check and the write run in one transaction. A plain consistent read never decides a write. |
| L2 | A transaction that changes the state another rule depends on takes X on that row. A transaction whose validity depends on a row's state takes at least S on it. |
| L3 | Lock order, to reduce deadlocks: `projects` row → Sprint/Release rows → Work Item rows by ascending `id` → `board_columns` rows → `project_settings` row. |
| L4 | On deadlock (error 1213) or snapshot conflict (error 1020, `ER_CHECKREAD`, raised when `innodb_snapshot_isolation=ON` [S14]), the application rolls back and retries the whole transaction. The number of retries is bounded, and the bound is not yet specified ([TOQ-007](open-questions.md#toq-007)). |
| L5 | The audit entries of an operation (FR-AUD-002, FR-AUD-014) are inserted in the operation's transaction, all with one `createddt`. |

## Rule enforcement

"Decl." = declarative constraint. "App" = application check under L1.

| Rule | Enforced by | Locks taken (L1–L3) |
|------|-------------|---------------------|
| BR-CLS-001 / FR-CLS-001: no close with open children | App | X on the WI. S on its children via `ix_work_items_parent` (`parent_id = ? AND closeddt IS NULL`). |
| BR-CLS-002: no close while blocked by an open Issue | App | X on the WI. S on blocking Issues (join `blockings` → `work_items`). |
| FR-CLS-002: no create or reopen under a closed parent | App | S on the parent. X on the WI when reopening. |
| FR-CLS-003: no create or reopen of an Issue blocking a closed WI | App | X on the Issue. S on the blocked WIs. |
| FR-CLS-004: no link from an open Issue to a closed WI | App | S on the Issue and on the blocked WI. |
| FR-CLS-005: no parent set to a closed WI | App | X on the child. S on the new parent. |
| FR-CLS-006 / FR-WI-024: no reopen or delete in a closed Sprint or released Release | App | S on the WI's Sprint and Release. X on the WI. |
| FR-WI-023: no blocking cycle | App: `WITH RECURSIVE` over `blockings` from the blocked id, looking for the blocker | X on `project_settings` (id 1), the per-project serialisation point for link insertion. Two concurrent inserts (A→B, B→A) would otherwise both pass. |
| BR-WI-014/015/016, FR-WI-016 | Decl. (composite FK + CHECK) | none beyond the implicit FK check |
| FR-SPR-011, FR-REL-007 | Decl. (CHECK) | none |
| FR-SPR-008 / BR-SPR-001 | Decl. (`uq_sprints_one_active`) | none. A duplicate-key error maps to refusal. |
| FR-SPR-007: state transitions | App | X on the Sprint. |
| FR-SPR-009 / BR-SPR-002: close only when all assigned WIs are closed | App | X on the Sprint. S on assigned open WIs (`ix_work_items_sprint`). |
| FR-SPR-020 / FR-REL-015: frozen assignments | App | S on the source and target Sprint/Release. X on the WI. |
| FR-SPR-022 / FR-REL-019: frozen fields | App | X on the Sprint/Release. |
| FR-SPR-021 / FR-REL-016: no delete with assignments | Decl. (FK RESTRICT) | none |
| FR-SPR-023 / FR-REL-020: no delete when closed/released | App | X on the Sprint/Release. |
| FR-REL-005 / BR-REL-001: release only when all assigned WIs are closed | App | X on the Release. S on assigned open WIs. |
| FR-REL-014 / FR-REL-018: no un-release; `releaseddt` ≤ `UTC_DATE` | App | X on the Release. |
| FR-TAG-006/007, FR-KBN-020, FR-KBN-002 (at most one each) | Decl. (PK/UNIQUE) | none |
| FR-ACC-014: last owner | App | X on the `projects` row (serialises all membership changes of the project). |
| FR-IDN-018: no deactivation of a last owner | App | X on every `projects` row where the user is `owner`, ascending `id`. X on the user. |
| FR-ACC-016 / FR-IDN-020: last active administrator | App | X on all `users` rows with `is_admin = 1`. |
| FR-KBN-021 / FR-KBN-015: append at bottom, reorder | App | X on the `board_columns` row before computing `MAX(position) + 1` or renumbering. |
| FR-AUD-006 / BR-AUD-001 | Decl. (privileges) | none |

Insertions under a parent, Sprint or Release take S on that row. A close or state change takes X on it. Conflicting operations therefore serialise. Where both sides hold one lock and wait on the other, L4 resolves the deadlock by retrying.

## Work Item deletion

Requirements: FR-WI-020, FR-WI-021, FR-WI-024, FR-AUD-014. One transaction:

1. X on the WI. S on its Sprint and Release. Refuse if either is frozen (FR-WI-024).
2. Read, with locks, for the audit: the children ids, the comment ids, and the WIs at the other end of each blocking link.
3. `UPDATE work_items SET parent_id = NULL, parent_kind = NULL WHERE parent_id = ?`. This step is required: the parent FK is `ON DELETE RESTRICT` ([database-schema.md](database-schema.md#project-database-planner_project)).
4. `DELETE` the WI. FK cascades remove its comments, blocking links, tag assignments and board cards.
5. Insert the audit entries: `delete` for the WI and each comment; `modify` for each child and each blocking counterpart.

## Tag deletion

Requirements: FR-TAG-010, FR-KBN-017, FR-AUD-014. One transaction:

1. Read the WIs carrying the tag, and the `board_columns` it defines (with their boards).
2. For each such board: compute the open WIs that, without that column, carry none of the board's remaining column tags. Each must enter the `open` column (FR-KBN-004).
3. `DELETE` the tag. Cascades remove its assignments, its columns and those columns' cards.
4. Append the step-2 WIs to `open` ([below](#board-card-maintenance)).
5. Audit: `delete` `tag`; `modify` `workitem` for each WI that lost it; `modify` `board` for each board that lost a column.

## Board card maintenance

Invariant: the `board_cards` rows of every column equal the contents defined by FR-KBN-004…006.

- Cards are maintained eagerly, in the transaction of every change that moves a WI into or out of a column: WI create or delete, close or reopen, tag applied or removed, column added or removed, card move (UC-KBN-001).
- A WI entering a column is inserted at `MAX(position) + 1` (FR-KBN-021). A WI leaving a column has its row deleted. The positions of other cards are unchanged.
- When a column is added, the matching WIs already present are appended in ascending `id`. When a column is removed, WIs that newly qualify for `open` are appended there in ascending `id`. FR-KBN-021 does not order simultaneous entries, so this order is a technical choice.
- Creating a board inserts its `open` and `closed` columns, and their cards in ascending `id`, in the same transaction.
- A manual reorder (FR-KBN-015) rewrites positions within one column. It is audited as `modify` `board`. Automatic placement records no `board` entry (FR-KBN-015, FR-KBN-021).
- Cost: every board holds one row per WI in its `closed` or `open` column, so storage is about boards × WIs.

## Project provisioning and deletion

`CREATE`/`DROP DATABASE`, `CREATE TABLE` and `GRANT`/`REVOKE` commit implicitly [S13]. `DROP DATABASE` is crash-safe but not atomic as a whole [S13]. `projects.state` makes the sequences recoverable. Only `active` projects are visible to any API operation or token check.

Creation (FR-PRJ-008, FR-ACC-015):

1. Central transaction: `INSERT projects (name, state='provisioning')`. The UNIQUE name reserves the name (FR-PRJ-006).
2. `CREATE DATABASE planner_<name>`, the tables, the `schema_version` row, the `project_settings` row (id 1), and the application's table-level grants ([Privileges](database-schema.md#privileges)).
3. Central transaction: set `state='active'`, insert the owner `project_roles`, and insert the audit entries (`create` `project`, `create` `role` per owner).

Deletion (FR-PRJ-009, FR-AUD-013):

1. Central transaction: X on the `projects` row, `state='deleting'`, audit `delete` `project`, `delete` `role` per role, and `modify` `token` per token with a grant for the project (FR-AUD-014). The audit entries are written here because the decision is irreversible from this point, and recovery does not know the actor.
2. `REVOKE` the application's grants on the project database. `DROP DATABASE IF EXISTS planner_<name>`.
3. Central transaction: `DELETE` the `projects` row. Cascades remove roles and token grants (FR-AUTH-020).

Recovery: a project left in `provisioning` is rolled back (drop its database if it exists, delete the row, no audit entry). A project left in `deleting` is completed from step 2. Every step is idempotent. How recovery tells a crashed sequence from one still running on another server instance depends on the deployment topology: [TOQ-007](open-questions.md#toq-007).

## Other composite operations

| Operation | Transaction content |
|-----------|---------------------|
| User deactivation (FR-IDN-010, FR-IDN-012) | Locks per [the enforcement table](#rule-enforcement). Set `is_deactivated`, `revoked = 1` on all the user's tokens, delete all the user's `project_roles`. Audit: `modify` `user`; `modify` `token` per token revoked; `delete` `role` per role removed (FR-AUD-014; mapping in [database-schema.md](database-schema.md#central-database-planner)). |
| Card move (UC-KBN-001) | Tag changes (FR-KBN-008…012, FR-KBN-019), close or reopen with their checks (FR-KBN-013, FR-KBN-014), card maintenance. One `modify` `workitem` entry. |
| Role change | Update `project_roles.role` in place, so `entityid` stays stable. Audit `modify` `role`. |
