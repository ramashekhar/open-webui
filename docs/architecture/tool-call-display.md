# Tool Call Display (Payload/Response Modal)
_Last updated: 2026-05-27 | Approx commit: 03a5bec8d_

## What It Does
When Function Calling is set to "Default", each tool call executed during a chat response renders as a clickable chip showing the tool name. Clicking it expands an inline panel showing Parameters (JSON payload sent to tool) and Content (tool response). Only available in "Default" mode — not in "Native" mode.

## Key Files

- `src/lib/components/common/ToolCallDisplay.svelte` — main component; chip + collapsible Parameters/Content panel; decodes HTML-encoded `arguments` and `result` attributes; shows spinner/checkmark/wrench status icon
- `src/lib/components/chat/Messages/Markdown/MarkdownTokens.svelte` — detects `type="tool_calls"` token from marked.js, renders `ToolCallDisplay`
- `src/lib/components/chat/Messages/Markdown/ConsecutiveDetailsGroup.svelte` — groups multiple consecutive tool calls under a summary header
- `src/lib/components/chat/Settings/Advanced/AdvancedParams.svelte:165` — "Default" (`null`) vs "Native" (`'native'`) toggle

## Data Flow

```
Backend executes tool call
→ Injects <details> tag into message content HTML with encoded attrs
→ marked.js extension parses → token with attributes object
→ MarkdownTokens.svelte detects type="tool_calls" → renders ToolCallDisplay
→ User clicks chip → expands → shows Parameters + Content
```

## `<details>` Tag Shape (backend-injected)

```html
<details type="tool_calls"
  name="workday_get_career_profile"
  arguments="%7B%22employee_id%22%3A%2221103%22%7D"   <!-- URL-encoded JSON -->
  result="%7B%22Report_Entry%22%3A..."                  <!-- URL-encoded JSON -->
  done="true"
  id="call_abc123"
  files="[]"
  embeds="[]">
</details>
```

## ToolCallDisplay Attribute Parsing

```typescript
// ToolCallDisplay.svelte:79-87
$: args   = decode(attributes?.arguments ?? '');   // URL-decode
$: result = decode(attributes?.result ?? '');
$: parsedArgs   = parseJSONString(args);            // JSON.parse or null
$: parsedResult = parseJSONString(result);
```

Parameters section: renders `parsedArgs` as key-value pairs if object, else raw JSON code block.
Content section: renders `parsedResult` as JSON or plain text.

## "Default" vs "Native" Modes

| Setting | Value | Behavior |
|---------|-------|----------|
| Default | `null` | Backend intercepts tool calls, injects `<details>` tags, chip visible |
| Native  | `'native'` | Model handles tool calls natively, no `<details>` injection, no chip |

Setting path: `settings.params.function_calling` → sent in chat completion request body.

## Gotchas / Non-obvious
- Not a dialog/modal — inline `<details>` collapse, no overlay
- `arguments` and `result` are HTML/URL-encoded, not raw JSON — must decode before parse
- Chip shows spinner while `done="false"`, checkmark when `done="true"`
- Multiple tool calls in one response are grouped by `ConsecutiveDetailsGroup` with a single summary header
- Feature invisible in "Native" mode — tool calls handled by model, no backend intercept
