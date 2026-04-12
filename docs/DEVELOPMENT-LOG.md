# Development Log

## 2026-04-12 — Career profile artifact panel customisations

### Preview Card pill — light-mode colour fix + hint text polish

Updated the "Preview Card" pill button rendered in place of HTML code blocks:
- Light mode: pill now uses `bg-blue-50 / border-blue-200 / text-blue-700` for clear readability (was white-on-white, invisible)
- Dark mode: unchanged (`bg-black / border-white/10 / text-white`)
- Hint text updated: "Click to view full details" (was "Click to view full career profile")
- Hint text size bumped to `text-sm`; `←` arrow replaced with `👈` emoji

| File | Change |
|---|---|
| `src/lib/components/chat/Messages/CodeBlock.svelte` | Pill colours, hint text copy, text size, arrow → emoji |

### HTML code blocks replaced globally with Preview Card

All `html` fenced code blocks in assistant responses are now replaced with a "Preview Card" pill button (+ hint text) instead of showing raw HTML. Clicking opens the artifact panel. Behaviour is global — applies to any HTML block, not just career profiles.

Supersedes the earlier auto-collapse approach. Old `collapsed` logic for `html`/`svg` langs removed from `MarkdownTokens.svelte`.

| File | Change |
|---|---|
| `src/lib/components/chat/Messages/CodeBlock.svelte` | Added `{:else if preview && lang === 'html'}` branch rendering pill + hint; imported `Eye` icon |
| `src/lib/components/chat/Messages/Markdown/MarkdownTokens.svelte` | Removed html/svg auto-collapse override; simplified `collapsed` prop to `$settings?.collapseCodeBlocks ?? false` |

### Artifact panel — auto-collapse HTML code blocks

~~HTML/SVG code blocks now auto-collapsed when `detectArtifacts` is enabled, so users don't see raw HTML in chat.~~

Superseded by the Preview Card approach above.

| File | Change |
|---|---|
| `src/lib/components/chat/Messages/Markdown/MarkdownTokens.svelte` | `collapsed` prop on `<CodeBlock>`: returns `true` when `lang` is `html`/`svg` and `detectArtifacts` is on, instead of always using `collapseCodeBlocks` setting |

### Artifact panel — auto-open when HTML block detected

Added a reactive statement to auto-open the artifact side panel whenever a completed message contains a ` ```html ` block, without requiring the user to click Preview.

| File | Change |
|---|---|
| `src/lib/components/chat/Messages/ContentRenderer.svelte` | Added `$: if (done && content && detectArtifacts && !mobile && chatId)` reactive — calls `showArtifacts.set(true)` + `showControls.set(true)` when content matches ` ```(html\|svg) ` |

**Status:** Auto-open reactive added but not yet confirmed working end-to-end — further debugging needed (see pending issues).

### Artifact panel — fixed width when artifacts open

Artifact panel now opens at a fixed percentage width regardless of the user's saved `localStorage.chatControlsSize`, so the profile card renders without needing to drag.

| File | Change |
|---|---|
| `src/lib/components/chat/ChatControls.svelte` | `openPane()`: checks `$showArtifacts` first, resizes to `Math.max(50, minSize)` before falling back to localStorage. Added `$: if ($showArtifacts && pane && paneReady)` reactive for the same resize. Both `minSize` calculations updated from `350px` → `550px` baseline. |

**Pending:** Width target is 60% — currently at 50%, needs one more bump + rebuild.

---

## 2026-04-11

### Branding — Renamed "Open WebUI" → "Workday Chat Partner"

Updated all user-facing UI and static files to reflect the new brand name.

**Files updated:**

| File | Change |
|---|---|
| `src/lib/constants.ts` | `APP_NAME` constant → `'Workday Chat Partner'` (feeds `WEBUI_NAME` store used app-wide) |
| `src/app.html` | `<title>` tag updated |
| `static/static/site.webmanifest` | `name` and `short_name` (`"WD Chat"`) updated |
| `static/opensearch.xml` | `ShortName` and `Description` updated |
| `src/lib/components/channel/Channel.svelte` | Page `<title>` suffix `• Open WebUI` updated |
| `src/routes/+layout.svelte` | Browser notification title suffix updated |
| `src/lib/components/chat/SettingsModal.svelte` | About-page search keywords updated |
| `src/lib/i18n/locales/en-US/translation.json` | Display strings containing "Open WebUI" updated to new brand name |
| `backend/open_webui/env.py` | Removed upstream attribution line that appended ` (Open WebUI)` to any custom `WEBUI_NAME` |

