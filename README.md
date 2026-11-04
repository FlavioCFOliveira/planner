# planner
Simple platform for software agile management, from planning to release.

## Goal

planner aims to manage the entire software development lifecycle — from planning to release — drawing inspiration from agile methodologies such as **Scrum** and **Kanban**.

## Core concept: Work Items

The central element of planner is the **Work Item (WI)**. Work Items can have different **types**, which distinguish the nature of the work each one is meant to address.

| Type | Summary |
|------|---------|
| Epic | Large business initiative spanning multiple Sprints |
| Feature | Software component delivering clear value to the end user |
| User Story | User need told from the user's point of view |
| Task | Technical breakdown of a work item into actual work |
| Defect | Software behaving differently from what is expected |
| Refactoring | Internal quality improvement (technical debt) |
| Spike | Research or prototyping before estimating |
| Issue | Something blocking the team's progress |

### Strategic planning types

**Epics** and **Features** are high-level strategic planning items. They group large volumes of work and align development with business goals.

#### Epic

An **Epic** is a large business initiative that cannot be delivered within a single iteration (Sprint). It is too big to be estimated accurately and must be broken down into smaller parts. An Epic is composed of several **Features**.

#### Feature

A **Feature** is a software component that delivers clear value to the end user and usually fits within a single release cycle (Release). A Feature is composed of several **User Stories**.

### User Story

A **User Story** describes a user need told from that user's point of view — **who** needs it, **what** they need, and **why**. It is the basic unit of value in agile development. A User Story is decomposed into multiple **Tasks**.

### Task

A **Task** is the technical breakdown of a User Story, Defect, Refactoring or Spike. It represents the actual work that developers, testers or designers need to do. A User Story, Defect, Refactoring or Spike is only done when all of its Tasks are finished.

### Defect

A **Defect** (error or bug) is a problem or failure in the software that causes it to behave differently from what is expected. It requires either immediate or planned correction. Like a User Story, a Defect can be decomposed into multiple **Tasks**.

### Refactoring

A **Refactoring** (technical debt) is internal work to improve code quality, performance or architecture without changing the behaviour visible to the user. It prevents the system from becoming slow or hard to maintain in the future. A Refactoring can be decomposed into multiple **Tasks**.

### Spike

A **Spike** (investigation) is a research-oriented piece of work, or the creation of a prototype. It is used when the team does not know how to implement something and needs to investigate before it can estimate the real work. A Spike can be decomposed into multiple **Tasks**.

### Issue

An **Issue** is a work item that blocks the progress of the team's work. An Issue can block any other work item in the hierarchy (Epic, Feature, User Story, Defect, Refactoring, Spike or Task). Unlike those items, an Issue is not decomposed into Tasks.

### Hierarchy

```
Epic
└── Feature
    └── User Story
        └── Task

Defect | Refactoring | Spike
└── Task

Issue ──blocks──▶ any work item above
```

## Architecture

planner follows a **client/server architecture**, with both components developed in **Rust**:

- **Server** — a REST Web API that also serves a web interface.
- **Client** — a command-line interface (CLI) that talks to the server's REST API.
