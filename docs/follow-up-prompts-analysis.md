# Follow-Up Prompt Generation — Code Analysis

## What It Does

After each LLM response completes, Open WebUI asynchronously calls a second LLM request to generate 3–5 contextually relevant follow-up questions. These appear as clickable chips below the response. Clicking one either inserts it into the input box or submits it directly.

## Data Flow

```
User query → Chat.svelte sends { follow_up_generation: true }
→ middleware.py triggers after LLM response completes
→ calls POST /api/tasks/follow_up/completions with chat history
→ LLM returns JSON { "follow_ups": ["Q1?", "Q2?", ...] }
→ WebSocket emits 'chat:message:follow_ups' to client
→ Chat.svelte updates message.followUps array
→ ResponseMessage.svelte renders FollowUps component
→ User clicks → submit or insert to input
```

Generation runs **after** main response, async via middleware, communicated back via Socket.IO (not HTTP).

---

## Backend

### Config
`backend/open_webui/config.py:1785-1811`
- `ENABLE_FOLLOW_UP_GENERATION` — server-wide feature flag
- `DEFAULT_FOLLOW_UP_GENERATION_PROMPT_TEMPLATE` — instructs LLM to return `{"follow_ups": ["Q1?", ...]}`
- `FOLLOW_UP_GENERATION_PROMPT_TEMPLATE` — customizable override

### Template Builder
`backend/open_webui/utils/task.py:295-301`
- `follow_up_generation_template()` — injects chat history into prompt via `replace_messages_variable()`

### API Endpoint
`backend/open_webui/routers/tasks.py:229-294`
- `POST /api/tasks/follow_up/completions`
- Builds prompt → calls configured LLM → parses JSON response

### Trigger Point
`backend/open_webui/utils/middleware.py:2897-2944`
- Inside `chat_completion_handler()`, fires after response stream completes
- Checks `FOLLOW_UP_GENERATION` task flag → calls endpoint → parses `follow_ups` array
- Emits WebSocket event `chat:message:follow_ups`
- Persists `followUps` field to DB via `Chats.upsert_message_to_chat_by_id_and_message_id()`

### Task Constant
`backend/open_webui/constants.py:101`
- `FOLLOW_UP_GENERATION = 'follow_up_generation'`

---

## Frontend

### Task Flag (per request)
`src/lib/components/chat/Chat.svelte:2283`
- `follow_up_generation: $settings?.autoFollowUps ?? true` — sent in request body each query

### WebSocket Listener
`src/lib/components/chat/Chat.svelte:465-470`
- Listens for `chat:message:follow_ups` → sets `message.followUps = data.follow_ups`

### Render Gate
`src/lib/components/chat/Messages/ResponseMessage.svelte:1458-1471`
- Shows `<FollowUps>` only when `message.followUps.length > 0`, message is done, not readOnly
- `keepFollowUpPrompts` setting controls whether all messages show chips or just the last

### UI Component
`src/lib/components/chat/Messages/ResponseMessage/FollowUps.svelte`
- Clickable button list
- Click fires `onClick(followUp)` → submit directly or insert to input box

### User Settings
`src/lib/components/chat/Settings/Interface.svelte:28,57-58`
- `autoFollowUps` (default `true`) — enable/disable generation
- `keepFollowUpPrompts` (default `false`) — show on all messages vs last only
- `insertFollowUpPrompt` (default `false`) — insert to input vs auto-submit

---

## How to Disable

**Server-wide:** Set env var `ENABLE_FOLLOW_UP_GENERATION=false` — backend skips task regardless of client flag.

**Per-user:** Settings → Interface → turn off "Auto Follow-Ups" → sets `autoFollowUps: false` → request sends `follow_up_generation: false`.
