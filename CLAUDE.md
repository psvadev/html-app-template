# [App Name] — Claude Code Briefing

## Stack — hard constraints, never change these
- Single `index.html`, no build step, no npm, no bundler
- React 18 + Babel Standalone via CDN (JSX compiled in-browser at runtime)
- All data stored locally via `localStorage`; Google Drive sync is opt-in

## Architecture
- `const PFX = 'xyz_'` at the top of the script — all `localStorage` keys use this prefix. Rename it once per app.
- `lsGet(key, fallback)` / `lsSet(key, value)` — every localStorage read/write goes through these helpers, never directly.
- `const VIEWS = [...]` — hash routing targets. Add a string here and a matching `{view === 'x' && <XView />}` in App's return.
- All state lives in `App`; views are pure display components that receive props.
- Persistence pattern: `useState(() => lsGet('key', default))` for lazy init + `useEffect(() => lsSet('key', val), [val])` for write-on-change.

## Design system
- Colors live in CSS custom properties on `:root` — never hardcode hex values in JSX inline styles or CSS rules.
- `[data-theme="light"]` overrides the dark-mode defaults; toggled by `document.documentElement.setAttribute('data-theme', theme)`.
- Button variants: `.btn-primary`, `.btn-secondary`, `.btn-ghost`, `.btn-danger`, `.btn-sm`, `.btn-icon`
- Layout primitives: `.card`, `.card-sm`, `.settings-section`, `.settings-row`
- Responsive: nav visible on desktop (>640px); tab bar replaces it on mobile (≤640px)

## Google Drive sync (if the optional block is enabled)
- PKCE OAuth2, scope `drive.file` only
- Three keys stored **without** the PFX prefix: `driveToken`, `driveFileId`, `drive_pkce_verifier`
- Auto-save fires 2 seconds after primary data state changes (debounce on `data`)
- `getValidAccessToken()` returns the string `'TOKEN_EXPIRED'` on `invalid_grant` — check for this before any Drive API call
- Backup filename derived from `APP_NAME`: `my-app-backup.json`

## What NOT to do
- No `npm install`, no webpack/vite/parcel/esbuild
- No external CSS frameworks (Tailwind, Bootstrap, Material UI)
- No external state libraries (Redux, Zustand, Jotai)
- No comments explaining *what* the code does — only add a comment when the *why* is non-obvious
- No hardcoded colors in JSX or CSS — use CSS vars
- No splitting into multiple files

## Plan before building
For any change larger than ~20 lines, describe the approach in one paragraph and get confirmation before writing code. Mention which existing functions will be reused or extended.
