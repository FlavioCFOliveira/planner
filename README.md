# planner
Simple platform for software agile management, from planning to release.

## Architecture

planner follows a **client/server architecture**, with both components developed in **Rust**:

- **Server** — a REST Web API that also serves a web interface.
- **Client** — a command-line interface (CLI) that talks to the server's REST API.
