---
title: Technical open questions
status: draft
last-reviewed: 2026-11-23
related: [README.md, database-schema.md, domain-model.md, data-integrity-and-concurrency.md, ../functional-spec/open-questions.md]
---

# Technical open questions

## Overview

Nothing here is decided. When an item is decided, the decision goes into the affected file and the item is removed. Numbers are never reused: TOQ-002 to TOQ-006 and TOQ-008 were resolved on 2026-11-23 and removed.

## TOQ-001

**Topic**: User photo storage backend.
**Gap**: Photos are stored outside the database, which holds only `users.photo_key` (decision: user, 2026-11-23). The store (filesystem, object store, other) is not chosen. Its key format, access control, size limits and backup are therefore unspecified.
**Affects**: [database-schema.md](database-schema.md#central-database-planner) `users.photo_key`; FR-IDN-006.

## TOQ-007

**Topic**: Operational parameters not yet specified.
**Gap**: a) the retry bound for L4 ([data-integrity-and-concurrency.md](data-integrity-and-concurrency.md#locking-protocol)); b) the full privilege list of the provisioning and migration accounts; c) whether several server instances run concurrently. This determines how crash recovery of `provisioning`/`deleting` projects avoids interfering with a live sequence, and how migrations are serialised.
**Affects**: [database-schema.md](database-schema.md#privileges), [data-integrity-and-concurrency.md](data-integrity-and-concurrency.md#project-provisioning-and-deletion).
