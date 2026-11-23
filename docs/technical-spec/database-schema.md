---
title: Database schema
status: draft
last-reviewed: 2026-11-23
related: [domain-model.md, data-integrity-and-concurrency.md, open-questions.md, ../functional-spec/multi-project.md, ../functional-spec/overview.md]
---

# Database schema

## Overview

Persistence is MariaDB (NFR-ARCH-004). It uses one central database, `planner` (FR-PRJ-003), and one database per project, `planner_<project>` (FR-PRJ-002, FR-PRJ-004). No Rust driver or ORM has been chosen. This file is independent of that choice.

## Decisions

Decider key: **U** = user, 2026-11-23. **B** = Claude (design brief), accepted by the user on 2026-11-23. **[S22]** = local test on MariaDB 11.8.9, 2026-11-23.

| # | Decision | Rationale | By |
|---|----------|-----------|----|
| D1 | The minimum server version is **MariaDB 11.8 LTS**. | 11.8 is LTS: first GA 11.8.2 on 2025-06-04, maintained until June 2028 [S1]. Latest at review: 11.8.10, 2026-11-16 [S2]. | U |
| D2 | Ids are native `AUTO_INCREMENT BIGINT UNSIGNED`. Gaps are acceptable and ids are never reused. | InnoDB keeps the counter across restarts. Deleted values are never reused. Rolled-back and failed inserts leave gaps [S7]. The counter survives a restart [S22]. Matches FR-WI-003, FR-CMT-004, FR-SPR-015 and FR-REL-010 (increasing, never reused, gaps allowed). All ids share one width so that the single audit column `entityid` can hold any of them. | U (width: B) |
| D3 | Engine InnoDB. Each database's default is `CHARACTER SET utf8mb4 COLLATE utf8mb4_nopad_bin`. | FKs are supported only by InnoDB [S5]. An explicit default stops the 11.8 server default, `utf8mb4_uca1400_ai_ci` [S1], which ignores case and accents, from applying silently. `nopad_bin` compares exactly, including trailing spaces [S6, S10]. | B |
| D4 | Machine identifiers (project name, tag name and value, `deviceid`, `color`, IdP issuer and subject) use `CHARACTER SET ascii COLLATE ascii_nopad_bin`. | Their alphabets are ASCII (FR-PRJ-005, FR-TAG-012, FR-AUTH-023, FR-TAG-013, [S21]). NO PAD keeps `'x'` and `'x '` distinct [S6], which matters for IdP subjects (identity confusion). | B (NO PAD instead of `ascii_bin`: see [deviations](#deviations-from-the-brief)) |
| D5 | Sprint, Release and Board `name` use `utf8mb4_uca1400_nopad_ai_ci`. | Uniqueness ignores letter case and diacritics (FR-SPR-016, FR-REL-012, FR-KBN-022) (U). UCA-14.0.0 collations exist since 10.10.1 [S6]. `'Versão' = 'versao'` [S22]. The equality goes beyond case and diacritics: `'ß' = 'ss'` and full-width `'Ａ' = 'a'` also hold [S22]. This is acceptable under the decision. | U / B |
| D6 | Date-only fields are `DATE`. Date-times are `DATETIME(6)` holding UTC, produced by the application. Every connection sets `time_zone = '+00:00'`. | NFR-DATA-001 (INVIOLABLE), FR-PLT-005. `DATETIME` covers years 1000–9999 with 0–6 fractional digits [S9]. It is not shifted by `time_zone`, while `TIMESTAMP` is: a value inserted at `+00:00` and read at `+05:00` returns 12:00 from `DATETIME(6)` and 17:00 from `TIMESTAMP(6)` [S9, S22]. `TIMESTAMP` is rejected: it converts between the session time zone and UTC on every store and read. Its range is limited to 1970-01-01 00:00:01 – 2106-02-07 06:28:15 UTC in 11.8, and only on 64-bit platforms [S1, S9]. | B |
| D7 | Text with no functional limit uses `MEDIUMTEXT`. Technical maximum: **16,777,215 bytes** of UTF-8. | MEDIUMTEXT has a 3-byte length prefix that counts bytes, so multibyte text holds fewer characters [S10]. This fills the gaps left by FR-WI-025, FR-CMT-012, FR-SPR-024, FR-REL-021 and FR-PRJ-010. Tag `desc` has no functional limit either (FR-TAG-002). | B |
| D8 | Bounded Unicode text (≤200 code points) uses `VARCHAR(200)`. The application trims (FR-PLT-006) and counts code points. | `VARCHAR(M)` counts characters [S10]. A multibyte character counts as one [S10, CHAR_LENGTH]. In strict mode a longer value is an error, not a truncation [S10]. | B |
| D9 | Plural table names. The identifiers `desc` and `release` are avoided. Column `description` maps to the API field `desc`. FK columns end in `_id`. | `DESC` and `RELEASE` are reserved [S8]. `TEXT` and `ACTION` are permitted unquoted [S8]. `name`, `value`, `kind`, `state`, `level`, `position` and `role` are not reserved [S8]. | B |
| D10 | References from a project database to central `users.id` (`comments.author_id`, `audit_log.user_id`) are logical. They have no FK. | Users are never deleted (FR-IDN-011), so references stay valid. Each project database stays self-contained for `DROP DATABASE` (FR-PRJ-009) and for backup. | B |
| D11 | Project database name = `planner_` + project name, at most 58 characters. | The identifier limit is 64 characters [S12]. FR-PRJ-005 caps the name at 50. | B |
| D12 | Boolean columns are `BOOL` with `CHECK (col IN (0,1))`. | `BOOL` is a synonym for `TINYINT(1)` and does not restrict the range by itself [S20]. | B |
| D13 | Every `REGEXP` in a CHECK starts with `(?-i)`. | `REGEXP` is case-insensitive except on binary strings. `(?-i)` forces a case-sensitive match [S17]. | B |
| D14 | `CHECK` is used for row-local rules. An expression that evaluates to NULL passes. | A row is rejected only when an expression returns false. NULL acceptance is shown by example [S3]. | B |
| D15 | Each database has a `schema_version` table. Migrations run over the central database and every project database. | Project databases are created at runtime (FR-PRJ-002) and must stay structurally identical. | B |

### Required server settings

| Setting | Value | Reason |
|---------|-------|--------|
| `sql_mode` | Must include `STRICT_TRANS_TABLES`. This is the default [S11]. | Invalid ENUM values and over-long strings become errors, not silent coercion or truncation [S19, S10]. |
| `time_zone` (session) | `'+00:00'` on every connection. The default is `SYSTEM` [S11]. | D6. |
| `max_allowed_packet` | ≥ 64 MiB on the server. Default 16 MiB, range 1 KiB – 1 GiB [S11]. | One MEDIUMTEXT value can reach 16 MiB − 1 byte (D7). Text-protocol escaping can double it, and the statement adds overhead. 16 MiB cannot carry one maximal value. |
| `innodb_snapshot_isolation` | Default `ON` since 11.6.2 [S14]. Kept. | The application retries on error 1020 ([data-integrity-and-concurrency.md](data-integrity-and-concurrency.md#locking-protocol)). |

## Central database `planner`

```sql
CREATE DATABASE planner CHARACTER SET utf8mb4 COLLATE utf8mb4_nopad_bin;

CREATE TABLE schema_version (
  id        TINYINT UNSIGNED NOT NULL PRIMARY KEY,
  version   INT UNSIGNED     NOT NULL,
  applieddt DATETIME(6)      NOT NULL,
  CONSTRAINT ck_schema_version_single CHECK (id = 1)
) ENGINE=InnoDB;

CREATE TABLE users (
  id             BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  idp_issuer     VARCHAR(255) CHARACTER SET ascii COLLATE ascii_nopad_bin NOT NULL,
  idp_subject    VARCHAR(255) CHARACTER SET ascii COLLATE ascii_nopad_bin NOT NULL,
  name           VARCHAR(64) NULL,
  email          VARCHAR(64) NULL,
  displayname    VARCHAR(200) NULL,
  photo_key      VARCHAR(512) NULL,
  is_admin       BOOL NOT NULL DEFAULT 0,
  is_deactivated BOOL NOT NULL DEFAULT 0,
  PRIMARY KEY (id),
  UNIQUE KEY uq_users_idp (idp_issuer, idp_subject),
  CONSTRAINT ck_users_displayname    CHECK (displayname IS NULL OR displayname <> ''),
  CONSTRAINT ck_users_is_admin       CHECK (is_admin IN (0,1)),
  CONSTRAINT ck_users_is_deactivated CHECK (is_deactivated IN (0,1))
) ENGINE=InnoDB;

CREATE TABLE projects (
  id    BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  name  VARCHAR(50) CHARACTER SET ascii COLLATE ascii_nopad_bin NOT NULL,
  state ENUM('provisioning','active','deleting') NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uq_projects_name (name),
  CONSTRAINT ck_projects_name CHECK (name REGEXP '(?-i)^[a-z][a-z0-9_]{0,49}$')
) ENGINE=InnoDB;

CREATE TABLE project_roles (
  id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  project_id BIGINT UNSIGNED NOT NULL,
  user_id    BIGINT UNSIGNED NOT NULL,
  role       ENUM('owner','member','viewer') NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uq_project_roles (project_id, user_id),
  KEY ix_project_roles_user (user_id),
  CONSTRAINT fk_project_roles_project FOREIGN KEY (project_id) REFERENCES projects (id) ON DELETE CASCADE,
  CONSTRAINT fk_project_roles_user    FOREIGN KEY (user_id)    REFERENCES users (id)    ON DELETE RESTRICT
) ENGINE=InnoDB;

CREATE TABLE tokens (
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  user_id     BIGINT UNSIGNED NOT NULL,
  secret_hash BINARY(32) NOT NULL,
  deviceid    CHAR(24) CHARACTER SET ascii COLLATE ascii_nopad_bin NOT NULL,
  issueddt    DATETIME(6) NOT NULL,
  expiresdt   DATETIME(6) NOT NULL,
  revoked     BOOL NOT NULL DEFAULT 0,
  PRIMARY KEY (id),
  UNIQUE KEY uq_tokens_secret_hash (secret_hash),
  KEY ix_tokens_user (user_id),
  CONSTRAINT fk_tokens_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE RESTRICT,
  CONSTRAINT ck_tokens_deviceid CHECK (deviceid REGEXP '(?-i)^[A-Za-z0-9]{24}$'),
  CONSTRAINT ck_tokens_expiry   CHECK (expiresdt > issueddt),
  CONSTRAINT ck_tokens_revoked  CHECK (revoked IN (0,1))
) ENGINE=InnoDB;

CREATE TABLE token_grants (
  token_id   BIGINT UNSIGNED NOT NULL,
  project_id BIGINT UNSIGNED NOT NULL,
  level      ENUM('read','read+write') NOT NULL,
  PRIMARY KEY (token_id, project_id),
  KEY ix_token_grants_project (project_id),
  CONSTRAINT fk_token_grants_token   FOREIGN KEY (token_id)   REFERENCES tokens (id)   ON DELETE CASCADE,
  CONSTRAINT fk_token_grants_project FOREIGN KEY (project_id) REFERENCES projects (id) ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE audit_log (
  id        BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  createddt DATETIME(6) NOT NULL,
  user_id   BIGINT UNSIGNED NOT NULL,
  deviceid  VARCHAR(24) CHARACTER SET ascii COLLATE ascii_nopad_bin NOT NULL,
  action    ENUM('create','modify','delete') NOT NULL,
  entity    ENUM('project','user','role','token') NOT NULL,
  entityid  BIGINT UNSIGNED NOT NULL,
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
| `users.idp_issuer`, `idp_subject` | The IdP user id (FR-IDN-003, BR-IDN-001). `iss` + `sub` is the only stable unique identifier of an end user. `sub` is at most 255 ASCII characters and unique within the issuer [S21]. OIDC sets no length for `iss`. The 255 cap is a project choice. FR-ACC-012's configuration value is a `sub` of the configured issuer. |
| `users.name`, `email` | `VARCHAR(64)`, the FR-IDN-022 maximum. At profile creation the application truncates an IdP value to its first 64 code points and keeps an absent value NULL (FR-IDN-023). It applies no other validation, so there is no `<> ''` CHECK. FR-IDN-022 is applied by the application on planner edits. |
| `users.photo_key` | Opaque key into an external store (decision U). The backend is open: [TOQ-001](open-questions.md#toq-001). |
| `users.is_admin` | Administrator status survives deactivation (FR-IDN-019). |
| `tokens.secret_hash` | SHA-256 of a random secret of at least 256 bits. The secret is shown once and never stored. A secret of that entropy makes an unsalted, fast, indexed digest sufficient for lookup (decision B). |
| `tokens.revoked` | Set by explicit revocation (FR-AUTH-004) and by deactivation (FR-IDN-012). Expiry is computed against `expiresdt` (FR-AUTH-026), never stored. |
| `token_grants` | Project deletion cascades (FR-AUTH-020, FR-PRJ-009). `tokens` rows are never deleted, so the token cascade does not fire in normal operation. |
| `audit_log` | Same structure as the project table (FR-AUD-007), so `user_id` has no FK. Consequence entries (FR-AUD-014) use this mapping, a technical choice (B): removed role → `delete` `role`; revoked token → `modify` `token`; removed token permission → `modify` `token`, one entry per token, because grants have no entity of their own. Administrator status changes and (de)activation are `modify` `user`. Role changes are `create`/`modify`/`delete` `role` with `entityid` = `project_roles.id`. Profile creation at first login (FR-IDN-017) is `create` `user`, attributed to that user. |

## Project database `planner_<project>`

Tables are created in this order: referenced tables first.

```sql
CREATE DATABASE planner_<project> CHARACTER SET utf8mb4 COLLATE utf8mb4_nopad_bin;

CREATE TABLE schema_version (       -- identical to the central table
  id        TINYINT UNSIGNED NOT NULL PRIMARY KEY,
  version   INT UNSIGNED     NOT NULL,
  applieddt DATETIME(6)      NOT NULL,
  CONSTRAINT ck_schema_version_single CHECK (id = 1)
) ENGINE=InnoDB;

CREATE TABLE project_settings (
  id          TINYINT UNSIGNED NOT NULL PRIMARY KEY,
  description MEDIUMTEXT NULL,
  CONSTRAINT ck_project_settings_single CHECK (id = 1),
  CONSTRAINT ck_project_settings_description CHECK (description IS NULL OR description <> '')
) ENGINE=InnoDB;

CREATE TABLE sprints (
  id            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
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
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
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
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  kind        ENUM('epic','feature','userstory','task','defect','refactoring','spike','issue') NOT NULL,
  title       VARCHAR(200) NOT NULL,
  description MEDIUMTEXT NOT NULL,
  parent_id   BIGINT UNSIGNED NULL,
  parent_kind ENUM('epic','feature','userstory','task','defect','refactoring','spike','issue') NULL,
  planstart   DATE NULL,
  planend     DATE NULL,
  starteddt   DATETIME(6) NULL,
  closeddt    DATETIME(6) NULL,
  sprint_id   BIGINT UNSIGNED NULL,
  release_id  BIGINT UNSIGNED NULL,
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
  blocker_id   BIGINT UNSIGNED NOT NULL,
  blocker_kind ENUM('epic','feature','userstory','task','defect','refactoring','spike','issue')
               NOT NULL DEFAULT 'issue',
  blocked_id   BIGINT UNSIGNED NOT NULL,
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
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  name        VARCHAR(50) CHARACTER SET ascii COLLATE ascii_nopad_bin NOT NULL,
  value       VARCHAR(50) CHARACTER SET ascii COLLATE ascii_nopad_bin NOT NULL DEFAULT '',
  color       CHAR(7)     CHARACTER SET ascii COLLATE ascii_nopad_bin NULL,
  description MEDIUMTEXT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uq_tags_name_value (name, value),
  CONSTRAINT ck_tags_name  CHECK (name REGEXP '(?-i)^[a-z0-9_-]{1,50}$'),
  CONSTRAINT ck_tags_value CHECK (value = '' OR value REGEXP '(?-i)^[a-z0-9_-]{1,50}$'),
  CONSTRAINT ck_tags_color CHECK (color REGEXP '(?-i)^#[0-9A-F]{6}$'),
  CONSTRAINT ck_tags_description CHECK (description IS NULL OR description <> '')
) ENGINE=InnoDB;

CREATE TABLE work_item_tags (
  work_item_id BIGINT UNSIGNED NOT NULL,
  tag_id       BIGINT UNSIGNED NOT NULL,
  PRIMARY KEY (work_item_id, tag_id),
  KEY ix_work_item_tags_tag (tag_id),
  CONSTRAINT fk_work_item_tags_wi  FOREIGN KEY (work_item_id) REFERENCES work_items (id) ON DELETE CASCADE,
  CONSTRAINT fk_work_item_tags_tag FOREIGN KEY (tag_id)       REFERENCES tags (id)       ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE comments (
  id           BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  work_item_id BIGINT UNSIGNED NOT NULL,
  kind         ENUM('HYPOTHESIS','TEST','FINDING','DECISION','PROGRESS','UPDATE','NOTE','QUESTION',
                    'FOLLOW-UP','BLOCKER','HANDOFF','ASSUMPTION','RISK','REVIEW','DEBT') NOT NULL,
  text         MEDIUMTEXT NOT NULL,
  author_id    BIGINT UNSIGNED NOT NULL,
  createddt    DATETIME(6) NOT NULL,
  editeddt     DATETIME(6) NULL,
  PRIMARY KEY (id),
  KEY ix_comments_work_item (work_item_id),
  CONSTRAINT fk_comments_work_item FOREIGN KEY (work_item_id) REFERENCES work_items (id) ON DELETE CASCADE,
  CONSTRAINT ck_comments_text CHECK (text <> '')
) ENGINE=InnoDB;

CREATE TABLE boards (
  id   BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  name VARCHAR(200) COLLATE utf8mb4_uca1400_nopad_ai_ci NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uq_boards_name (name),
  CONSTRAINT ck_boards_name CHECK (name <> '')
) ENGINE=InnoDB;

CREATE TABLE board_columns (
  id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  board_id   BIGINT UNSIGNED NOT NULL,
  kind       ENUM('open','tag','closed') NOT NULL,
  tag_id     BIGINT UNSIGNED NULL,
  position   INT NULL,
  fixed_kind VARCHAR(6) AS (IF(kind <> 'tag', kind, NULL)) PERSISTENT,
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
  column_id    BIGINT UNSIGNED NOT NULL,
  work_item_id BIGINT UNSIGNED NOT NULL,
  position     BIGINT NOT NULL,
  PRIMARY KEY (column_id, work_item_id),
  KEY ix_board_cards_order (column_id, position),
  KEY ix_board_cards_work_item (work_item_id),
  CONSTRAINT fk_board_cards_column    FOREIGN KEY (column_id)    REFERENCES board_columns (id) ON DELETE CASCADE,
  CONSTRAINT fk_board_cards_work_item FOREIGN KEY (work_item_id) REFERENCES work_items (id)    ON DELETE CASCADE
) ENGINE=InnoDB;

CREATE TABLE audit_log (
  id        BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  createddt DATETIME(6) NOT NULL,
  user_id   BIGINT UNSIGNED NOT NULL,
  deviceid  VARCHAR(24) CHARACTER SET ascii COLLATE ascii_nopad_bin NOT NULL,
  action    ENUM('create','modify','delete') NOT NULL,
  entity    ENUM('workitem','comment','tag','board','sprint','release','project') NOT NULL,
  entityid  BIGINT UNSIGNED NOT NULL,
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
| `tags.color` | Stored in uppercase (FR-TAG-015). The application normalises it. The API therefore returns uppercase whatever the input case. |
| `comments.author_id` | Logical reference (D10). Set once by the application (FR-CMT-011). |
| `sprints.active_marker` | It is `1` only for the active Sprint and NULL otherwise. With the UNIQUE index this allows at most one active Sprint (BR-SPR-001, FR-SPR-008), because NULLs do not collide [S18]. Indexes on PERSISTENT generated columns are supported [S16]. `IF()` in a PERSISTENT generated column loads, and a second active Sprint is rejected with error 1062 [S22]. |
| `board_columns.fixed_kind` | With `uq_board_columns_fixed`, a board has at most one `open` column and one `closed` column (FR-KBN-002). The application creates both with the board, and no path deletes them. Tag columns are ordered by `position` (FR-KBN-023). `open` is always first and `closed` always last, by definition. `uq_board_columns_tag` enforces FR-KBN-020. The tag FK cascade enforces FR-KBN-017 and FR-TAG-010. |
| `board_cards` | Persisted manual card order (FR-KBN-015). `position` is not unique: InnoDB checks uniqueness per row, which would make swaps fail. Ties are ordered by `work_item_id`. Maintenance: [data-integrity-and-concurrency.md](data-integrity-and-concurrency.md#board-card-maintenance). |
| `project_settings` | One row (`id = 1`) holding the project `desc` (FR-PRJ-004, FR-PRJ-010; decision U). Its edits are audited as `modify` `project`, with `entityid` = central `projects.id` (FR-AUD-015). |
| `audit_log` | Append-only (FR-AUD-006, BR-AUD-001): see [Privileges](#privileges). `'GUI'` cannot match the 24-character pattern, so the two `deviceid` forms are disjoint (FR-AUTH-016, FR-AUTH-023). Card reorder is `modify` `board`. Tag (un)assignment and blocking-link changes are `modify` `workitem` for each Work Item affected (FR-AUD-014). |

ENUM columns used in a composite FK (`kind`/`parent_kind`, `kind`/`blocker_kind`) are declared identically. FK columns must be of the same or similar type, with the same character set and collation [S5]. ENUM columns qualify: the FKs load and are enforced (a non-issue blocker is rejected with error 1452) [S22].

## Privileges

| Account | Grants | Purpose |
|---------|--------|---------|
| Application (runtime) | Per table: `SELECT, INSERT, UPDATE, DELETE` on every table except `audit_log`. `SELECT, INSERT` on `audit_log`. `SELECT` on `schema_version`. Granted in `planner` and, by the provisioning step, in each `planner_<project>`. | FR-AUD-006 and BR-AUD-001 are also enforced by the server (decision B). |
| Provisioning | `CREATE, DROP` on `` `planner\_%`.* `` plus the privileges it grants, `WITH GRANT OPTION`. | Project creation and deletion (FR-PRJ-008, FR-PRJ-009). `CREATE`/`DROP` at database level permit `CREATE`/`DROP DATABASE`. A grantor needs `GRANT OPTION` and the privileges it grants [S13]. |

Consequences:

- A database-level grant applies to all tables of the database, including later ones [S13]. It cannot exclude `audit_log`. The application's project-database privileges are therefore table-level grants, issued per table at provisioning.
- Privileges on a database are not removed by `DROP DATABASE` [S13]. Table-level grants also remain in `mysql.tables_priv` after `DROP DATABASE` [S22]. Project deletion therefore revokes them explicitly.
- `GRANT` database names accept the `%` and `_` wildcards [S13]. `planner\_%` matches `planner_y` and does not match `plannerXa` [S22].
- The full privilege list for schema creation and migration is not specified here: see [TOQ-007](open-questions.md#toq-007).

## Schema versioning

- `schema_version` holds one row with the version number of the schema the database conforms to.
- Project creation writes the current version.
- A migration is applied to `planner` and to every project database whose `projects.state` is `active`. Each database records its new version on completion.
- DDL statements commit implicitly [S13]. A migration is therefore not atomic across its statements, and it is not atomic across databases.

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
| S14 | MariaDB, "InnoDB System Variables" (`innodb_snapshot_isolation`, `innodb_autoinc_lock_mode`): <https://mariadb.com/docs/server/server-usage/storage-engines/innodb/innodb-system-variables> |
| S15 | MariaDB, "SELECT" (locking clauses): <https://mariadb.com/docs/server/reference/sql-statements/data-manipulation/selecting-data/select> |
| S16 | MariaDB, "Generated Columns": <https://mariadb.com/docs/server/reference/sql-statements/data-definition/create/generated-columns> |
| S17 | MariaDB, "REGEXP": <https://mariadb.com/docs/server/reference/sql-functions/string-functions/regular-expressions-functions/regexp> |
| S18 | MariaDB, "UNIQUE" (InnoDB schema design): <https://mariadb.com/docs/storage-engines/innodb/schema-design/indexes/unique/> |
| S19 | MariaDB, "ENUM": <https://mariadb.com/docs/server/reference/data-types/string-data-types/enum> |
| S20 | MariaDB, "BOOLEAN": <https://mariadb.com/docs/server/reference/data-types/numeric-data-types/boolean> |
| S21 | OpenID Connect Core 1.0, §2 (`iss`, `sub`), §5.7: <https://openid.net/specs/openid-connect-core-1_0.html> |
| S22 | Local test on MariaDB 11.8.9, 2026-11-23, reported by the coordinator. Results: the error 1901 CHECK/FK restriction; ENUM composite FKs enforced; `IF()` generated column and `uq_sprints_one_active`; `DATETIME` vs `TIMESTAMP` under `time_zone`; `\_` in `GRANT`; table-level grants persist after `DROP DATABASE`; `utf8mb4_uca1400_nopad_ai_ci` equalities; auto-increment persists across restart (max id 4 deleted, restart, next id 5). Both DDL scripts in this file load without error. Also confirmed: the hierarchy CHECK rejects task→epic, `s1` is rejected against `S1`, and a second simple `frontend` tag is rejected. |
