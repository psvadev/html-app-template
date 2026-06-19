# Patterns

Non-obvious decisions made in this template, with the *why*. Read this before starting a new app or when wondering why something is done a particular way.

---

## Stack

**Single HTML file, no build step**
Zero dev-environment friction. Any device with a browser can open, edit, and run the file. No Node, no terminal, no configuration. The source file is the deployed artifact.

**React 18 + Babel Standalone via CDN**
JSX without a build step. Babel compiles JSX in-browser at load time — acceptable startup cost for a personal app. The CDN URLs pin to major versions (`@18`, `@7`), not patch versions, so they receive security fixes automatically without breaking changes.

**Always pin the major version on CDN deps — `@7`, not the bare package name.**
Unpinned CDN URLs silently upgrade to new major versions. When Babel 8 shipped, it became the default on unpkg and broke all `type="text/babel"` scripts with a confusing error:

```
Uncaught SyntaxError: import declarations may only appear at top level of a module
```

The error originates inside `babel.min.js`, not your code, which makes it hard to identify as a dependency issue. The page is blank, `Ctrl+Shift+R` doesn't help (it's not a cache issue), and nothing in the app's own code looks wrong.

**Fix:** ensure the Babel script tag includes the major version pin:
```html
<!-- BAD — upgrades silently to Babel 8 -->
<script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>

<!-- GOOD — stays on 7.x, gets patch fixes, no breaking changes -->
<script src="https://unpkg.com/@babel/standalone@7/babel.min.js"></script>
```

**When to revisit:** Babel 7.x is actively maintained. Migrate to `@8` only when (a) a specific Babel 8 feature is needed, or (b) 7.x approaches end-of-life. Check the Babel 8 migration guide before doing so.

**`#root` fallback + `window.onerror` for blank-page diagnosis**
The template puts a "Loading…" message inside `<div id="root">` and registers a `window.onerror` handler *before* the Babel script tag. React replaces the fallback content immediately on a successful render. On failure the fallback remains visible. Three blank-page causes in order of likelihood:

1. **Babel pin stripped** → Babel 8 breaks JSX silently; "Loading…" persists, console shows the `import declarations` error
2. **CDN blocked** (corporate firewall, ad-blocker) → "Loading…" persists, Network tab shows failed requests
3. **Runtime error on first render** → `window.onerror` updates the fallback to "⚠ Script error — open DevTools (F12)"

Without the fallback all three look identical: a blank page.

---

## Storage

**`const PFX = 'app_'` prefix on all keys**
Multiple apps can be open in the same browser origin (e.g. `file://` or `localhost`) without their `localStorage` keys colliding. Renaming the prefix is the one edit that scopes an app.

**`lsGet` / `lsSet` helpers**
Every localStorage access goes through these two functions. They centralise the JSON parse/stringify, silently swallow `QuotaExceededError` on writes, and return the fallback on any parse error. Direct `localStorage.getItem` calls bypass the prefix and error handling.

**Lazy `useState` init + paired `useEffect` persistence**
```js
const [items, setItems] = useState(() => lsGet('items', []));
useEffect(() => lsSet('items', items), [items]);
```
The lazy init reads from storage once on mount. The effect writes only when state actually changes — not on every render. This is the correct pattern; do not inline `lsSet` inside event handlers.

---

## Routing

**Hash routing (`window.location.hash`)**
No server needed. Works on `file://` URLs, GitHub Pages, and any static host. React Router would add 50 KB of JavaScript and require a server to handle deep links. Hash changes never trigger a page reload.

**`VIEWS` constant + `hashchange` listener**
Routing is intentional: only strings in the `VIEWS` array are valid routes. Unknown hashes fall back to `VIEWS[0]`. The `hashchange` listener handles browser back/forward correctly.

---

## Theme

**`[data-theme="light"]` attribute on `<html>`**
Toggle by setting `document.documentElement.setAttribute('data-theme', 'light')` (or removing the attribute for dark). No JavaScript needed in CSS — the cascade handles it. Adding `transition: background 0.2s, color 0.2s` on `body` gives a smooth swap.

