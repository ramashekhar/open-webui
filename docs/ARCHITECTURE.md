# Architecture Knowledge Base

Incremental codebase knowledge cache. Read this first before exploring source files.
Each subsystem file in `docs/architecture/` contains key files, data flow, and gotchas.

If a subsystem is missing → explore source, then add a new file here and append an entry below.

## Subsystems

| Subsystem | File | Summary |
|-----------|------|---------|
| Follow-Up Prompts | [architecture/follow-up-prompts.md](architecture/follow-up-prompts.md) | Async LLM call after each response generates contextual follow-up chips |
| MCP Tools | [architecture/mcp-tools.md](architecture/mcp-tools.md) | MCP tool registry, spec fetching, assembly into chat completion requests |
| Notifications | [architecture/notifications.md](architecture/notifications.md) | Toast + native notification system; Socket.IO user rooms; external push options |
| Tool Call Display | [architecture/tool-call-display.md](architecture/tool-call-display.md) | Clickable chip showing tool payload+response; inline details panel; Default mode only |
| Custom Dialog (planned) | [architecture/custom-dialog.md](architecture/custom-dialog.md) | Trigger modal popup from backend event; ~30 lines frontend, 0 backend changes |
