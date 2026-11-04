# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**planner** — Simple platform for software agile management, from planning to release.

## Architecture

Client/server architecture, both components written in **Rust**:

- **Server** — REST Web API, which also serves a web interface.
- **Client** — CLI that consumes the server's REST API.
