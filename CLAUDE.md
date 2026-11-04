# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**planner** — Simple platform for software agile management, from planning to release.

Goal: manage the entire software development lifecycle, inspired by agile methodologies such as **Scrum** and **Kanban**.

## Domain

The central entity is the **Work Item (WI)**. Each WI has a **type** that distinguishes the nature of the work it addresses.

Work Item types: **Epic, Feature, User Story, Task, Defect, Refactoring, Spike, Issue**. All concepts below are WI types.

**Strategic planning types** (high level): Epic and Feature. They group large volumes of work and align development with business goals.

**Epic** — a large business initiative that cannot be delivered within a single Sprint; too big to estimate accurately, so it must be broken down into smaller parts. An Epic is composed of several Features.

**Feature** — a software component that delivers clear value to the end user and usually fits within a single Release. A Feature is composed of several User Stories.

**User Story** — a user need described from that user's point of view (who, what, why); the basic unit of value in agile development. A User Story is decomposed into multiple Tasks.

**Task** — the technical breakdown of a User Story, Defect, Refactoring or Spike: the actual work developers, testers or designers must do. Business rule: any item decomposed into Tasks (User Story, Defect, Refactoring, Spike) is done only when all of its Tasks are finished.

Hierarchy: Epic → Feature → User Story → Task.

**Defect** (error/bug) — a problem or failure that makes the software behave differently from what is expected; requires immediate or planned correction. A Defect can be decomposed into multiple Tasks.

**Refactoring** (technical debt) — internal work to improve code quality, performance or architecture without changing user-visible behaviour; keeps the system from becoming slow or hard to maintain. A Refactoring can be decomposed into multiple Tasks.

**Spike** (investigation) — research or prototyping work, used when the team does not know how to implement something and must investigate before it can estimate the real work. A Spike can be decomposed into multiple Tasks.

Hierarchy: Defect | Refactoring | Spike → Task.

**Issue** — a work item that blocks the progress of the team's work. It can block any item in the hierarchy (Epic, Feature, User Story, Defect, Refactoring, Spike, Task). It is not part of the hierarchy and is not decomposed into Tasks.

## Architecture

Client/server architecture, both components written in **Rust**:

- **Server** — REST Web API, which also serves a web interface.
- **Client** — CLI that consumes the server's REST API.
