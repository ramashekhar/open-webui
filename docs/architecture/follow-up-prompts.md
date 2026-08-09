# Follow-Up Prompt Generation
_Last updated: 2026-05-27 | Approx commit: 03a5bec8d_

## What It Does
After each LLM response completes, fires a second async LLM call with chat history to generate 3–5 contextual follow-up question chips. Result returned via WebSocket and rendered as clickable buttons below the response. Currently has no awareness of available MCP tools — may suggest actions that have no tool backing.

## Key Files

**Backend**
- `backend/open_webui/config.py:1785` — `ENABLE_FOLLOW_UP_GENERATION` flag + `DEFAULT_FOLLOW_UP_GENERATION_PROMPT_TEMPLATE` (uses `{{MESSAGES:END:6}}`)
- `backend/open_webui/constants.py:101` — `TASKS.FOLLOW_UP_GENERATION` constant
- `backend/open_webui/utils/task.py:295` — `follow_up_generation_template(template, messages, user)` — injects chat history into prompt
- `backend/open_webui/routers/tasks.py:229` — `POST /api/tasks/follow_up/completions` — builds prompt, calls LLM, returns `{"follow_ups": [...]}`
- `backend/open_webui/utils/middleware.py:2897` — trigger point inside `chat_completion_handler()`; checks flag, calls endpoint, emits WebSocket, persists to DB

**Frontend**
- `src/lib/components/chat/Chat.svelte:2283` — sends `follow_up_generation: $settings?.autoFollowUps ?? true` in request body
- `src/lib/components/chat/Chat.svelte:465` — listens `chat:message:follow_ups` WebSocket event → sets `message.followUps`
- `src/lib/components/chat/Messages/ResponseMessage.svelte:1458` — renders `<FollowUps>` when `message.followUps.length > 0` and message done
- `src/lib/components/chat/Messages/ResponseMessage/FollowUps.svelte` — clickable button list UI
- `src/lib/components/chat/Settings/Interface.svelte:28` — user settings: `autoFollowUps`, `keepFollowUpPrompts`, `insertFollowUpPrompt`

## Data Flow

```
User query
→ Chat.svelte includes { follow_up_generation: true } in request
→ middleware.py: after response stream completes, checks FOLLOW_UP_GENERATION flag
→ POST /api/tasks/follow_up/completions (model + messages + chat_id)
→ follow_up_generation_template() builds prompt with {{MESSAGES:END:6}}
→ LLM returns JSON: { "follow_ups": ["Q1?", "Q2?", "Q3?"] }
→ middleware parses, emits WebSocket event 'chat:message:follow_ups'
→ Chat.svelte sets message.followUps = data.follow_ups
→ ResponseMessage.svelte renders <FollowUps> chips
→ User click → submit directly OR insert to input box
```

## Gotchas / Non-obvious
- Runs **after** stream completes, returned via **Socket.IO not HTTP** — not part of the main completion response
- `keepFollowUpPrompts: false` (default) — chips only shown on last message; set true to show on all
- `insertFollowUpPrompt: false` (default) — click submits directly; set true to insert into input
- Template variable `{{MESSAGES:END:6}}` = last 6 messages only, not full history
- LLM has **no knowledge of available MCP tools** — may suggest actions with no tool backing (known gap, fix planned)
- `follow_up_generation_template()` supports `{{PROMPT}}`, `{{MESSAGES}}`, `{{USER_*}}` variables (same system as other task templates)
