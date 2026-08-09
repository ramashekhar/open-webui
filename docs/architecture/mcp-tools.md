# MCP Tools — Registry & Chat Integration
_Last updated: 2026-05-27 | Approx commit: 03a5bec8d_

## What It Does
MCP (Model Context Protocol) tools are external capability providers. On every chat completion request, middleware connects to configured MCP servers, fetches their tool specs, wraps them in callables, and merges them with local/builtin tools. For native function calling, the full tool list is packed into `form_data['tools']` as an OpenAI-style array. Tool results are intercepted and routed back by the middleware.

## Key Files

- `backend/open_webui/utils/mcp/client.py:65` — `MCPClient.list_tool_specs()` — fetches tool specs from MCP server via MCP protocol; returns `[{name, description, parameters}]`
- `backend/open_webui/utils/mcp/client.py:86` — `MCPClient.call_tool()` — executes a tool call
- `backend/open_webui/utils/middleware.py:2459` — main tool assembly block; resolves `tool_ids`, creates MCP clients, fetches specs, builds `tools_dict`
- `backend/open_webui/utils/middleware.py:2475` — MCP tool detection: checks if `tool_id.startswith('server:mcp:')`
- `backend/open_webui/utils/middleware.py:2538` — creates `MCPClient`, connects to server URL
- `backend/open_webui/utils/middleware.py:2551` — calls `list_tool_specs()`, wraps each in closure
- `backend/open_webui/utils/middleware.py:2570` — stores in `mcp_tools_dict` with prefixed name
- `backend/open_webui/utils/middleware.py:2604` — merges: `tools_dict = {**tools_dict, **mcp_tools_dict}`
- `backend/open_webui/utils/middleware.py:2681` — packs into `form_data['tools']` as OpenAI array (native FC only)
- `backend/open_webui/utils/tools.py` — `get_tools()`, `get_builtin_tools()`, `convert_openapi_to_tool_payload()`
- `backend/open_webui/routers/tools.py:64` — `GET /api/tools/` — lists all tools available to a user (local DB + OpenAPI + MCP)
- `backend/open_webui/models/tools.py` — `ToolModel`, `Tool` ORM models

## Data Structures

**Tool spec (OpenAI-style):**
```python
{
    'name': str,           # prefixed: '{server_id}_{tool_name}' for MCP
    'description': str,
    'parameters': {
        'type': 'object',
        'properties': { ... },
        'required': [...]
    }
}
```

**tools_dict entry (internal):**
```python
{
    'spec': dict,          # spec above
    'callable': async fn,  # wrapped MCP call
    'type': 'mcp',         # or 'builtin' / 'external'
    'client': MCPClient,
    'direct': bool
}
```

**form_data['tools'] (sent to LLM):**
```python
[{'type': 'function', 'function': spec_dict}, ...]
```

## Data Flow

```
Client sends POST /chat/completions with tool_ids=['server:mcp:...']
→ middleware detects MCP tool IDs
→ creates MCPClient per server, connects
→ list_tool_specs() enumerates available tools
→ each tool wrapped in async callable closure
→ merged into tools_dict with local/builtin tools
→ native FC: packed into form_data['tools'] as OpenAI array
→ LLM response contains tool_calls
→ middleware intercepts, routes to callable in tools_dict
→ result injected back into message stream
```

## Gotchas / Non-obvious
- MCP tool names **prefixed** `{server_id}_{tool_name}` to avoid name collisions across servers
- Tool specs fetched **fresh per request** — no caching; adding tools to existing MCP server picked up immediately, no OWUI restart
- Adding a **new MCP server connection** requires saving it in OWUI Admin settings (no service restart needed)
- `metadata['tools']` holds the full `tools_dict` (for execution); `form_data['tools']` holds only specs (for LLM)
- Follow-up prompt generation does **not** receive tool context — reads from `metadata['tools']` but currently not passed to template (fix planned)
- Tool execution intercept is at `middleware.py:1279`