**CSS custom properties on `:root`, never inline hex**
`var(--accent)` in ten places changes with one edit. Hardcoded `#e8a838` in ten places requires a search-and-replace and always misses one. This also makes light/dark theme overrides possible — `[data-theme="light"] { --bg: #f0f2f5; }` is the entire light theme.

---

## Google Drive sync

**PKCE OAuth2 (Proof Key for Code Exchange)**
Public clients (browser apps with no server) cannot keep a client secret truly secret. PKCE is the current standard for this flow — it uses a one-time verifier/challenge pair so the `code` returned by Google is useless to an interceptor who doesn't also have the verifier.

**Client Secret stored in `localStorage`**
Google's token endpoint requires it even for browser apps. This is a known trade-off, not a mistake — the Client Secret is not meaningfully more secret than the Client ID in a browser context. The scope (`drive.file`) limits damage: the app can only access files it created.

**Drive keys stored without the PFX prefix**
`driveToken`, `driveFileId`, and `drive_pkce_verifier` are stored without the app prefix. This means the same credentials work if you copy-paste them into a new app — the Drive file association persists. Apps that set different backup filenames (`APP_NAME-backup.json`) create separate Drive files automatically.

**`'TOKEN_EXPIRED'` string sentinel from `getValidAccessToken`**
The function returns three distinct values: `null` (no token), a token string (valid), or the literal string `'TOKEN_EXPIRED'` (token existed but `invalid_grant` from the refresh). The sentinel avoids throwing and lets callers branch cleanly: disconnect the UI, don't try to call Drive, don't silently swallow the error.

**2-second debounce on auto-save**
Prevents hammering the Drive API on rapid edits (e.g. typing in a text field). Short enough to feel live — 2s after the last keystroke, not 2s after the first. The debounce timer is stored in a `useRef` so it survives re-renders without causing them.

**`pendingDriveLoad` flag for post-OAuth load**
After the OAuth redirect back to the app, the component re-mounts fresh with no in-memory state. The OAuth callback `useEffect` sets `lsSet('pendingDriveLoad', true)` and a second effect watches for `driveConnected && pendingDriveLoad` to trigger the load. Using a flag rather than a closure avoids the stale-closure problem.

---

## Drive conflict detection

When local data and Drive data differ on reconnect, silently overwriting local is data loss. `loadFromDrive` compares `JSON.stringify(local) !== JSON.stringify(driveData)` before calling `setData`. If they differ and local is non-empty, `window.confirm` lets the user pick:

- **OK** → load Drive data (replaces local)
- **Cancel** → keep local data; pushes local → Drive immediately so both sides re-sync

`window.confirm` is appropriate here — it's a no-build, no-custom-modal app. Both branches end with both sides in sync.

**When to adapt:** if you rename the primary state key from `data` (e.g. to `playthroughs`), update `lsGet('data', [])` in `loadFromDrive` and the `saveToDrive({ version: 1, data: local })` call to match. The nav/tab-bar Settings ⚠ badge (triggered on `driveStatus === 'expired'` or `'error'`) should be removed if you remove the Drive block entirely.

---

## Anti-patterns (seen across audited repos, avoided here)

**Inline styles with hardcoded colors** (seen in FreezerBox)
`style={{ background: '#0e0f11' }}` makes theming, dark/light toggle, and future design changes impossible without touching every element. Always use `var(--token-name)` and define it in `:root`.

**All styling via inline `style` props** (seen in FreezerBox)
Harder to read, impossible to override with CSS, prevents pseudo-selectors (`:hover`, `:focus`). Use class names with CSS rules; reserve inline styles for truly dynamic values (e.g. a width derived from state).

**Using `localStorage` directly without helpers**
Bypasses the prefix, loses error handling, scatters JSON.parse/stringify throughout the code.
