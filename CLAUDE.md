# [App Name] — Claude Code Briefing

## Stack — hard constraints, never change these
- Single `index.html`, no build step, no npm, no bundler
- React 18 + Babel Standalone **7** via CDN (JSX compiled in-browser at runtime)
- CDN scripts must keep their major-version pins (`@18`, `@7`) — removing the Babel pin causes a silent upgrade to Babel 8, which breaks all JSX with a blank page
- **Blank page on load?** Three causes in order of likelihood: (1) Babel pin stripped → Babel 8 breaks JSX silently; (2) CDN blocked (corporate firewall, ad-blocker) — check DevTools → Network tab; (3) runtime JS error on first render — check DevTools → Console. The template's `#root` fallback ("Loading…") and `window.onerror` handler surface failures (1) and (3); cause (2) shows "Loading…" indefinitely.
- All data stored locally via `localStorage`; Google Drive sync is opt-in

## Architecture
- `const PFX = 'xyz_'` at the top of the script — all `localStorage` keys use this prefix. Rename it once per app.
- `lsGet(key, fallback)` / `lsSet(key, value)` — every localStorage read/write goes through these helpers, never directly. `lsSet` returns `false` on write failure (quota); the primary data effect surfaces that as a persistent error toast — never let a failed data write pass silently, and never make `lsSet` throw.
- `const VIEWS = [...]` — hash routing targets. Add a string here and a matching `{view === 'x' && <XView />}` in App's return.
- All state lives in `App`; views are pure display components that receive props.
- Persistence pattern: `useState(() => lsGet('key', default))` for lazy init + `useEffect(() => lsSet('key', val), [val])` for write-on-change.
- Any operation that replaces the whole dataset (backup restore, Drive conflict load) must call `snapshotData(current)` first — rolling last-3 under the `snapshots` key. Clear All Data is exempt: deletion is its intent, and its confirm promises permanence.

## Design system
- Colors live in CSS custom properties on `:root` — never hardcode hex values in JSX inline styles or CSS rules.
- `[data-theme="light"]` overrides the dark-mode defaults; toggled by `document.documentElement.setAttribute('data-theme', theme)`.
- Button variants: `.btn-primary`, `.btn-secondary`, `.btn-ghost`, `.btn-danger`, `.btn-sm`, `.btn-icon`
- Layout primitives: `.card`, `.card-sm`, `.settings-section`, `.settings-row`
- Responsive: nav visible on desktop (>640px); tab bar replaces it on mobile (≤640px)

## Google Drive sync (if the optional block is enabled)
- PKCE OAuth2, scope `drive.file` only
- `driveToken`, `driveFileId`, `driveModifiedTime` are stored via lsGet/lsSet, so **with** the PFX prefix — each app holds its own token and file association. Only `drive_pkce_verifier` is raw unprefixed `localStorage` (one-shot value around the OAuth redirect)
- Auto-save fires 2 seconds after primary data state changes (debounce on `data`)
- `saveToDrive` refuses to upload until `driveLoadConfirmed.current` is true — set only by a successful `loadFromDrive` read (the no-file first run counts). Never remove this gate: a failed load must block sync and show `driveStatus 'error'`, or the next local edit silently overwrites the Drive copy.
- Before overwriting an existing Drive file, `saveToDrive` fetches its `modifiedTime` and compares against the `driveModifiedTime` key (recorded at every successful load/save, via lsGet/lsSet). Drive newer → conflict prompt, never a silent overwrite. "Load Drive" resolves via `loadFromDrive({ skipConfirm: true })` through `loadFromDriveRef`.
- `getValidAccessToken()` returns the string `'TOKEN_EXPIRED'` on `invalid_grant` — check for this before any Drive API call
- `loadFromDrive` compares Drive data against local before overwriting — on conflict, `window.confirm` shows item counts for both sides and lets the user choose; picking "Cancel" pushes local → Drive immediately so both sides re-sync. Keep the counts in the message: an uninformed binary choice is how a near-empty local copy overwrites the full cloud copy.
- The Settings ⚠ badge in nav and tab bar triggers on both `driveStatus === 'expired'` and `driveStatus === 'error'` — remove both badge snippets if you remove the Drive block
- Backup filename derived from `APP_NAME`: `my-app-backup.json`

## Version footer (if the optional block is enabled)
- `VersionFooter` in Settings fetches the latest commit for `REPO_FILE` from GitHub's public API and shows short-SHA + date — set `REPO_OWNER`/`REPO_NAME` or it renders nothing
- It proves the commit exists on GitHub, not that the browser is running it — a stale cached page shows the newest SHA; hard refresh fixes
- Must fail silently (offline, rate limit) — never surface an error for a convenience footer

## What NOT to do
- No `npm install`, no webpack/vite/parcel/esbuild
- No external CSS frameworks (Tailwind, Bootstrap, Material UI)
- No external state libraries (Redux, Zustand, Jotai)
- No comments explaining *what* the code does — only add a comment when the *why* is non-obvious
- No `dangerouslySetInnerHTML` with user-entered text — JSX auto-escaping is the app's XSS protection, and it only holds while nobody bypasses it
- No hardcoded colors in JSX or CSS — use CSS vars
- No `toISOString().slice(0, 10)` for user-entry dates — it's UTC and mis-dates entries logged after local midnight; use `localISODate()`
- No splitting into multiple files

## Plan before building
For any change larger than ~20 lines, describe the approach in one paragraph and get confirmation before writing code. Mention which existing functions will be reused or extended.
