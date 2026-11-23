# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**planner** — simple platform for software agile management, from planning to release, inspired by **Scrum** and **Kanban**.

## Functional specification

`docs/functional-spec/` is the **single source of truth** for what planner does: domain model, business rules, permissions, views, CLI behaviour and authentication. Start at `docs/functional-spec/README.md` (index, conventions, ID scheme).

- Read the relevant spec files before implementing or changing behaviour, and reference requirement IDs (`FR-`, `NFR-`, `BR-`, `UC-`) in code reviews, commits and tests where useful.
- Do not restate or redefine requirements here or in `README.md`; change them only in the spec (through the `specification-manager` agent).
- Never resolve an ambiguity by assumption: unresolved gaps are recorded in `docs/functional-spec/open-questions.md` and must be put to the user.

## Architecture

Client/server, both in **Rust**:

- **Server** — web application with a **WEB API** (REST) and a **WEBSITE** (GUI); persistence in **MariaDB** (central `planner` database plus one `planner_<project>` database per project).
- **Client** — the **CLI**, client of the WEB API.

**The CLI is COMPLETELY AI-agent-oriented.** Its entire architecture — from command structure to how the help is organised — must be designed and optimised to be operated by AI agents such as Claude Code. Treat AI agents as the primary user of the CLI in every design decision.
