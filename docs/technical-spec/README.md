---
title: planner technical specification
status: draft
last-reviewed: 2026-11-23
related: [../functional-spec/README.md]
---

# planner technical specification

## Overview

This folder records **how** planner is built. Every decision traces to a requirement of the [functional specification](../functional-spec/README.md), which is the source of truth for behaviour, or to an explicit constraint or decision named here. Requirements are cited by ID (`FR-`, `NFR-`, `BR-`, `UC-`) and never restated.

The current scope is the domain model and the database schema. Topics not covered here (API contract, deployment, security beyond storage) have not been specified.

## Index

| File | Purpose |
|------|---------|
| [domain-model.md](domain-model.md) | Rust domain types: identifiers, enums, value objects, entities, invariant placement; entity-relationship diagrams. |
| [database-schema.md](database-schema.md) | MariaDB conventions, server settings, DDL of the central and project databases, privileges, schema versioning, source register. |
| [data-integrity-and-concurrency.md](data-integrity-and-concurrency.md) | Where each cross-row rule is enforced, the locking protocol, multi-step operations, project provisioning and crash recovery. |
| [open-questions.md](open-questions.md) | Unresolved technical questions. |

Start with [domain-model.md](domain-model.md).

## Conventions

- **Frontmatter.** Each file carries `title`, `status` (`draft` | `approved` | `deprecated`), `last-reviewed` and `related`, as in the functional specification.
- **Requirement language.** RFC 2119 / RFC 8174 key words, in capitals only.
- **Decisions.** Each subject file states the decisions in force for its subject, with rationale and decider. There is no separate decision log.
- **Open questions.** Technical open questions are numbered `TOQ-NNN` in [open-questions.md](open-questions.md). Numbers are never reused. Functional open questions (`OQ-NNN`) are kept in [../functional-spec/open-questions.md](../functional-spec/open-questions.md).
- **Sources.** Each claim about a technology cites an entry in the [source register](database-schema.md#sources) as `[Sn]`. A claim confirmed by local test is cited as `[S22]`. A claim confirmed by neither an official source nor a test is marked **unverified** and listed in [open-questions.md](open-questions.md).
- **Naming.** Functional field names (for example `desc`) are API names. Where a database column differs, the mapping is stated in [domain-model.md](domain-model.md).

## Status

All files are `draft`.
