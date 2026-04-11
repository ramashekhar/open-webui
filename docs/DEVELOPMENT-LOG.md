# Development Log

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
