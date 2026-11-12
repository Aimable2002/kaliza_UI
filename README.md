# Kaliza Web

Supporting web workspace for Kaliza. WhatsApp is the primary user control interface; this application focuses on visual, document-heavy and administrative experiences that are difficult to handle in chat.

## Start here
- [Master System Architecture](docs/MASTER_SYSTEM_ARCHITECTURE.md) — shared product and platform architecture.
- [Frontend Developer Specification](docs/FRONTEND_DEVELOPER_SPEC.md) — screen, interaction, API, security and acceptance requirements.
- [Legacy blueprint pointer](docs/FRONTEND_BLUEPRINT.md) — retained path for repository history; see the developer specification for current detailed requirements.

## Project status
This is a greenfield blueprint. No existing Claude-built frontend or implemented feature is assumed. The UI must consume backend APIs and must not own agent orchestration, task authority, permissions, connector secrets or billing calculations.

## Product constraints
- WhatsApp remains the primary control channel.
- Keep the web application focused and relatively small.
- Business Growth is first; Career Growth is future scope.
- Both channels share the same backend conversations, tasks, approvals and files.
