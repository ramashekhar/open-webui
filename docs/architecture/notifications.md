# Notification System
_Last updated: 2026-05-27 | Approx commit: 03a5bec8d_

## What It Does
Two-tier notification system: toast UI (svelte-sonner) + browser native Notification API. Backend pushes events to frontend via Socket.IO user rooms. Frontend already handles `type=notification` events and fires toasts. No standalone external ingestion endpoint exists yet — planned addition.

## Key Files

**Frontend**
- `src/lib/components/NotificationToast.svelte` — custom toast component; markdown content via marked+DOMPurify; plays `/audio/notification.mp3`; drag-to-dismiss
- `src/routes/+layout.svelte:1071` — `<Toaster>` container (svelte-sonner), top-right
- `src/routes/+layout.svelte:878` — socket listeners: `events` → `chatEventHandler`, `events:channel` → `channelEventHandler`
- `src/lib/components/chat/Chat.svelte:507-519` — handles `type === 'notification'` → `toast.success/error/warning/info(data.content)`
- `src/lib/stores/index.ts:117-118` — `isLastActiveTab`, `playingNotificationSound` stores

**Backend**
- `backend/open_webui/socket/main.py:283` — `emit_to_users(event, data, user_ids)` — broadcast to specific users across all sessions
- `backend/open_webui/socket/main.py:299` — `enter_room_for_users(room, user_ids)`
- `backend/open_webui/socket/main.py:339` — `SESSION_POOL[sid]` — maps socket session to user data
- `backend/open_webui/socket/main.py:789` — emits `events` to `room=f'user:{user_id}'`
- `backend/open_webui/routers/chats.py:969` — `POST /{id}/messages/{message_id}/event` — existing but requires chat ownership + message ID

## Socket.IO Room Structure

```
user:{user_id}        → personal events (all tabs/devices of a user)
channel:{channel_id}  → shared channel messages
doc_{document_id}     → Yjs collaborative docs
note:{note_id}        → collaborative notes
```

User auto-joins `user:{user_id}` room on connect via `user-join` event.

## Data Flow (existing — chat/channel notifications)

```
Backend event → sio.emit('events', {chat_id, message_id, data:{type, data:{...}}}, room=f'user:{user_id}')
→ Frontend: Chat.svelte chatEventHandler receives 'events'
→ type === 'notification' → toast.success/error/warning/info(data.content)
→ type === 'chat:completion' → NotificationToast component (with sound)
Channel messages → sio.emit('events:channel') → layout.svelte channelEventHandler → native Notification API + toast
```

## Event Payload Shape (notification type)

```json
{
  "chat_id": null,
  "message_id": null,
  "data": {
    "type": "notification",
    "data": {
      "type": "success|error|warning|info",
      "content": "message text"
    }
  }
}
```

## Gotchas / Non-obvious
- `Chat.svelte` `type=notification` handler already exists — **zero frontend changes needed** to support external push
- Toast sound only plays if `navigator.userActivation.hasBeenActive` (user must have interacted with page first)
- `isLastActiveTab` prevents duplicate sounds across multiple open tabs
- Existing `POST /chats/{id}/messages/{message_id}/event` requires specific chat_id + message_id — not usable for standalone push
- No external ingestion endpoint yet — planned: new `POST /api/notifications/push` router (~40 lines, admin auth)
- Option B: session DB polling also viable since all services on same VPS — external app writes to notifications table, OWUI polls and emits
