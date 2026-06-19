# html-app-template

A starting point for single-file React apps — no build step, no dependencies, works anywhere a browser can open a file.

## What's in the template

| File | Purpose |
|------|---------|
| `template.html` | The scaffold — copy this to start a new app |
| `CLAUDE.md` | Copy into your new repo; Claude Code reads it for architecture constraints |
| `PATTERNS.md` | Reference: the *why* behind each non-obvious pattern |
| `HANDOFF.md` | Dev context: audit findings, decisions, how to keep the template current |

## Starting a new app

1. Create a new directory (or GitHub repo) for your app
2. Copy `template.html` → rename to `index.html`
3. Copy `CLAUDE.md` into the repo → update the app name at the top
4. Edit `index.html`:
   - Change `const PFX = 'app_'` → your app's prefix (e.g. `'fb_'`)
   - Change `const APP_NAME = 'My App'` → your app's name
   - Change `const VIEWS = ['home', 'settings']` → add your views
   - Replace `HomeView` with your first real view
   - Update the favicon SVG in `<head>` (optional)
5. If you don't need Google Drive sync, delete the two marked optional blocks (see comments in the file)

See `PATTERNS.md` for the reasoning behind each architectural choice.

## Google Drive sync setup

Drive sync requires HTTPS — GitHub Pages is the easiest free host.

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a project → **APIs & Services** → **OAuth consent screen** (External, add your email as test user)
3. **Credentials** → **Create Credentials** → **OAuth 2.0 Client ID** → Web application
4. Add an **Authorized redirect URI**: `https://your-username.github.io/your-repo-name/`
   - Include the trailing slash
   - Also add `http://localhost:PORT/` if you want to test locally via a simple server
5. Copy the **Client ID** and **Client Secret** into the app's Settings page
6. Click **Connect Google Drive** — you'll be redirected to Google's consent screen once

Credentials are stored in `localStorage` under unprefixed keys (`driveToken`, `driveFileId`). They're never sent anywhere except Google's OAuth and Drive APIs.

## Stack

- React 18 via CDN (no build step)
- Babel Standalone 7 for JSX transform in the browser
- Zero runtime dependencies beyond those two CDN scripts

> **CDN version pinning:** both scripts use a major-version pin (`@18`, `@7`). Do not remove the version from the Babel URL — unpinned Babel silently upgrades to Babel 8, which breaks `type="text/babel"` scripts with a blank page and a misleading `SyntaxError` inside `babel.min.js`. See PATTERNS.md for the full diagnosis.

> **Blank page on load?** The template includes a "Loading…" fallback inside `#root` and a `window.onerror` handler that updates it to "⚠ Script error — open DevTools (F12)". If the page stays blank with no message, open DevTools → Network tab and check whether the CDN scripts loaded. Common causes: stripped Babel version pin, CDN blocked, or a runtime JS error on first render.
