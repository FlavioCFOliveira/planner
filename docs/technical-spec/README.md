---
title: planner technical specification
status: draft
last-reviewed: 2026-11-23
related: [../functional-spec/README.md]
---

# planner technical specification

## Overview

This folder records **how** planner is built. Every decision traces to a requirement of the [functional specification](../functional-spec/README.md), which is the source of truth for behaviour, or to an explicit constraint or decision named here. Requirements are cited by ID (`FR-`, `NFR-`, `BR-`, `UC-`) and never restated.

The scope covers architecture and stack, the domain model, the database schema, integrity and concurrency, security, the WEB API conventions and resource map, the CLI, operations and testing. Field-level schemas of API bodies are not specified beyond the functional field names ([web-api.md](web-api.md)).

## Index

| File | Purpose |
|------|---------|
| [architecture.md](architecture.md) | Topology (single instance), technology stack, components, database access and pool, startup sequence, photo storage. |
| [domain-model.md](domain-model.md) | Rust domain types: identifiers, enums, value objects, entities, invariant placement; entity-relationship diagrams. |
| [database-schema.md](database-schema.md) | MariaDB conventions, server settings, DDL of the central and project databases, privileges, schema versioning, source register. |
| [data-integrity-and-concurrency.md](data-integrity-and-concurrency.md) | Where each cross-row rule is enforced, the locking protocol, multi-step operations, project provisioning and crash recovery. |
| [security.md](security.md) | OIDC login, first administrator, sessions, CSRF, CLI tokens, authorisation, rate limiting, reverse proxy. |
| [web-api.md](web-api.md) | REST conventions, representation, pagination, problem details and codes, resource map, index coverage. |
| [cli.md](cli.md) | Command structure, help and `planner schema`, output, exit codes, credentials. |
| [operations.md](operations.md) | Packaging, deployment, configuration, observability, backup and restore. |
| [testing.md](testing.md) | Unit, integration, end-to-end and performance tests. |
| [open-questions.md](open-questions.md) | Unresolved technical questions. |

Start with [architecture.md](architecture.md), then [domain-model.md](domain-model.md).

## Conventions

- **Frontmatter.** Each file carries `title`, `status` (`draft` | `approved` | `deprecated`), `last-reviewed` and `related`, as in the functional specification.
- **Requirement language.** RFC 2119 / RFC 8174 key words, in capitals only.
- **Decisions.** Each subject file states the decisions in force for its subject, with rationale and decider. There is no separate decision log. Decider key: **U** = the user, 2026-11-23. **B** = a Claude proposal accepted by the user on 2026-11-23 (including defaults proposed with a user decision).
- **Open questions.** Technical open questions are numbered `TOQ-NNN` in [open-questions.md](open-questions.md). Numbers are never reused. Functional open questions (`OQ-NNN`) are kept in [../functional-spec/open-questions.md](../functional-spec/open-questions.md).
- **Sources.** Each claim about a technology cites an entry in the [source register](database-schema.md#sources) as `[Sn]`. A claim confirmed by local test cites its local-test entry (`[S22]`, `[S48]`, `[S58]`). A claim confirmed by neither an official source nor a test is marked **unverified** and listed in [open-questions.md](open-questions.md).
- **Naming.** Functional field names (for example `desc`) are API names. Where a database column differs, the mapping is stated in [domain-model.md](domain-model.md).

## Status

All files are `draft`.