### adchat.service — Admin instance migrated to source build

Migrated `open-webui.service` (uvx, port 8080) to a new source-built `adchat.service` (port 8084), matching the wdchat migration pattern.

**Changes:**
- Created `/home/admin/projects/wdchat/configs/services/adchat.service` — no `BindsTo` auth dependency, no trusted header, `WEBUI_NAME=Workday Chat Partner`, port 8084
- Updated `/home/admin/projects/wdchat/configs/nginx/adchat.aipoc.dev` — `proxy_pass` changed from port 8080 → 8084
- Deployed service and nginx config; old `open-webui.service` (uvx) still running pending final verification

**Verified:** Both old uvx services confirmed `inactive (dead)` and `disabled` — already retired. No action needed.

### Favicon / Icons — Custom logo applied across all variants

Replaced all default Open WebUI icons with custom logo (`tmp/screenshots/logo.png`, 582x502px source). Used ImageMagick to generate all required size variants. Backups saved to `tmp/favicon-backup/`.

**Files updated (both `static/static/` and `backend/open_webui/static/`):**

| File | Dimensions | Notes |
|---|---|---|
| `favicon.png` | 500x500 | Primary browser tab icon |
| `favicon-96x96.png` | 96x96 | Browser fallback |
| `favicon.ico` | 48/32/16 multi-res | Legacy browser support |
| `favicon-dark.png` | 500x500 | Inverted logo for dark theme backgrounds |
| `apple-touch-icon.png` | 180x180 | iOS home screen icon |
| `logo.png` | 500x500 | General logo, used in PWA manifest |
| `splash.png` | 500x500 | Light theme splash/loading screen |
| `splash-dark.png` | 500x500 | Dark theme splash — inverted logo |
| `web-app-manifest-192x192.png` | 192x192 | PWA install icon (medium) |
| `web-app-manifest-512x512.png` | 512x512 | PWA install icon (large) |

Frontend rebuilt (`npm run build`) after icon updates. Services restarted to pick up changes.

### Branding — Dark mode model icons + sidebar logo fixed

Resolved three dark-mode icon issues identified via screenshots:

1. **Sidebar brand logo (top-left "W")** — was hardcoded to `favicon.png` in both the collapsed and expanded sidebar states; now serves `favicon-dark.png` in dark mode using Tailwind `dark:hidden` / `hidden dark:block` on paired `<img>` tags.
2. **Chat message model icon (ResponseMessage)** — was using a raw API URL without the `&theme=` param, so always returned the light logo. Now uses the shared `getModelImageUrl()` helper which reads `document.documentElement.classList.contains('dark')` at render time to inject the correct theme.
3. **Workspace model list icons (MCP Workday models)** — these were showing the old base64-encoded blue logo stored in the model's `profile_image_url` DB field. The backend `/models/model/profile/image` endpoint now falls back to `favicon-dark.png` when `?theme=dark` is passed and no custom image is set, so once component-level URLs are updated the DB-stored value is bypassed.

**Files modified:**

| File | Change |
|---|---|
| `src/lib/components/layout/Sidebar.svelte` | Lines ~716 and ~914: replaced single hardcoded `favicon.png` `<img>` with paired `<img>` tags — light (`dark:hidden`) and dark (`hidden dark:block`) — in both the collapsed button and the expanded header logo positions |
| `src/lib/components/chat/Messages/ResponseMessage.svelte` | Line ~39: imported `getModelImageUrl` from `$lib/constants`. Line ~633: replaced inline URL template string with `getModelImageUrl(WEBUI_API_BASE_URL, model?.id ?? '', $i18n.language)` so `&theme=dark` is appended in dark mode |

**Build:** Frontend rebuilt (`npm run build`) after changes. Services pending restart (`sudo systemctl restart wdchat.service adchat.service`).

**Verified working (SSO):** Workday SSO flow confirmed end-to-end on `wdchat.aipoc.dev` — login, callback, session, OWU auto-login all functional.
