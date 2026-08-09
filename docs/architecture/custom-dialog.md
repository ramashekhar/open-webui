# Custom Dialog / Popup from Backend
_Last updated: 2026-05-27 | Approx commit: 03a5bec8d_

## What It Does (Planned)
Trigger a custom modal/dialog in the chat UI from a backend event — e.g. in response to a user query, show rich content (markdown/HTML) in an overlay popup. Uses existing Socket.IO event pipeline. Zero backend infrastructure changes needed.

## Why It's Low-Risk
- `Chat.svelte:507` already switches on `event.data.type` — adding `'dialog'` is one new case
- NotificationToast already renders markdown via marked+DOMPurify — reusable
- OWUI modal pattern: writable store + `{#if $store}` conditional render — well-established
- Backend emits via existing `sio.emit('events', ...)` — no new routes

## Implementation Sketch (~30 lines frontend, 0 backend changes)

### Frontend changes
1. Add `dialogContent` store in `src/lib/stores/index.ts`:
   ```ts
   export const dialogContent: Writable<{title: string, content: string} | null> = writable(null);
   ```
2. Add case in `Chat.svelte` chatEventHandler (line ~507):
   ```ts
   } else if (type === 'dialog') {
       dialogContent.set(data);
   }
   ```
3. Add modal render in `ResponseMessage.svelte` or `+layout.svelte`:
   ```svelte
   {#if $dialogContent}
     <Modal bind:show={$dialogContent}>
       <div>{@html marked($dialogContent.content)}</div>
     </Modal>
   {/if}
   ```

### Backend emit (no new code — reuse existing socket)
```python
await sio.emit('events', {
    'chat_id': chat_id,
    'message_id': message_id,
    'data': {
        'type': 'dialog',
        'data': { 'title': 'My Title', 'content': '**markdown** content here' }
    }
}, room=f'user:{user_id}')
```

## Alternative (Even Simpler)
Reuse `NotificationToast` with larger content — already does markdown+DOMPurify. No new modal component needed. Trade-off: toast auto-dismisses, dialog stays until closed.

## Key Files to Touch

| File | Change |
|------|--------|
| `src/lib/stores/index.ts` | Add `dialogContent` writable store |
| `src/lib/components/chat/Chat.svelte:507` | Add `type === 'dialog'` case |
| `src/routes/+layout.svelte` or ResponseMessage | Add `{#if $dialogContent}` modal render |

## Gotchas / Non-obvious
- `confirmation` event type already exists in chatEventHandler — check its implementation before adding `dialog` to avoid duplication
- Content must be sanitized via DOMPurify before `{@html}` render — already done in NotificationToast pattern
- Dialog triggered mid-stream (before response done) is fine — socket event is independent of response stream
