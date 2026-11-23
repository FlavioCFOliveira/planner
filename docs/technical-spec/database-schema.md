---
title: Database schema
status: draft
last-reviewed: 2026-11-23
related: [domain-model.md, data-integrity-and-concurrency.md, architecture.md, security.md, open-questions.md, ../functional-spec/multi-project.md, ../functional-spec/overview.md]
---

# Database schema

## Overview

Persistence is MariaDB (NFR-ARCH-004). It uses one central database, `planner` (FR-PRJ-003), and one database per project, `planner_<project>` (FR-PRJ-002, FR-PRJ-004). Access is through sqlx with hand-written SQL ([architecture.md](architecture.md#database-access)).

## Decisions

Decider key: [README.md](README.md#conventions). **[S22]** = local test on MariaDB 11.8.9, 2026-11-23.

| # | Decision | Rationale | By |
|---|----------|-----------|----|
| D1 | The minimum server version is **MariaDB 11.8 LTS**. | 11.8 is LTS: first GA 11.8.2 on 2025-06-04, maintained until June 2028 [S1]. Latest at review: 11.8.10, 2026-11-16 [S2]. | U |
| D2 | Ids are native `AUTO_INCREMENT`, unsigned, sized to the smallest type that covers the expected number of creations: `SMALLINT UNSIGNED` (2 bytes, max 65,535) for projects, Sprints, Releases, boards, board columns and tags; `MEDIUMINT UNSIGNED` (3 bytes, max 16,777,215) for users, project roles, tokens, Work Items and the central audit log; `INT UNSIGNED` (4 bytes, max 4,294,967,295) for comments and the project audit log. Gaps are acceptable and ids are never reused. | InnoDB keeps the counter across restarts. Deleted values are never reused. Rolled-back and failed inserts leave gaps [S7]. The counter survives a restart [S22]. Matches FR-WI-003, FR-CMT-004, FR-SPR-015 and FR-REL-010 (increasing, never reused, gaps allowed). Policy: use the least, grow only as far as needed. The ceiling bounds creations, not live rows, because ids are not reused. Examples: 65,535 Sprints is more than a weekly Sprint for 1,000 years; 16.7 million Work Items per project exceeds the largest known trackers' projects; comments and project audit entries grow fastest (several per Work Item and per change), so they get `INT`. Every FK column has exactly the type of the id it references. `audit_log.entityid` has the type of the widest id its entities use (`INT` in projects, `MEDIUMINT` centrally); the central log therefore has the same columns as the project log (FR-AUD-007) with narrower integer widths. Exhausting a counter makes inserts fail with error 167, SQLSTATE 22003, as received by the client [S48], and needs a widening migration. | U (widths: B) |
| D3 | Engine InnoDB. Each database's default is `CHARACTER SET utf8mb4 COLLATE utf8mb4_nopad_bin`. | FKs are supported only by InnoDB [S5]. An explicit default stops the 11.8 server default, `utf8mb4_uca1400_ai_ci` [S1], which ignores case and accents, from applying silently. `nopad_bin` compares exactly, including trailing spaces [S6, S10]. | B |
| D4 | Machine identifiers (project name, tag name and value, `deviceid`, IdP subject) use `CHARACTER SET ascii COLLATE ascii_nopad_bin`. | Their alphabets are ASCII (FR-PRJ-005, FR-TAG-012, FR-AUTH-023, [S21]). NO PAD keeps `'x'` and `'x '` distinct [S6], which matters for IdP subjects (identity confusion). | B (NO PAD instead of `ascii_bin`: see [deviations](#deviations-from-the-brief)) |
| D5 | Sprint, Release and Board `name` use `utf8mb4_uca1400_nopad_ai_ci`. | Uniqueness ignores letter case and diacritics (FR-SPR-016, FR-REL-012, FR-KBN-022) (U). UCA-14.0.0 collations exist since 10.10.1 [S6]. `'Versão' = 'versao'` [S22]. The equality goes beyond case and diacritics: `'ß' = 'ss'` and full-width `'Ａ' = 'a'` also hold [S22]. This is acceptable under the decision. | U / B |
| D6 | Date-only fields are `DATE`. Date-times are `DATETIME` (whole seconds, 5 bytes) holding UTC, produced by the application at second precision. No requirement needs sub-second precision: ordering uses the ids. `DATETIME(6)` would cost 8 bytes per value for nothing. Every connection sets `time_zone = '+00:00'`. | NFR-DATA-001 (INVIOLABLE), FR-PLT-005. `DATETIME` covers years 1000–9999 with 0–6 fractional digits [S9]. It is not shifted by `time_zone`, while `TIMESTAMP` is: a value inserted at `+00:00` and read at `+05:00` returns 12:00 from `DATETIME(6)` and 17:00 from `TIMESTAMP(6)` [S9, S22]. A fraction supplied to `DATETIME` is truncated (`10:11:12.987` → `10:11:12`) [S22]. `TIMESTAMP` is rejected: it converts between the session time zone and UTC on every store and read. Its range is limited to 1970-01-01 00:00:01 – 2106-02-07 06:28:15 UTC in 11.8, and only on 64-bit platforms [S1, S9]. | B |
| D7 | Text with no functional limit uses `MEDIUMTEXT`. Technical maximum: **16,777,215 bytes** of UTF-8. | MEDIUMTEXT has a 3-byte length prefix that counts bytes, so multibyte text holds fewer characters [S10]. This fills the gaps left by FR-WI-025, FR-CMT-012, FR-SPR-024, FR-REL-021 and FR-PRJ-010. Tag `desc` has no functional limit either (FR-TAG-002). | B |
| D8 | Bounded Unicode text (≤200 code points) uses `VARCHAR(200)`. The application trims (FR-PLT-006) and counts code points. | `VARCHAR(M)` counts characters [S10]. A multibyte character counts as one [S10, CHAR_LENGTH]. In strict mode a longer value is an error, not a truncation [S10]. | B |
| D9 | Plural table names. The identifiers `desc` and `release` are avoided. Column `description` maps to the API field `desc`. FK columns end in `_id`. | `DESC` and `RELEASE` are reserved [S8]. `TEXT` and `ACTION` are permitted unquoted [S8]. `name`, `value`, `kind`, `state`, `level`, `position` and `role` are not reserved [S8]. | B |
| D10 | References from a project database to central `users.id` (`comments.author_id`, `audit_log.user_id`) are logical. They have no FK. | Users are never deleted (FR-IDN-011), so references stay valid. Each project database stays self-contained for `DROP DATABASE` (FR-PRJ-009) and for backup. | B |
| D11 | Project database name = `planner_` + project name, at most 58 characters. | The identifier limit is 64 characters [S12]. FR-PRJ-005 caps the name at 50. | B |
| D12 | Boolean columns are `TINYINT UNSIGNED` (1 byte) with `CHECK (col IN (0,1))`. Every numeric column is `UNSIGNED`: no column holds negative values. | `BOOL` is a synonym for signed `TINYINT(1)` and does not restrict the range by itself [S20]; `UNSIGNED` states that no negative value is meaningful and doubles the positive range at the same size. | B |
| D13 | Every `REGEXP` in a CHECK starts with `(?-i)`. | `REGEXP` is case-insensitive except on binary strings. `(?-i)` forces a case-sensitive match [S17]. | B |
| D14 | `CHECK` is used for row-local rules. An expression that evaluates to NULL passes. | A row is rejected only when an expression returns false. NULL acceptance is shown by example [S3]. | B |
| D15 | Each database has a `schema_version` table. Migrations run over the central database and every project database. | Project databases are created at runtime (FR-PRJ-002) and must stay structurally identical. | B |
| D16 | Migrations run automatically at server startup, before requests are accepted. Each migration is a list of idempotent steps; `schema_version` records the version and the completed step, so a re-run resumes ([Schema versioning](#schema-versioning)). | DDL commits implicitly [S13], so a migration cannot be one transaction. Idempotent steps plus a recorded step make a crash at any point recoverable. | U |
| D17 | Three MariaDB accounts: runtime, provisioning, migration ([Privileges](#privileges)). | Least privilege: the account that serves requests cannot change the schema or the audit log (FR-AUD-006). | U |
| D18 | WEBSITE sessions are rows of the central table `sessions`. Only a SHA-256 hash of the session id is stored. | Server-side sessions can be deleted at once on logout and deactivation (FR-IDN-014). A copy of the database does not yield usable session ids. Rules: [security.md](security.md#website-sessions). | U (table name: B) |
| D19 | The migration account issues the runtime account's table-level grants as migration steps, in `planner` (including the initial schema) and in every project database. | Table privileges other than `CREATE` can be granted only on existing tables [S13]; the account that creates a table is the one able to grant on it at that moment. One account owns every schema change. | U |

### Required server settings

| Setting | Value | Reason |
|---------|-------|--------|
| `sql_mode` | Must include `STRICT_TRANS_TABLES`. This is the default [S11]. | Invalid ENUM values and over-long strings become errors, not silent coercion or truncation [S19, S10]. |
| `time_zone` (session) | `'+00:00'` on every connection. The default is `SYSTEM` [S11]. | D6. |
| `max_allowed_packet` | ≥ 64 MiB on the server. Default 16 MiB, range 1 KiB – 1 GiB [S11]. | One MEDIUMTEXT value can reach 16 MiB − 1 byte (D7). Text-protocol escaping can double it, and the statement adds overhead. 16 MiB cannot carry one maximal value. |
| `innodb_snapshot_isolation` | Default `ON` since 11.6.2 [S14]. Kept. | The application retries on error 1020 ([data-integrity-and-concurrency.md](data-integrity-and-concurrency.md#locking-protocol)). |
| `innodb_lock_wait_timeout` (session) | `5` seconds, set by the server on every connection. Global default 50; Global and Session scope, dynamic [S14]. | A request waits at most 5 s for a row lock; error 1205 is then retried under L4 ([data-integrity-and-concurrency.md](data-integrity-and-concurrency.md#locking-protocol)). 50 s would hold a request far beyond NFR-PERF-003. (U) |
| `max_connections` | ≥ runtime pool maximum + 3. Default 151 [S11]. | The pool ([architecture.md](architecture.md#database-access)), the instance guard connection, and one migration and one provisioning connection. The default covers the default pool of 32. |
| `table_definition_cache`, `table_open_cache` | Each ≥ the number of tables on the instance (13 per project + 8 central). Defaults 400 and 2,000 [S11]. (B) | NFR-PERF-001 requires ≥ 500 projects, that is ≥ 6,508 tables. Below that, table metadata is evicted and reloaded under normal traffic, against NFR-PERF-003. The performance test confirms the values ([testing.md](testing.md#performance-test)). |

## Central database `planner`

```sql
CREATE DATABASE planner CHARACTER SET utf8mb4 COLLATE utf8mb4_nopad_bin;

CREATE TABLE schema_version (
  id        TINYINT UNSIGNED  NOT NULL PRIMARY KEY,
  version   SMALLINT UNSIGNED NOT NULL,            -- last migration fully applied
  step      SMALLINT UNSIGNED NOT NULL DEFAULT 0,  -- steps of migration version + 1 applied
  applieddt DATETIME          NOT NULL,            -- UTC, last change to this row
  CONSTRAINT ck_schema_version_single CHECK (id = 1)
) ENGINE=InnoDB;

CREATE TABLE users (
  id       MEDIUMINT UNSIGNED NOT NULL AUTO_INCREMENT,
  idp_subject    VARCHAR(255) CHARACTER SET ascii COLLATE ascii_nopad_bin NOT NULL,
  name           VARCHAR(64) NULL,
  email          VARCHAR(64) NULL,
  displayname    VARCHAR(200) NULL,
  photo_key      BINARY(16) NULL,
  is_admin       TINYINT UNSIGNED NOT NULL DEFAULT 0,
  is_deactivated TINYINT UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (id),
  UNIQUE KEY uq_users_idp_subject (idp_subject),
  CONSTRAINT ck_users_displayname    CHECK (displayname IS NULL OR displayname <> ''),
  CONSTRAINT ck_users_is_admin       CHECK (is_admin IN (0,1)),
  CONSTRAINT ck_users_is_deactivated CHECK (is_deactivated IN (0,1))
) ENGINE=InnoDB;

CREATE TABLE projects (
  id    SMALLINT UNSIGNED NOT NULL AUTO_INCREMENT,
  name  VARCHAR(50) CHARACTER SET ascii COLLATE ascii_nopad_bin NOT NULL,
  state ENUM('provisioning','active','deleting') NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uq_projects_name (name),
  CONSTRAINT ck_projects_name CHECK (name REGEXP '(?-i)^[a-z][a-z0-9_]{0,49}$')
) ENGINE=InnoDB;

CREATE TABLE project_roles (
  id   MEDIUMINT UNSIGNED NOT NULL AUTO_INCREMENT,
  project_id SMALLINT UNSIGNED NOT NULL,
  user_id MEDIUMINT UNSIGNED NOT NULL,
  role       ENUM('owner','member','viewer') NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uq_project_roles (project_id, user_id),
  KEY ix_project_roles_user (user_id),
  CONSTRAINT fk_project_roles_project FOREIGN KEY (project_id) REFERENCES projects (id) ON DELETE CASCADE,
  CONSTRAINT fk_project_roles_user    FOREIGN KEY (user_id)    REFERENCES users (id)    ON DELETE RESTRICT
) ENGINE=InnoDB;

CREATE TABLE tokens (
  id    MEDIUMINT UNSIGNED NOT NULL AUTO_INCREMENT,
  user_id MEDIUMINT UNSIGNED NOT NULL,
  secret_hash BINARY(32) NOT NULL,
  deviceid    CHAR(24) CHARACTER SET ascii COLLATE ascii_nopad_bin NOT NULL,
  issueddt    DATETIME    NOT NULL,
  expiresdt   DATETIME    NOT NULL,
  revoked     TINYINT UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (id),
  UNIQUE KEY uq_tokens_secret_hash (secret_hash),
  KEY ix_tokens_user (user_id),
  CONSTRAINT fk_tokens_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE RESTRICT,
  CONSTRAINT ck_tokens_deviceid CHECK (deviceid REGEXP '(?-i)^[A-Za-z0-9]{24}$'),
  CONSTRAINT ck_tokens_expiry   CHECK (expiresdt > issueddt),
  CONSTRAINT ck_tokens_revoked  CHECK (revoked IN (0,1))
) ENGINE=InnoDB;

CREATE TABLE token_grants (
  token_id MEDIUMINT UNSIGNED NOT NULL,
  project_id SMALLINT UNSIGNED NOT NULL,
  level      ENUM('read','read+write') NOT NULL,
  PRIMARY KEY (token_id, project_id),
  KEY ix_token_grants_project (project_id),
  CONSTRAINT fk_token_grants_token   FOREIGN KEY (token_id)   REFERENCES tokens (id)   ON DELETE CASCADE,
  CONSTRAINT fk_token_grants_project FOREIGN KEY (project_id) REFERENCES projects (id) ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE sessions (
  id_hash    BINARY(32)         NOT NULL,
  user_id    MEDIUMINT UNSIGNED NOT NULL,
  csrf_token BINARY(32)         NOT NULL,
  createddt  DATETIME           NOT NULL,
  lastseendt DATETIME           NOT NULL,
  PRIMARY KEY (id_hash),
  KEY ix_sessions_user (user_id),
  KEY ix_sessions_lastseen (lastseendt),
  KEY ix_sessions_created (createddt),
  CONSTRAINT fk_sessions_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE RESTRICT,
  CONSTRAINT ck_sessions_seen CHECK (lastseendt >= createddt)
) ENGINE=InnoDB;

CREATE TABLE audit_log (
  id  MEDIUMINT UNSIGNED NOT NULL AUTO_INCREMENT,
  createddt DATETIME    NOT NULL,
  user_id MEDIUMINT UNSIGNED NOT NULL,
  deviceid  VARCHAR(24) CHARACTER SET ascii COLLATE ascii_nopad_bin NOT NULL,
  action    ENUM('create','modify','delete') NOT NULL,
  entity    ENUM('project','user','role','token') NOT NULL,
  entityid MEDIUMINT UNSIGNED NOT NULL,
  PRIMARY KEY (id),
  KEY ix_audit_log_entity (entity, entityid),
  KEY ix_audit_log_createddt (createddt),
  CONSTRAINT ck_audit_log_deviceid
    CHECK (deviceid = 'GUI' OR deviceid REGEXP '(?-i)^[A-Za-z0-9]{24}$')
) ENGINE=InnoDB;
```

Column notes (central):

| Column | Note |
|--------|------|
| `users.id` | The user `id` of FR-IDN-025, used for every external reference to a user (FR-IDN-026). |
| `users.idp_subject` | The IdP user id (FR-IDN-003, BR-IDN-001): the OIDC `sub`, at most 255 ASCII characters and unique within its issuer [S21]. There is one IdP (FR-IDN-001), so the issuer is server configuration, not a per-row column: storing it would repeat the same value in every row. FR-ACC-012's configuration value is a `sub` of that issuer. It is internal and never returned by project-level operations, user search or token reads (FR-IDN-027). Supporting several IdPs later would need an issuer column and a migration. |
| `users.name`, `email` | `VARCHAR(64)`, the FR-IDN-022 maximum. At profile creation the application truncates an IdP value to its first 64 code points and keeps an absent value NULL (FR-IDN-023). It applies no other validation, so there is no `<> ''` CHECK. FR-IDN-022 is applied by the application on planner edits. |
| `users.photo_key` | Key of the photo file (decision U): a random 128-bit value generated by planner on each upload, `BINARY(16)`. The file is named from it, and a new upload changes the key ([architecture.md](architecture.md#user-photo-storage)). |
| `users.is_admin` | Administrator status survives deactivation (FR-IDN-019). |
| `tokens.secret_hash` | SHA-256 of a random secret of at least 256 bits. The secret is shown once and never stored. A secret of that entropy makes an unsalted, fast, indexed digest sufficient for lookup (decision B). |
| `tokens.revoked` | Set by explicit revocation (FR-AUTH-004) and by deactivation (FR-IDN-012). Expiry is computed against `expiresdt` (FR-AUTH-026), never stored. |
| `token_grants` | Project deletion cascades (FR-AUTH-020, FR-PRJ-009). `tokens` rows are never deleted, so the token cascade does not fire in normal operation. |
| `sessions` | One row per WEBSITE session (D18). `id_hash` is the SHA-256 of the cookie value, a 256-bit random secret, so an unsalted indexed digest suffices, as for `tokens.secret_hash`. `csrf_token` is the per-session synchronizer token; it is stored in clear because it is embedded in served pages and is useless without the session cookie. `createddt` and `lastseendt` are UTC (NFR-DATA-001) and drive the absolute and idle timeouts. Sessions are not audited: FR-AUD-007 does not list them. `ix_sessions_lastseen` and `ix_sessions_created` serve the hourly purge, which runs one `DELETE` per timeout so that each uses its index ([security.md](security.md#website-sessions)). Rules: [security.md](security.md#website-sessions). |
| `audit_log` | Same structure as the project table (FR-AUD-007), so `user_id` has no FK. Consequence entries (FR-AUD-014) use this mapping, a technical choice (B): removed role → `delete` `role`; revoked token → `modify` `token`; removed token permission → `modify` `token`, one entry per token, because grants have no entity of their own. Administrator status changes and (de)activation are `modify` `user`. Role changes are `create`/`modify`/`delete` `role` with `entityid` = `project_roles.id`. Profile creation at first login (FR-IDN-017) is `create` `user`, attributed to that user. |

## Project database `planner_<project>`

Tables are created in this order: referenced tables first.

```sql
CREATE DATABASE planner_<project> CHARACTER SET utf8mb4 COLLATE utf8mb4_nopad_bin;

CREATE TABLE schema_version (       -- identical to the central table
  id        TINYINT UNSIGNED  NOT NULL PRIMARY KEY,
  version   SMALLINT UNSIGNED NOT NULL,
  step      SMALLINT UNSIGNED NOT NULL DEFAULT 0,
  applieddt DATETIME          NOT NULL,
  CONSTRAINT ck_schema_version_single CHECK (id = 1)
) ENGINE=InnoDB;

CREATE TABLE project_settings (
  id          TINYINT UNSIGNED NOT NULL PRIMARY KEY,
  description MEDIUMTEXT NULL,
  CONSTRAINT ck_project_settings_single CHECK (id = 1),
  CONSTRAINT ck_project_settings_description CHECK (description IS NULL OR description <> '')
) ENGINE=InnoDB;

CREATE TABLE sprints (
  id            SMALLINT UNSIGNED NOT NULL AUTO_INCREMENT,
  name          VARCHAR(200) COLLATE utf8mb4_uca1400_nopad_ai_ci NOT NULL,
  goal          MEDIUMTEXT NULL,
  startdt       DATE NOT NULL,
  enddt         DATE NOT NULL,
  state         ENUM('planned','active','closed') NOT NULL DEFAULT 'planned',
  active_marker TINYINT UNSIGNED AS (IF(state = 'active', 1, NULL)) PERSISTENT,
  PRIMARY KEY (id),
  UNIQUE KEY uq_sprints_name (name),
  UNIQUE KEY uq_sprints_one_active (active_marker),
  CONSTRAINT ck_sprints_name  CHECK (name <> ''),
  CONSTRAINT ck_sprints_goal  CHECK (goal IS NULL OR goal <> ''),
  CONSTRAINT ck_sprints_dates CHECK (enddt >= startdt)
) ENGINE=InnoDB;

CREATE TABLE releases (
  id          SMALLINT UNSIGNED NOT NULL AUTO_INCREMENT,
  name        VARCHAR(200) COLLATE utf8mb4_uca1400_nopad_ai_ci NOT NULL,
  description MEDIUMTEXT NULL,
  plandt      DATE NULL,
  releaseddt  DATE NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uq_releases_name (name),
  CONSTRAINT ck_releases_name CHECK (name <> ''),
  CONSTRAINT ck_releases_description CHECK (description IS NULL OR description <> '')
) ENGINE=InnoDB;

CREATE TABLE work_items (
  id    MEDIUMINT UNSIGNED NOT NULL AUTO_INCREMENT,
  kind        ENUM('epic','feature','userstory','task','defect','refactoring','spike','issue') NOT NULL,
  title       VARCHAR(200) NOT NULL,
  description MEDIUMTEXT NOT NULL,
  parent_id MEDIUMINT UNSIGNED NULL,
  parent_kind ENUM('epic','feature','userstory','task','defect','refactoring','spike','issue') NULL,
  planstart   DATE NULL,
  planend     DATE NULL,
  starteddt   DATETIME    NULL,
  closeddt    DATETIME    NULL,
  sprint_id   SMALLINT UNSIGNED NULL,
  release_id  SMALLINT UNSIGNED NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uq_work_items_id_kind (id, kind),
  KEY ix_work_items_parent  (parent_id, parent_kind, closeddt),
  KEY ix_work_items_sprint  (sprint_id, closeddt),
  KEY ix_work_items_release (release_id, closeddt),
  CONSTRAINT fk_work_items_parent FOREIGN KEY (parent_id, parent_kind)
    REFERENCES work_items (id, kind) ON DELETE RESTRICT ON UPDATE RESTRICT,
  CONSTRAINT fk_work_items_sprint  FOREIGN KEY (sprint_id)  REFERENCES sprints (id)
    ON DELETE RESTRICT ON UPDATE RESTRICT,
  CONSTRAINT fk_work_items_release FOREIGN KEY (release_id) REFERENCES releases (id)
    ON DELETE RESTRICT ON UPDATE RESTRICT,
  CONSTRAINT ck_work_items_title       CHECK (title <> ''),
  CONSTRAINT ck_work_items_description CHECK (description <> ''),
  CONSTRAINT ck_work_items_parent_pair
    CHECK ((parent_id IS NULL) = (parent_kind IS NULL)),
  CONSTRAINT ck_work_items_hierarchy CHECK (            -- BR-WI-014 / BR-WI-016
       parent_kind IS NULL
    OR (kind = 'feature'   AND parent_kind = 'epic')
    OR (kind = 'userstory' AND parent_kind = 'feature')
    OR (kind = 'task'      AND parent_kind IN ('userstory','defect','refactoring','spike'))),
  CONSTRAINT ck_work_items_sprint_kind  CHECK (sprint_id  IS NULL OR kind NOT IN ('epic','feature')),
  CONSTRAINT ck_work_items_release_kind CHECK (release_id IS NULL OR kind <> 'issue')
) ENGINE=InnoDB;

CREATE TABLE blockings (
  blocker_id MEDIUMINT UNSIGNED NOT NULL,
  blocker_kind ENUM('epic','feature','userstory','task','defect','refactoring','spike','issue')
               NOT NULL DEFAULT 'issue',
  blocked_id MEDIUMINT UNSIGNED NOT NULL,
  PRIMARY KEY (blocker_id, blocked_id),
  KEY ix_blockings_blocker (blocker_id, blocker_kind),
  KEY ix_blockings_blocked (blocked_id),
  CONSTRAINT fk_blockings_blocker FOREIGN KEY (blocker_id, blocker_kind)
    REFERENCES work_items (id, kind) ON DELETE CASCADE ON UPDATE RESTRICT,
  CONSTRAINT fk_blockings_blocked FOREIGN KEY (blocked_id)
    REFERENCES work_items (id) ON DELETE CASCADE ON UPDATE RESTRICT,
  CONSTRAINT ck_blockings_blocker_is_issue CHECK (blocker_kind = 'issue'),
  CONSTRAINT ck_blockings_not_self         CHECK (blocker_id <> blocked_id)
) ENGINE=InnoDB;

CREATE TABLE tags (
  id     SMALLINT UNSIGNED NOT NULL AUTO_INCREMENT,
  name        VARCHAR(50) CHARACTER SET ascii COLLATE ascii_nopad_bin NOT NULL,
  value       VARCHAR(50) CHARACTER SET ascii COLLATE ascii_nopad_bin NOT NULL DEFAULT '',
  color       MEDIUMINT UNSIGNED NULL,
  description MEDIUMTEXT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uq_tags_name_value (name, value),
  CONSTRAINT ck_tags_name  CHECK (name REGEXP '(?-i)^[a-z0-9_-]{1,50}$'),
  CONSTRAINT ck_tags_value CHECK (value = '' OR value REGEXP '(?-i)^[a-z0-9_-]{1,50}$'),
  CONSTRAINT ck_tags_description CHECK (description IS NULL OR description <> '')
) ENGINE=InnoDB;

CREATE TABLE work_item_tags (
  work_item_id MEDIUMINT UNSIGNED NOT NULL,
  tag_id  SMALLINT UNSIGNED NOT NULL,
  PRIMARY KEY (work_item_id, tag_id),
  KEY ix_work_item_tags_tag (tag_id),
  CONSTRAINT fk_work_item_tags_wi  FOREIGN KEY (work_item_id) REFERENCES work_items (id) ON DELETE CASCADE,
  CONSTRAINT fk_work_item_tags_tag FOREIGN KEY (tag_id)       REFERENCES tags (id)       ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE comments (
  id           INT UNSIGNED NOT NULL AUTO_INCREMENT,
  work_item_id MEDIUMINT UNSIGNED NOT NULL,
  kind         ENUM('HYPOTHESIS','TEST','FINDING','DECISION','PROGRESS','UPDATE','NOTE','QUESTION',
                    'FOLLOW-UP','BLOCKER','HANDOFF','ASSUMPTION','RISK','REVIEW','DEBT') NOT NULL,
  text         MEDIUMTEXT NOT NULL,
  author_id MEDIUMINT UNSIGNED NOT NULL,
  createddt    DATETIME    NOT NULL,
  editeddt     DATETIME    NULL,
  PRIMARY KEY (id),
  KEY ix_comments_work_item (work_item_id),
  CONSTRAINT fk_comments_work_item FOREIGN KEY (work_item_id) REFERENCES work_items (id) ON DELETE CASCADE,
  CONSTRAINT ck_comments_text CHECK (text <> '')
) ENGINE=InnoDB;

CREATE TABLE boards (
  id   SMALLINT UNSIGNED NOT NULL AUTO_INCREMENT,
  name VARCHAR(200) COLLATE utf8mb4_uca1400_nopad_ai_ci NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uq_boards_name (name),
  CONSTRAINT ck_boards_name CHECK (name <> '')
) ENGINE=InnoDB;

CREATE TABLE board_columns (
  id         SMALLINT UNSIGNED NOT NULL AUTO_INCREMENT,
  board_id   SMALLINT UNSIGNED NOT NULL,
  kind       ENUM('open','tag','closed') NOT NULL,
  tag_id SMALLINT UNSIGNED NULL,
  position    TINYINT UNSIGNED NULL,
  fixed_kind TINYINT UNSIGNED AS (IF(kind <> 'tag', kind + 0, NULL)) PERSISTENT,
  PRIMARY KEY (id),
  UNIQUE KEY uq_board_columns_tag   (board_id, tag_id),
  UNIQUE KEY uq_board_columns_fixed (board_id, fixed_kind),
  KEY ix_board_columns_tag (tag_id),
  CONSTRAINT fk_board_columns_board FOREIGN KEY (board_id) REFERENCES boards (id) ON DELETE CASCADE,
  CONSTRAINT fk_board_columns_tag   FOREIGN KEY (tag_id)   REFERENCES tags (id)   ON DELETE CASCADE,
  CONSTRAINT ck_board_columns_tag      CHECK ((kind = 'tag') = (tag_id IS NOT NULL)),
  CONSTRAINT ck_board_columns_position CHECK ((kind = 'tag') = (position IS NOT NULL))
) ENGINE=InnoDB;

CREATE TABLE board_cards (
  column_id    SMALLINT UNSIGNED NOT NULL,
  work_item_id MEDIUMINT UNSIGNED NOT NULL,
  position     INT UNSIGNED NOT NULL,
  PRIMARY KEY (column_id, work_item_id),
  KEY ix_board_cards_order (column_id, position),
  KEY ix_board_cards_work_item (work_item_id),
  CONSTRAINT fk_board_cards_column    FOREIGN KEY (column_id)    REFERENCES board_columns (id) ON DELETE CASCADE,
  CONSTRAINT fk_board_cards_work_item FOREIGN KEY (work_item_id) REFERENCES work_items (id)    ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE audit_log (
  id        INT UNSIGNED NOT NULL AUTO_INCREMENT,
  createddt DATETIME    NOT NULL,
  user_id MEDIUMINT UNSIGNED NOT NULL,
  deviceid  VARCHAR(24) CHARACTER SET ascii COLLATE ascii_nopad_bin NOT NULL,
  action    ENUM('create','modify','delete') NOT NULL,
  entity    ENUM('workitem','comment','tag','board','sprint','release','project') NOT NULL,
  entityid  INT UNSIGNED NOT NULL,
  PRIMARY KEY (id),
  KEY ix_audit_log_entity (entity, entityid),
  KEY ix_audit_log_createddt (createddt),
  CONSTRAINT ck_audit_log_deviceid
    CHECK (deviceid = 'GUI' OR deviceid REGEXP '(?-i)^[A-Za-z0-9]{24}$')
) ENGINE=InnoDB;
```

Column notes (project):

| Element | Note |
|---------|------|
| `work_items` parent FK | `(parent_id, parent_kind)` references `(id, kind)` through `uq_work_items_id_kind`. A referenced key must be an index or an index prefix [S5]. `parent_kind` therefore always equals the parent's real `kind`, and `ck_work_items_hierarchy` enforces BR-WI-014 and BR-WI-016 declaratively. Every allowed pair links distinct kinds of strictly decreasing rank, so no cycle or self-parent is possible (BR-WI-015). `ck_work_items_parent_pair` is required because a multi-column FK with any NULL column is not checked [S5]. `ON UPDATE RESTRICT` also blocks a `kind` change while children exist. FR-WI-010 itself is enforced by the application. |
| `work_items` parent `ON DELETE RESTRICT` | Not `SET NULL`. MariaDB 11.8 rejects (error 1901) any CHECK that references a column of an FK declared `ON UPDATE CASCADE/SET NULL/SET DEFAULT` or `ON DELETE SET NULL/SET DEFAULT`. `ON DELETE CASCADE` with `ON UPDATE RESTRICT` is accepted [S4]. The application clears the children's `parent` in the deleting transaction (FR-WI-021). `RESTRICT` makes omitting that step fail rather than corrupt ([data-integrity-and-concurrency.md](data-integrity-and-concurrency.md#work-item-deletion)). The rule is in the source code, not in the documentation. Reproduced on 11.8.9: "Function or expression '…' cannot be used in the CHECK clause" [S22]. InnoDB cascades do not validate CHECK (MDEV-30674, closed as duplicate of MDEV-22880, which is open) [S4]. |
| `work_items.sprint_id`, `release_id` | `RESTRICT` enforces FR-SPR-021 and FR-REL-016. The `closeddt` suffix in the indexes serves the "open assigned Work Item" checks (FR-SPR-009, FR-REL-005). `ix_work_items_parent` serves the FK and BR-CLS-001. |
| `blockings` | The CHECKs on FK columns are allowed because the FKs use `ON DELETE CASCADE` only [S4]. The cascade removes links on deletion (FR-WI-021). Cycle prevention (FR-WI-023) is in the application. |
| `tags.value` | `''` encodes "absent" (simple tag). A UNIQUE index admits multiple NULLs [S18], so a NULL value would let two simple `frontend` tags coexist (BR-TAG-002). `''` is free as a sentinel because FR-TAG-014 forbids an empty value. |
| `tags.color` | The 24-bit RGB value as `MEDIUMINT UNSIGNED` (3 bytes, range 0–16,777,215 = exactly `#000000`–`#FFFFFF`), so no CHECK is needed and values that differ only in letter case are the same number (FR-TAG-013, FR-TAG-015). The API parses `#RRGGBB` and returns it in uppercase. An out-of-range value is rejected (error 1264) [S22]. |
| `comments.author_id` | Logical reference (D10). Set once by the application (FR-CMT-011). |
| `sprints.active_marker` | It is `1` only for the active Sprint and NULL otherwise. With the UNIQUE index this allows at most one active Sprint (BR-SPR-001, FR-SPR-008), because NULLs do not collide [S18]. Indexes on PERSISTENT generated columns are supported [S16]. `IF()` in a PERSISTENT generated column loads, and a second active Sprint is rejected with error 1062 [S22]. |
| `board_columns.fixed_kind` | The ENUM index of `kind` (`kind + 0`: 1 = `open`, 3 = `closed`) for the fixed columns, NULL for tag columns; 1 byte. With `uq_board_columns_fixed`, a board has at most one `open` column and one `closed` column (FR-KBN-002). The application creates both with the board, and no path deletes them. Tag columns are ordered by `position` (FR-KBN-023). `open` is always first and `closed` always last, by definition. `uq_board_columns_tag` enforces FR-KBN-020. The tag FK cascade enforces FR-KBN-017 and FR-TAG-010. |
| `board_cards` | Persisted manual card order (FR-KBN-015). `position` is not unique: InnoDB checks uniqueness per row, which would make swaps fail. Ties are ordered by `work_item_id`. Maintenance: [data-integrity-and-concurrency.md](data-integrity-and-concurrency.md#board-card-maintenance). |
| `board_cards.position` growth | An entering card takes `MAX(position) + 1`. Positions are never compacted, except that a manual reorder renumbers its column `1…n`. A column's maximum therefore grows by at most 1 per card entry since its last manual reorder. `INT UNSIGNED` (max 4,294,967,295) allows over 429,000 entries per Work Item into one column at the NFR-PERF-005 volume of 10,000 Work Items, without any reorder. If the bound were reached, strict mode rejects the out-of-range value (error 1264) [S11, S22, S26], so the operation is refused and no position is corrupted; a manual reorder of the column restores the headroom. |
| `project_settings` | One row (`id = 1`) holding the project `desc` (FR-PRJ-004, FR-PRJ-010; decision U). Its edits are audited as `modify` `project`, with `entityid` = central `projects.id` (FR-AUD-015). |
| `audit_log` | Append-only (FR-AUD-006, BR-AUD-001): see [Privileges](#privileges). `'GUI'` cannot match the 24-character pattern, so the two `deviceid` forms are disjoint (FR-AUTH-016, FR-AUTH-023). Card reorder is `modify` `board`. Tag (un)assignment and blocking-link changes are `modify` `workitem` for each Work Item affected (FR-AUD-014). |

ENUM columns used in a composite FK (`kind`/`parent_kind`, `kind`/`blocker_kind`) are declared identically. FK columns must be of the same or similar type, with the same character set and collation [S5]. ENUM columns qualify: the FKs load and are enforced (a non-issue blocker is rejected with error 1452) [S22].

## Privileges

Three accounts (D17). Each is a separate credential in the server configuration ([operations.md](operations.md#configuration)).

| Account | Grants | Used by |
|---------|--------|---------|
| Runtime | In `planner`, per table: `SELECT, INSERT, UPDATE, DELETE` on `users`, `projects`, `project_roles`, `tokens`, `token_grants`, `sessions`; `SELECT, INSERT` on `audit_log`; `SELECT` on `schema_version`. Granted by migration steps in `planner` (D19). In each `planner_<project>`, per table, granted by the provisioning account at creation and by migration steps for tables added later: `SELECT, INSERT, UPDATE, DELETE` on every table except `audit_log` and `schema_version`; `SELECT, INSERT` on `audit_log`; `SELECT` on `schema_version`. | The connection pool, which serves every request; the instance guard ([architecture.md](architecture.md#startup-sequence)), which needs no privilege [S48]. |
| Provisioning | On `` `planner\_%`.* ``: `CREATE, DROP, SELECT, INSERT, UPDATE, DELETE` `WITH GRANT OPTION`. | Project creation, deletion and recovery ([data-integrity-and-concurrency.md](data-integrity-and-concurrency.md#project-provisioning-and-deletion)). A connection is opened for each sequence and closed after it. |
| Migration | On `planner`.* and on `` `planner\_%`.* ``: `CREATE, ALTER, DROP, INDEX`, and `SELECT, INSERT, UPDATE, DELETE` `WITH GRANT OPTION`. | Startup migration only, including the runtime grants on the tables it creates (D19). The connection is closed before requests are accepted ([Schema versioning](#schema-versioning)). |

Why each privilege is needed [S13, S23]:

| Privilege | Account | Reason |
|-----------|---------|--------|
| `CREATE` (database level) | Provisioning | `CREATE DATABASE`. At database level it also grants `CREATE` on every table of the database, so `CREATE TABLE` needs nothing more. Indexes declared inside `CREATE TABLE` need only `CREATE`, not `INDEX`. |
| `DROP` (database level) | Provisioning | `DROP DATABASE` on deletion and recovery. |
| `SELECT, INSERT, UPDATE, DELETE` `WITH GRANT OPTION` | Provisioning | A grantor needs `GRANT OPTION` and every privilege it grants. `REVOKE` needs the same. `INSERT` also writes the `schema_version` and `project_settings` rows of a new project. |
| `REFERENCES` | none | Documented as unused [S13]. FKs need no privilege of their own. |
| `CREATE, ALTER, DROP, INDEX` | Migration | `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE` and `CREATE INDEX` in migration steps. |
| `SELECT, INSERT, UPDATE, DELETE` `WITH GRANT OPTION` | Migration | Data migrations; reading and writing `schema_version`; granting the runtime account its table-level privileges on tables a migration creates, which needs `GRANT OPTION` and the privileges granted [S13]. |

Consequences:

- A database-level grant applies to all tables of the database, including later ones [S13]. It cannot exclude `audit_log`. The runtime account's privileges are therefore table-level grants.
- Table-level `SELECT`/`INSERT`/`UPDATE`/`DELETE` can be granted only after the table exists. Only `CREATE` is documented as grantable on tables that do not yet exist [S13]. A migration step that creates a table is followed, in the same migration, by an idempotent step that grants the runtime account its privileges on it (`GRANT` of a privilege already held changes nothing). This covers the initial central schema (D19).
- The provisioning and migration accounts hold database-level privileges on every `planner_<project>`, including `UPDATE` and `DELETE` on `audit_log` and the right to grant them, and the migration account's `DROP` permits `DROP DATABASE`. FR-AUD-006 therefore relies on these credentials being used only for their sequences. The runtime account, which handles every request, cannot change the audit log.
- Privileges on a database are not removed by `DROP DATABASE` [S13]. Table-level grants also remain in `mysql.tables_priv` after `DROP DATABASE` [S22]. Project deletion therefore revokes them explicitly.
- `GRANT` database names accept the `%` and `_` wildcards [S13]. `planner\_%` matches `planner_y` and does not match `plannerXa` [S22]. It does not match `planner`, which is granted separately.

## Schema versioning

- `schema_version` holds one row: `version`, the last migration fully applied, and `step`, the number of steps of migration `version + 1` already applied (D16).
- Each migration is an ordered list of steps embedded in the server binary, one list for the central database and one for project databases. Each step MUST be idempotent. MariaDB accepts `IF NOT EXISTS` on `ADD COLUMN`, `ADD INDEX`, `ADD FOREIGN KEY` and `CREATE INDEX`, and `IF EXISTS` on `DROP COLUMN`, `DROP INDEX`, `DROP FOREIGN KEY`, `DROP CONSTRAINT`, `CHANGE` and `MODIFY`; a clause whose condition triggers raises a warning, not an error [S24]. `ADD CONSTRAINT` (for example a CHECK) has no such clause [S24]: that step first queries `information_schema` and is skipped when the constraint exists. DML steps are written so that a second run changes nothing.
- After each step the migration account sets `step = step + 1` and `applieddt`. After the last step it sets `version = version + 1, step = 0`. A crash between a step and its `step` update re-runs that step on the next start, which is safe because the step is idempotent.
- DDL statements commit implicitly [S13]. A migration is therefore not atomic across its steps, and it is not atomic across databases.
- Order at startup: central database, then recovery of `provisioning` and `deleting` projects, then every project database in ascending `projects.id` ([architecture.md](architecture.md#startup-sequence)). After recovery every remaining project is `active`, so every project database is migrated.
- Project creation builds the schema of the current version directly and writes `version` = current, `step` = 0.
- If any database ends at a version other than the one the binary expects, including a newer one, the server exits without accepting requests.

## Deviations from the brief

| Brief | This file | Why |
|-------|-----------|-----|
| Parent FK `ON DELETE SET NULL` together with the BR-WI-014 CHECK | `ON DELETE RESTRICT`. The application clears `parent` first. | MariaDB 11.8 forbids that combination [S4]. This fallback keeps the CHECK declarative, which neither an application-only check nor a trigger would do. Triggers are not fired by FK actions either [S5]. |
| `ascii_bin` | `ascii_nopad_bin` | `ascii_bin` is not NO PAD [S6]. Trailing spaces would compare equal, a risk for `idp_subject` lookups. |
| Rationale "TIMESTAMP 2038 limit" | Range 1970–2106 (64-bit only) plus time-zone conversion | In 11.5 and later the upper bound is 2106 [S1, S9]. |
| Index on `sprint_id`, `release_id` | `(sprint_id, closeddt)`, `(release_id, closeddt)`, `(parent_id, parent_kind, closeddt)` | The FK requires the first columns. The suffix serves the closing checks. |
| Empty-string CHECKs | Added: `<> ''` on required text; `IS NULL OR <> ''` on optional text | Declarative backstop for FR-PLT-006: required text is never empty (FR-WI-005, FR-CMT-005, FR-SPR-003, FR-REL-011, FR-KBN-022), and optional text empty after trimming is NULL, never `''`. |

## Sources

Consulted 2026-11-23. Documentation pages describe current releases. Version-specific statements are as labelled.

| Id | Source |
|----|--------|
| S1 | MariaDB, "Changes and Improvements in MariaDB 11.8": <https://mariadb.com/docs/release-notes/community-server/11.8/what-is-mariadb-118> |
| S2 | MariaDB 11.8.10 release notes: <https://mariadb.com/docs/release-notes/community-server/11.8/11.8.10> |
| S3 | MariaDB KB, "CONSTRAINT": <https://mariadb.com/kb/en/constraint/> |
| S4 | MariaDB server source, tags `mariadb-11.8.2` and `mariadb-11.8.10`: `sql/sql_alter.cc` `Alter_info::check_vcol_field`, `sql/table.h` `enum_fk_option`, `sql/field.cc` `check_expression` (<https://github.com/MariaDB/server>). Jira MDEV-30606, MDEV-30674, MDEV-31769 (<https://jira.mariadb.org>). |
| S5 | MariaDB, "Foreign Keys": <https://mariadb.com/docs/server/ha-and-performance/optimization-and-tuning/optimization-and-indexes/foreign-keys> |
| S6 | MariaDB, "Supported Character Sets and Collations": <https://mariadb.com/docs/server/reference/data-types/string-data-types/character-sets/supported-character-sets-and-collations> |
| S7 | MariaDB, "AUTO_INCREMENT": <https://mariadb.com/docs/server/reference/data-types/auto_increment>. "AUTO_INCREMENT Handling in InnoDB": <https://mariadb.com/docs/server/server-usage/storage-engines/innodb/auto_increment-handling-in-innodb>. The 10.2.4 origin (MDEV-6076) is confirmed only by the Jira fix version and secondary sources. It is irrelevant given D1. |
| S8 | MariaDB, "Reserved Words": <https://mariadb.com/docs/server/reference/sql-structure/sql-language-structure/reserved-words> |
| S9 | MariaDB, "TIMESTAMP" and "DATETIME": <https://mariadb.com/docs/server/reference/data-types/date-and-time-data-types/timestamp>, <https://mariadb.com/docs/server/reference/data-types/date-and-time-data-types/datetime> |
| S10 | MariaDB, "VARCHAR", "TEXT", "MEDIUMTEXT", "CHAR_LENGTH": under <https://mariadb.com/docs/server/reference/data-types/string-data-types/> and <https://mariadb.com/docs/server/reference/sql-functions/string-functions/char_length> |
| S11 | MariaDB, "Server System Variables" (`max_allowed_packet`, `sql_mode`, `time_zone`): <https://mariadb.com/docs/server/server-management/variables-and-modes/server-system-variables> |
| S12 | MariaDB, "Identifier Names": <https://mariadb.com/docs/server/reference/sql-structure/sql-language-structure/identifier-names> |
| S13 | MariaDB, "GRANT", "DROP DATABASE", "CREATE DATABASE", "SQL statements that cause an implicit commit": under <https://mariadb.com/docs/server/reference/sql-statements/> |
| S14 | MariaDB, "InnoDB System Variables" (`innodb_snapshot_isolation`, `innodb_autoinc_lock_mode`, `innodb_lock_wait_timeout`: Global and Session, dynamic, default 50, a timeout rolls back the statement, not the transaction): <https://mariadb.com/docs/server/server-usage/storage-engines/innodb/innodb-system-variables> |
| S15 | MariaDB, "SELECT" (locking clauses): <https://mariadb.com/docs/server/reference/sql-statements/data-manipulation/selecting-data/select> |
| S16 | MariaDB, "Generated Columns": <https://mariadb.com/docs/server/reference/sql-statements/data-definition/create/generated-columns> |
| S17 | MariaDB, "REGEXP": <https://mariadb.com/docs/server/reference/sql-functions/string-functions/regular-expressions-functions/regexp> |
| S18 | MariaDB, "UNIQUE" (InnoDB schema design): <https://mariadb.com/docs/storage-engines/innodb/schema-design/indexes/unique/> |
| S19 | MariaDB, "ENUM": <https://mariadb.com/docs/server/reference/data-types/string-data-types/enum> |
| S20 | MariaDB, "BOOLEAN": <https://mariadb.com/docs/server/reference/data-types/numeric-data-types/boolean> |
| S21 | OpenID Connect Core 1.0, §2 (`iss`, `sub`), §3.1.2.1 (`nonce` OPTIONAL in the code flow), §3.1.3.7 (ID Token validation), §5.4 (scope claims returned from the UserInfo Endpoint when an Access Token is issued), §5.7, §9 (default client authentication `client_secret_basic`): <https://openid.net/specs/openid-connect-core-1_0.html> |
| S22 | Local test on MariaDB 11.8.9, 2026-11-23, reported by the coordinator. Results: the error 1901 CHECK/FK restriction; ENUM composite FKs enforced; `IF()` generated column and `uq_sprints_one_active`; `DATETIME` vs `TIMESTAMP` under `time_zone`; `\_` in `GRANT`; table-level grants persist after `DROP DATABASE`; `utf8mb4_uca1400_nopad_ai_ci` equalities; auto-increment persists across restart (max id 4 deleted, restart, next id 5). Both DDL scripts in this file load without error. Also confirmed: the hierarchy CHECK rejects task→epic, `s1` is rejected against `S1`, and a second simple `frontend` tag is rejected. |
| S23 | MariaDB, "REVOKE" (requires `GRANT OPTION` and the privileges revoked): <https://mariadb.com/docs/server/reference/sql-statements/account-management-sql-statements/revoke> |
| S24 | MariaDB, "ALTER TABLE" (`IF [NOT] EXISTS` clauses) and "CREATE INDEX": <https://mariadb.com/docs/server/reference/sql-statements/data-definition/alter/alter-table> |
| S25 | MariaDB, "GET_LOCK" (return values, server-wide names, release when the connection terminates): <https://mariadb.com/docs/server/reference/sql-functions/secondary-functions/miscellaneous-functions/get_lock> |
| S26 | MariaDB, "MariaDB Error Code Reference" (1020, 1062, 1205, 1213, 1264, 1451, 1452, 4025): <https://mariadb.com/docs/server/reference/error-codes/mariadb-error-code-reference> |
| S27 | MariaDB, mariadb-backup: "Overview" (hot online InnoDB backups; base privileges `RELOAD, PROCESS, LOCK TABLES, BINLOG MONITOR`), "Full Backup and Restore", "How mariadb-backup Works" (point-in-time consistency after prepare; global read lock unless `--no-lock`): under <https://mariadb.com/docs/server/server-usage/backup-and-restore/mariadb-backup/> |
| S28 | crates.io API, latest stable versions on 2026-11-23: axum 0.8.9, tokio 1.54.1, sqlx 0.9.1, askama 0.16.1, clap 4.6.9, tracing 0.1.44, tracing-subscriber 0.3.23, time 0.3.55, openidconnect 4.1.0, reqwest 0.13.5, image 0.25.10, governor 0.10.4, tower_governor 0.8.0, directories 6.0.0, testcontainers 0.28.0, testcontainers-modules 0.15.0, cookie 0.18.3, axum-extra 0.12.6 (`cookie` ^0.18.0 optional), windows-sys 0.61.2, windows-acl 0.3.0 (last release 2021-01-11); and dependency requirements of openidconnect 4.1.0 (`oauth2` ^5.0.0), oauth2 5.0.0 (`reqwest` ^0.12, optional), tower_governor 0.8.0 (`axum` ^0.8 optional, `governor` ^0.10.0), testcontainers-modules 0.15.0 (`testcontainers` ^0.27.0): <https://crates.io/api/v1/crates/{name}> |
| S29 | sqlx 0.9.1: README (PostgreSQL, MySQL, MariaDB, SQLite; `query!` needs `DATABASE_URL` or cached data), docs.rs `MySqlConnectOptions` (`timezone` default `+00:00`, `statement_cache_capacity` default 100, LRU), `MySqlSslMode` (`Disabled`, `Preferred` default, `Required`, `VerifyCa`, `VerifyIdentity`), MySQL type mapping for the `time` feature (`Date` ↔ `DATE`, `PrimitiveDateTime` ↔ `DATETIME`), `PoolOptions` (`after_connect`), `query!` (string literal only), sqlx-cli README (`cargo sqlx prepare`, `.sqlx`, `--check`): <https://github.com/launchbadge/sqlx/tree/v0.9.1>, <https://docs.rs/sqlx/0.9.1> |
| S30 | axum 0.8.9 documentation (works with tokio and hyper; middleware is `tower::Service`): <https://docs.rs/axum/0.8.9/axum/> |
| S31 | askama 0.16.1 documentation (templates compiled into the crate at build time): <https://docs.rs/crate/askama/0.16.1> |
| S32 | clap 4.6.9 documentation, `Command` (nested subcommands; `get_subcommands`, `get_arguments` and other getters): <https://docs.rs/clap/4.6.9/clap/struct.Command.html> |
| S33 | tracing-subscriber 0.3.23, `fmt` and `fmt::format::Json` (newline-delimited JSON; crate features `fmt`, `json`, `std`): <https://docs.rs/tracing-subscriber/0.3.23/tracing_subscriber/fmt/format/struct.Json.html> |
| S34 | htmx: <https://htmx.org/> (quick start uses 2.0.11; 4.0 released, not `latest` on npm until 2027), npm registry `htmx.org` dist-tags (`latest` 2.0.11, `next` 4.0.2), GitHub release v4.0.2 (2026-11-18) |
| S35 | SortableJS release 1.15.7, 2026-02-11: <https://github.com/SortableJS/Sortable/releases>; npm `sortablejs` `latest` 1.15.7 |
| S36 | OpenID Connect Discovery 1.0 (§3 defaults, §4 `/.well-known/openid-configuration`, Discovery optional for providers, §4.3 issuer match): <https://openid.net/specs/openid-connect-discovery-1_0.html> |
| S37 | RFC 7636, PKCE (§4.1 verifier 43–128 characters, §4.2 S256, §5 compatibility): <https://www.rfc-editor.org/rfc/rfc7636> |
| S38 | RFC 9457, Problem Details for HTTP APIs (§3 `application/problem+json`, §3.1 members, §3.2 extension members): <https://www.rfc-editor.org/rfc/rfc9457> |
| S39 | RFC 3339 (§2 `Z`, §5.6 `full-date`, `date-time`): <https://www.rfc-editor.org/rfc/rfc3339> |
| S40 | RFC 4648 (§3.2 padding, §5 base64url alphabet): <https://www.rfc-editor.org/rfc/rfc4648> |
| S41 | RFC 6750, Bearer Token Usage (§2.1 `b64token`, §3.1 `invalid_token` → 401, §5.3 TLS): <https://www.rfc-editor.org/rfc/rfc6750> |
| S42 | RFC 6585 (§4, 429 and `Retry-After`): <https://www.rfc-editor.org/rfc/rfc6585> |
| S43 | draft-ietf-httpbis-rfc6265bis-22 (2025-12-01), §4.1.2.7 and §5.6.7.1 (`SameSite=Lax`), §4.1.3.2 (`__Host-` prefix); in the RFC Editor queue, not yet an RFC on 2026-11-23: <https://datatracker.ietf.org/doc/draft-ietf-httpbis-rfc6265bis/> |
| S45 | XDG Base Directory Specification 0.8 (`$XDG_CONFIG_HOME`, default `$HOME/.config`): <https://specifications.freedesktop.org/basedir/latest/> |
| S46 | Keycloak release 26.8.2, 2026-11-05: <https://github.com/keycloak/keycloak/releases> |
| S47 | systemd 262 manual: systemd.service (`Restart=on-failure`), systemd.exec (`EnvironmentFile=`): <https://www.freedesktop.org/software/systemd/man/latest/> |
| S48 | Local test on MariaDB 11.8.9 (docker image `mariadb:11.8`), 2026-11-23, reported by the coordinator. Results: auto-increment exhaustion returns `ERROR 167 (22003): Out of range value for column 'id' at row 1` to the client; `GET_LOCK('…', 0)` succeeds for an account holding only `USAGE`; at `REPEATABLE-READ` with `innodb_snapshot_isolation=1`, `SELECT … FOR UPDATE` on an absent value of a UNIQUE index blocks a concurrent `INSERT` of that value (1205 after a 2 s lock wait timeout), and `LOCK IN SHARE MODE` on an absent value above the maximum (supremum gap) also blocks a concurrent insert of that value. |
| S49 | image 0.25.10 documentation: `ImageReader::with_guessed_format`, `guess_format` (magic bytes), `codecs::webp::WebPEncoder` (lossless VP8L only; lossy needs libwebp, e.g. the `webp` crate; feature `webp`), `WebPDecoder` (`has_animation`, `into_frames`), `PngDecoder` (`is_apng`, `apng`), `Limits` (`#[non_exhaustive]`, public `max_image_width`, `max_image_height`, `max_alloc`; defaults no dimension limit and 512 MiB; `max_alloc` non-strict), `error::LimitError` (`ImageError::Limits`): <https://docs.rs/image/0.25.10/image/> |
| S50 | governor 0.10.4 documentation, `RateLimiter` (`keyed`, `check_key` returning `NotUntil`, `retain_recent`): <https://docs.rs/governor/0.10.4/governor/struct.RateLimiter.html> |
| S51 | tower_governor 0.8.0 documentation (axum ^0.8 optional feature; `SmartIpKeyExtractor` reads `x-forwarded-for`, `x-real-ip`, `forwarded`; custom `KeyExtractor`): <https://docs.rs/tower_governor/0.8.0/tower_governor/> |
| S52 | openidconnect 4.1.0 documentation: PKCE, `discover_async`, `Client::new` and endpoint setters, nonce verification, async HTTP client closures, "configure the HTTP client not to follow redirects"; features (`reqwest`, `rustls-tls` default, via `oauth2` 5): <https://docs.rs/openidconnect/4.1.0/openidconnect/> |
| S53 | reqwest 0.13.5 documentation, features `rustls`, `json`, `default-tls`: <https://docs.rs/reqwest/0.13.5/reqwest/> |
| S54 | directories 6.0.0 documentation, `ProjectDirs::from` (returns `None` without a home directory) and `config_dir` per platform: <https://docs.rs/directories/6.0.0/directories/struct.ProjectDirs.html> |
| S55 | cookie 0.18.3 documentation, `PrivateJar` (authenticated encryption assuring confidentiality, integrity and authenticity) and `Key` (`from`: at least 64 bytes, panics otherwise; `try_from`): <https://docs.rs/cookie/0.18.3/cookie/>. axum-extra 0.12.6, `PrivateCookieJar` (features `cookie`, `cookie-private`; key via `FromRef`): <https://docs.rs/axum-extra/0.12.6/axum_extra/extract/cookie/struct.PrivateCookieJar.html> |
| S56 | testcontainers 0.28.0 documentation, `GenericImage::new(name, tag)`, `AsyncRunner::start`: <https://docs.rs/testcontainers/0.28.0/testcontainers/> |
| S57 | k6 v2.4.0 release, 2026-11-12: <https://github.com/grafana/k6/releases>; Grafana k6 documentation, "Thresholds" (`p(95)<200`, tag-scoped thresholds, non-zero exit on failure): <https://grafana.com/docs/k6/latest/using-k6/thresholds/> |
| S58 | Local test on MariaDB 11.8.9, 2026-11-23, reported by the coordinator, on a `utf8mb4_nopad_bin` column with `col COLLATE utf8mb4_uca1400_ai_ci LIKE … ESCAPE '!'`: `'Versão'` LIKE `'%versao%'` = 1; `'ANA SILVA'` LIKE `'%ana s%'` = 1; `'a_b'` LIKE `'%!_%'` = 1; `'axb'` LIKE `'%!_%'` = 0; `'José Ribeiro'` LIKE `'%jose%'` = 1 and `'%ribe%'` = 1; `'Ｊｏｓé'` LIKE `'%jose%'` = 1; `'Straße'` LIKE `'%strasse%'` = 0; `'Straße'` LIKE `'%ße%'` = 1. |
| S59 | windows-sys 0.61.2 documentation, `Win32::Security::Authorization` (`SetNamedSecurityInfoW`, `GetNamedSecurityInfoW`, `SetEntriesInAclW`, `EXPLICIT_ACCESS_W`, `SE_FILE_OBJECT`): <https://docs.rs/windows-sys/0.61.2/windows_sys/Win32/Security/Authorization/index.html> |
