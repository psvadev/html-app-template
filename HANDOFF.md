# HANDOFF — html-app-template

Dev-facing context: why this exists, what was audited, what was decided, and how to keep it current.

---

## Why this template exists

Four+ single-file React apps (PokéJournal, mealprepping, running, FreezerBox) were built using the same stack and patterns but each re-derived the wiring from scratch — or copied from a previous app that might go private or stale. This template is the canonical reference: copy it, rename the prefix, delete what you don't need.

**Goal:** A new app should be wirable in under 30 minutes with no decisions to make about routing, storage, theming, or Drive sync.

---

## Repos audited (2026-06-08)

| Repo | Findings |
|------|----------|
| **PokéJournal** (canonical) | Best design system: CSS vars, light/dark toggle, full component classes. Correct lsGet/lsSet. Most recent Drive sync. Hash routing. Template sources everything from here. |
| **mealprepping** | Origin of many patterns (PKCE Drive sync, 2s debounce). Hardcoded colors, dark-only. PokéJournal supersedes it as the reference. |
| **running** | Vanilla JS (not React) — stack not carried over. Best CSS custom-property naming of the three; confirmed the `--bg/--surface/--text` naming pattern. |
| **FreezerBox** | React + Babel, `fb_` prefix. Confirmed lsGet/lsSet, PKCE, 2s debounce, TOKEN_EXPIRED sentinel, Drive keys without prefix. New pattern: conflict modal for Drive merges (documented in PATTERNS.md, not implemented in template — last-write-wins is simpler and acceptable for most apps). Anti-pattern: no CSS vars, all inline styles. |
| **pokemon-tracker** | React + Babel, `pb_` prefix. Follows template pattern exactly. Two patterns backported: `storageKB()` utility (localStorage usage display in Danger Zone) and show/hide toggle buttons on Drive password inputs. |

---

## Key decisions

**PokéJournal as the canonical source**
It's the most recently developed app and has the most complete design system. mealprepping is older; running uses a different stack. All CSS vars, component classes, Drive sync code, and routing patterns in the template are sourced verbatim from PokéJournal's index.html.

**Drive sync as a clearly-marked optional block**
Most apps will want Drive sync. But some won't, and removing 50 lines of commented-out code is cleaner than adding 50 lines. The template marks the optional sections with `[OPTIONAL: ...]` / `[/OPTIONAL: ...]` comment banners so it's obvious what to delete as a unit.

**`data` as the primary state placeholder**
The template uses `const [data, setData] = useState(...)` as a generic placeholder. New apps rename this to their actual domain (e.g. `playthroughs`, `items`, `runs`). The Drive auto-save and export/import wire up to `data` so the connections are visible and easy to update.

**Drive keys without the PFX prefix**
`driveToken`, `driveFileId`, `drive_pkce_verifier` are stored without the app prefix. Same credentials work across apps; each app creates its own Drive file (keyed by `APP_NAME`).

**`APP_NAME` constant drives the backup filename**
`APP_NAME.toLowerCase().replace(/\s+/g, '-') + '-backup.json'` → e.g. `my-app-backup.json`. Changing `APP_NAME` is one of the five things to do when starting a new app.

**Rejected: framework/runtime coupling**
The template is a starting gun — not a shared library. Apps copy the file and diverge. No import, no version constraint, no maintenance obligation.

**Drive conflict detection (implemented)**
`loadFromDrive` now compares Drive data against local before overwriting. If they differ and local is non-empty, `window.confirm` lets the user pick which wins; "Cancel" immediately pushes local → Drive. This is the FreezerBox conflict pattern simplified to `window.confirm` — appropriate for a no-build, no-custom-modal app.

---

## Starting a new app — checklist

1. Create new directory (or GitHub repo)
2. Copy `template.html` → `index.html`
3. Copy `CLAUDE.md` → update app name at the top
4. In `index.html`:
   - `const PFX = 'app_'` → rename prefix (e.g. `'fb_'`)
   - `const APP_NAME = 'My App'` → your app name
   - `const VIEWS = ['home', 'settings']` → add your views
   - Replace `HomeView` placeholder with your first real view
   - Rename `data` / `setData` to your domain state
   - Update the favicon SVG in `<head>`
5. If no Drive sync: delete the two `[OPTIONAL: GOOGLE DRIVE SYNC]` blocks
6. If Drive sync: rename the primary state key in `loadFromDrive` — change `lsGet('data', [])` and `saveToDrive({ version: 1, data: local })` to match your key (e.g. `playthroughs`); also rename `setData(driveData)` to your setter
7. Create GitHub repo → enable Pages (Settings → Pages → Deploy from main branch root)
8. In Google Cloud Console: add the GitHub Pages redirect URI to your OAuth client
9. Test: open locally, verify theme toggle, hash routing, export/import

---

## How to keep the template current

When you improve a pattern in any app (better error handling, a new CSS class, a Drive sync fix), update `template.html` in the same commit. One line in the commit message: `backport to template`. That's the entire process — no ceremony.

When a pattern is confirmed across two or more apps, add it to PATTERNS.md with a *why*.

---

## Session log

### Session 3 — 2026-06-19
- Added `#root` fallback content ("Loading…") and a `window.onerror` handler before the Babel script tag — surfaces blank-page failures instead of showing a blank screen; React replaces the fallback on successful render
- Fixed Drive sync Bug 1 (silent data loss on reconnect): `loadFromDrive` now compares Drive data against local before overwriting; `window.confirm` lets user pick which wins; "Cancel" pushes local → Drive immediately; `saveToDrive` added to `loadFromDrive` dep array
- Fixed Drive sync Bug 2 (sync error gives no visible warning): Settings ⚠ badge in nav and tab bar now triggers on `driveStatus === 'expired'` OR `'error'` (was expired-only)
- Updated `CLAUDE.md`: added blank-page diagnostic note (3 causes in order), Drive conflict-resolution notes, badge removal reminder
- Updated `PATTERNS.md`: replaced "not implemented" conflict modal placeholder with actual implementation description; added `#root` fallback / `window.onerror` pattern explanation

### Session 2 — 2026-06-18
- Diagnosed blank-page bug reported across multiple apps: unpinned Babel CDN URL (`@babel/standalone` with no version) silently upgraded to Babel 8, breaking all `type="text/babel"` scripts with `Uncaught SyntaxError: import declarations may only appear at top level of a module` — error originates inside `babel.min.js`, not app code, making it hard to diagnose
- Fixed `template.html`: Babel CDN URL pinned to `@7`
- Updated `PATTERNS.md`: corrected the CDN section (previously implied both URLs were already pinned, which was wrong); added dedicated Babel 8 symptom/fix/revisit section
- Updated `README.md` and `CLAUDE.md`: added CDN pinning note as a hard constraint

### Session 1 — 2026-06-08
- Audited mealprepping, running, FreezerBox; read PokéJournal source directly
- Created: `template.html`, `CLAUDE.md`, `README.md`, `PATTERNS.md`, `HANDOFF.md`, `.gitignore`
- Template sources all patterns from PokéJournal (canonical)
- Drive sync included as a clearly-marked optional block (present but easy to remove)
- FreezerBox conflict modal documented in PATTERNS.md; not implemented (last-write-wins sufficient)
- FreezerBox anti-pattern (no CSS vars) documented as a negative example
- pokemon-tracker (local, not on git) also audited; two patterns backported: `storageKB()` in Danger Zone + show/hide toggles on Drive password inputs
