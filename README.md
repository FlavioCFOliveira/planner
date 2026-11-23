# planner
Simple platform for software agile management, from planning to release.

## Goal

planner manages the entire software development lifecycle — from planning to release — drawing inspiration from agile methodologies such as **Scrum** and **Kanban**.

## At a glance

- **Work Items** are the central entity: Epics, Features, User Stories, Tasks, Defects, Refactorings, Spikes and Issues, organised in a hierarchy with closing rules and Issue blocking.
- **Sprints** and **Releases** plan the work over time and delivery.
- **Tags** (simple and scoped, GitLab-style) classify Work Items and drive **Kanban boards**.
- **Comments** form a typed work log per Work Item.
- **Views**: Work Item list, Kanban boards and Gantt chart.
- **Audit log** of every state-changing operation, per project and platform-wide.
- **Multi-project**, with users authenticated through an external OIDC Identity Provider and fixed per-project roles (`owner`, `member`, `viewer`).

## Architecture

Client/server, both written in **Rust**:

- **Server** — a web application exposing a **WEB API** (REST) and a **WEBSITE** (GUI), persisting data in **MariaDB**.
- **Client** — a command-line interface (**CLI**) for the WEB API, designed first and foremost to be operated by **AI agents**.

## Documentation

- [Functional specification](docs/functional-spec/README.md) — the single source of truth for requirements, business rules, actors, use cases and vocabulary.

## License

See [LICENSE](LICENSE).
