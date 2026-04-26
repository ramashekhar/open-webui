# Backlog

## Outstanding Items

### Features
- ~~Hide the raw HTML block generated for career profile artifact — users don't need to see it; add a compact "Preview" link/button that opens the rendered card (same behaviour as the existing Open WebUI HTML block preview), replacing the raw block entirely~~

### Issues
- [ ] **Branding: remove "Open WebUI" references** — UI still shows "Open WebUI" branding (e.g. version string below chat UI). All visible references should be replaced or hidden.
- [ ] **Login screen flash** — On login, the Open WebUI version/branding briefly flashes and disappears. Investigate and suppress.
- [ ] **Typeahead "no match" state** — When a user starts typing a question and there is no match in the auto-available follow-up questions list, the typeahead widget shows an undesired empty/no-match state. Should hide gracefully when there are no suggestions.
- [ ] **Hide LLM controller panel** — The panel where users can set temperature and other LLM parameters should be hidden from end users; they don't need to control these settings.
- [ ] **Revert HTML sidebar auto-open** — Auto-open of the artifact/preview panel on HTML responses is inconsistent (sometimes works, sometimes doesn't). If the panel was previously open and a new query returns HTML, the auto-open fires but fails to refresh the content, leaving a blank panel. Revert or comment out the auto-open trigger; let the user click "Preview" manually to open it.
