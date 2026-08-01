# HANDOFF — html-app-template

Maintained per `HANDOFF-INSTRUCTIONS.md`. Update in the same session a feature ships.

---

## 1. What this is

The canonical template for the user's single-file React apps (PokéJournal, mealprepping, FreezerBox, pokemon-tracker lineage): copy `template.html`, rename the `PFX` prefix, delete the optional blocks you don't want, and a new app is wirable in under 30 minutes with no decisions about routing, storage, theming, or sync. Stack: React 18 + Babel Standalone 7 via CDN (no build step), `localStorage` persistence behind `lsGet`/`lsSet`, GitHub Pages hosting, optional Google Drive sync (PKCE OAuth, `drive.file` scope) and optional version footer.

## 2. Current state

- Complete and working as of 2026-07-12. All eight hardening items from the Løpelogger audit (github.com/psvadev/running, July 2026) are backported; each was verified by rendering `template.html` headless (home + `#settings` views) with zero console errors.
- Pushed to `origin/main` 2026-07-12 (backports + handoff-standard rollout). Check `git log origin/main..HEAD` before assuming GitHub is current.
- Verification pillar (PATTERNS.md, CLAUDE.md, `window._selftest()` in `template.html`) added 2026-08-01 (Session 6) — see Appendix C. Not yet pushed; commits on `main` only.
- To run/test: open `template.html` directly in a browser (`file://` works; Drive sync itself needs HTTPS or localhost). It must render past the `#root` "Loading…" fallback with zero console errors on both the home and `#settings` views.
- Headless verification on this machine: **use Firefox, not Edge** (see Quirks). Pattern: `& "C:\Program Files\Mozilla Firefox\firefox.exe" -no-remote -profile <throwaway-dir> --headless --window-size=1280,1600 --screenshot out.png <file:///-url>` then inspect the PNG. Working script: `check-ff.ps1` (session scratchpad, recreate if gone).
- Nothing half-done.

## 3. How things fit together

- All state lives in `App()`; views are pure display components receiving props. Per-key persistence: `useState(() => lsGet(key, default))` + `useEffect(() => { lsSet(key, val) }, [val])`. Every localStorage touch goes through `lsGet`/`lsSet`, which apply the `PFX` prefix — so renaming `PFX` once re-namespaces the whole app.
- The primary `data` effect is special: it checks `lsSet`'s boolean return and raises a persistent (`duration 0`) error toast on quota failure. Other effects ignore the return but need braced bodies so React never receives the boolean as a cleanup value.
- **Drive sync is a gated loop, not two independent functions.** `loadFromDrive` runs at connect/startup; only a *successful read* (a real file, or the confirmed no-file first run) sets `driveLoadConfirmed.current = true`, which is what unlocks `saveToDrive`. The auto-save effect (2s debounce on `data`) also fires at mount — the gate is what stops that mount-time save from racing a slow startup load and overwriting Drive.
- `saveToDrive` and `loadFromDrive` each need the other (conflict resolution both directions); the circular `useCallback` dependency is broken with `loadFromDriveRef`, a latest-callback ref. Conflict resolution calls `loadFromDriveRef.current({ skipConfirm: true })` so the user isn't prompted twice.
- `driveModifiedTime` (prefixed key) is recorded at every successful load *and* save. Before PATCHing an existing file, `saveToDrive` fetches Drive's current `modifiedTime` and compares — Drive newer means another device wrote since we last looked, so prompt instead of overwrite. Fail-open when there's no baseline.
- `VersionFooter` is self-contained: fetches the latest commit for `REPO_FILE` from GitHub's public API, renders nothing while `REPO_OWNER` still starts with the `'your-'` placeholder, fails silently on any error.

## 4. Quirks & gotchas

- **Unpinned Babel CDN URL silently upgrades to Babel 8** → blank page with `SyntaxError` that originates *inside `babel.min.js`*, not app code — which is exactly why it took a session to diagnose (Session 2). Both CDN pins (`@18`, `@7`) are load-bearing.
- **The version footer proves "latest exists on GitHub", not "the browser is running latest"** — a stale cached page happily shows the newest SHA. Hard refresh before concluding a deploy failed.
- **Drive keys ARE prefixed.** `driveToken`/`driveFileId`/`driveModifiedTime` go through `lsGet`/`lsSet`; only `drive_pkce_verifier` is raw unprefixed `localStorage` (one-shot around the OAuth redirect). Docs claimed the opposite from Session 1 until corrected 2026-07-12 (e12e6fb) — trust the code, not older prose.
- **`toISOString().slice(0, 10)` is UTC** — it mis-dates entries logged after local midnight. `localISODate()` exists for user-entry dates; it ships with no call sites (like `uuid()`), which is intentional.
- **Drive `modifiedTime` is RFC 3339 UTC** — plain lexicographic string comparison is a correct ordering; no Date parsing needed.
- **Headless Edge (`--dump-dom`) silently produces nothing on this machine** when the user's Edge startup-boost instance is running: the launcher exits 0 in ~1.5s with no output and no child processes. It is *not* the tool sandbox (verified). Use headless Firefox with a throwaway profile instead.
- **GUI-subsystem apps don't block PowerShell `&`** — they return immediately with no exit code. Use `Start-Process` and poll for the output file.

## 5. Decision log

### Accepted / shipped

- **PokéJournal as the canonical source** — most recently developed app, most complete design system (CSS vars, light/dark, component classes). mealprepping is older; running is a different stack. All template patterns sourced from its `index.html`.
- **Optional features as delete-unit blocks** (`[OPTIONAL: …]` banners) — most apps want Drive sync, some won't, and removing 50 marked lines is cleaner than adding 50 unmarked ones.
- **Generic `data`/`setData` placeholder state** — Drive auto-save and export/import wire to it so the connections are visible and easy to rename per app.
- **Drive keys per-app prefixed** (docs corrected 2026-07-12) — per-app tokens are the safer default: disconnecting one app can't break another.
- **Drive upload gating via `driveLoadConfirmed` ref** (d966ecd) — the audit's highest-severity finding: never upload over cloud data this session hasn't successfully read, or a failed load plus one local edit silently destroys the Drive copy. A ref, not state, to avoid stale closures in the debounced save.
- **`modifiedTime` pre-check before PATCH** (ef2fc30) — catches the stale-device sequential conflict (device A saves, device B with old data saves later) that load-time comparison alone misses.
- **Item counts in the conflict prompt** (36747b9) — an uninformed binary choice is how a near-empty local copy overwrites the full cloud copy.
- **Rolling last-3 snapshots before whole-dataset replaces** (1421f23) — import restore and Drive conflict load call `snapshotData()` first. Clear All Data is exempt by design: deletion is its intent and its confirm promises permanence.
- **`lsSet` returns success/failure; quota failure raises a sticky toast** (4d71c46) — tolerate the error, never hide it. `lsSet` must never throw.
- **`window.confirm` for all conflict/confirm UI** — appropriate for a no-build, no-custom-modal app; the FreezerBox modal pattern stays documented in PATTERNS.md as the upgrade path.
- **`localISODate()`** (30cadf5) and **optional `VersionFooter`** (d678293) — see Quirks for the why of each.
- **`## Verification` pillar added to PATTERNS.md** (Session 6) — pure-logic-in/out, `window._selftest()` harness, Python-port escalation (with the "hand transcription, not an import" cost stated explicitly), invariant assertions, and reproducing external aggregates, plus two copy-paste snippets (structural balance check, deploy-verification loop). Sourced from a second sister-project audit (a bug run, not a code review); nothing app-specific crossed over, only the shapes.
- **Invariants over examples elevated to the primary verification technique** — in the sister project, six UI bugs were each caught by an invariant, none by an example test; one bug survived 51 passing example tests because every example shared the same blind spot as the code that produced it (self-similar tests inherit the code's blind spot).
- **`window._selftest()` optional block** (Session 6, `template.html`) — zero-dependency harness callable from DevTools; asserts against `localISODate` and an `lsGet`/`lsSet` round-trip. Tests the code that actually shipped, not a copy, which is what makes it useful for confirming a deploy landed.
- **Version footer's honest caveat completed with a concrete alternative** (Session 6, `CLAUDE.md`) — it already said the footer proves existence, not liveness; added the missing half: grep the deployed file for a marker string to prove liveness.
- **`<input type="month">`/`week` flagged unsafe for round-tripped data, GitHub Pages one-commit-behind fix, conditional `## Derived data` and `## Showing numbers` sections** (Session 6, PATTERNS.md) — small, generic entries from the same audit.

### Rejected

- **Framework/runtime coupling (template as a shared library)** — permanent, don't re-propose. The template is a starting gun: apps copy the file and diverge. No import, no version constraint, no maintenance obligation.
- **Custom conflict-resolution modal (FreezerBox style)** — conditional. `window.confirm` with item counts is enough for now; revisit if the binary choice proves insufficient in a real app.
- **Backporting vanilla-JS audit lessons React already covers** (output escaping, listener hygiene) — permanent for this stack; React's JSX escaping and effect cleanup are the mechanism.

### Deferred / parked

- **Snapshot recovery UI** — recovery is currently manual via DevTools console (steps in PATTERNS.md). Parked because snapshots exist for rare disasters. Shape when built: a Danger Zone list of the 3 snapshots with timestamps and a restore button (which itself snapshots first). **Un-park trigger:** anyone actually performs a console recovery more than once.
- **Cloudflare Worker proxy** — documented in PATTERNS.md only. Shape: a tiny Worker holding the secret, app calls it, honest CORS limitation noted. **Un-park trigger:** an app needs a server-side secret (an API key that can't live client-side).
- **Field-level Drive merge** — last-write-wins with informed prompts shipped instead. Shape: per-item timestamps merged by recency. **Un-park trigger:** real data loss, or user complaints that the whole-dataset binary choice keeps discarding wanted edits despite the counts.
- **No `LICENSE` file** — a public-repo security check (2026-08-01) found no secrets or PII in the tracked files or full git history (grepped for OAuth/API-key/token/private-key patterns), and confirmed the Client Secret's client-side storage is an accepted, documented tradeoff (PATTERNS.md → Google Drive sync). No `LICENSE` means nobody else has explicit legal permission to reuse the code, even though it's already fully visible (client-side JS deployed to GitHub Pages is public regardless of repo visibility) — but the user confirmed the repo is "for my use only at the moment hence not so worried about the license file", so this is deliberately staying parked, not an oversight. Shape when unparked: add an MIT `LICENSE` at repo root — matches the "starting gun, no obligations" philosophy in the Rejected section above. **Un-park trigger:** the user decides they want others able to legally fork/reuse the template, or someone asks.

## 6. Workflow rules

- Commit directly on `main`, **one commit per logical item**. **NEVER push** — the user reviews and pushes themselves.
- PATTERNS.md (the *why*, in systems terms) and CLAUDE.md (the operating rule) are updated **in the same commit** as the code change they describe.
- Every change to `template.html` is verified in a browser before commit: renders past "Loading…", zero console errors, home + `#settings` views. Headless Firefox screenshot is the working method on this machine.
- Multi-item work ends with a summary table of commits for review.
- Changes larger than ~20 lines of code: describe the approach in one paragraph and get confirmation first (CLAUDE.md rule).
- Keeping the template current: improve a pattern in any app → update `template.html` in the same commit, note `backport to template` in the message. A pattern confirmed across 2+ apps earns a PATTERNS.md entry with a *why*.
- This file follows `HANDOFF-INSTRUCTIONS.md` and is updated in the same session a feature ships.

## 7. Learning log

- Verify docs against code before repeating a claim — "Drive keys are unprefixed" survived from Session 1 to Session 4 in three docs while the code always prefixed them.
- A passing smoke check must assert something unique to the surface under test — the first headless checker matched `nav-brand`, present in every view, so PASS never proved Settings actually mounted (Session 4).
- When a headless browser silently produces nothing, suspect an already-running instance of that browser before suspecting the tooling — Edge startup-boost, 2026-07-12.
- Pin CDN majors and treat "error inside the vendor script" as a version-drift symptom — the Babel 8 blank page (Session 2) presented as a `SyntaxError` in `babel.min.js`, not in app code.
- Don't carry audit findings across stacks mechanically — Løpelogger's escaping/listener findings were correct for vanilla JS and redundant for React; filter by mechanism, not by checklist.
- Invariants over examples: a test suite built from examples inherits the blind spot of the code it was written against — an invariant that bounds the output catches what more examples of the same shape can't (sister-project bug run, Session 6).

---

## Appendix A — Repos audited (2026-06-08)

| Repo | Findings |
|------|----------|
| **PokéJournal** (canonical) | Best design system: CSS vars, light/dark toggle, full component classes. Correct lsGet/lsSet. Most recent Drive sync. Hash routing. Template sources everything from here. |
| **mealprepping** | Origin of many patterns (PKCE Drive sync, 2s debounce). Hardcoded colors, dark-only. PokéJournal supersedes it as the reference. |
| **running** | Vanilla JS (not React) — stack not carried over. Best CSS custom-property naming of the three; confirmed the `--bg/--surface/--text` naming pattern. Its July 2026 audit is the source of the Session 4 hardening backports. |
| **FreezerBox** | React + Babel, `fb_` prefix. Confirmed lsGet/lsSet, PKCE, 2s debounce, TOKEN_EXPIRED sentinel. New pattern: conflict modal for Drive merges (documented in PATTERNS.md, not implemented — see Decision log). Anti-pattern: no CSS vars, all inline styles. |
| **pokemon-tracker** | React + Babel, `pb_` prefix. Follows template pattern exactly. Two patterns backported: `storageKB()` utility (localStorage usage display in Danger Zone) and show/hide toggle buttons on Drive password inputs. |

## Appendix B — Starting a new app checklist

1. Create new directory (or GitHub repo)
2. Copy `template.html` → `index.html`
3. Copy `CLAUDE.md` → update app name at the top
4. Copy `HANDOFF-INSTRUCTIONS.md` → write the app's `HANDOFF.md` following it
5. In `index.html`:
   - `const PFX = 'app_'` → rename prefix (e.g. `'fb_'`)
   - `const APP_NAME = 'My App'` → your app name
   - `const VIEWS = ['home', 'settings']` → add your views
   - Replace `HomeView` placeholder with your first real view
   - Rename `data` / `setData` to your domain state
   - Update the favicon SVG in `<head>`
6. If no Drive sync: delete the two `[OPTIONAL: GOOGLE DRIVE SYNC]` blocks; if no version footer: delete the `[OPTIONAL: VERSION FOOTER]` block (otherwise set `REPO_OWNER`/`REPO_NAME`)
7. If Drive sync: rename the primary state key in `loadFromDrive` — change `lsGet('data', [])` and `saveToDrive({ version: 1, data: local })` to match your key (e.g. `playthroughs`); also rename `setData(driveData)` to your setter
8. Create GitHub repo → enable Pages (Settings → Pages → Deploy from main branch root)
9. In Google Cloud Console: add the GitHub Pages redirect URI to your OAuth client
10. Test: open locally, verify theme toggle, hash routing, export/import

## Appendix C — Session log

### Session 6 — 2026-08-01
- Added a `## Verification` pillar to PATTERNS.md: pure-logic-in/out, `window._selftest()` harness, Python-port escalation, invariant assertions (the highest-yield entry), reproducing external aggregates, plus a structural-balance snippet and a deploy-verification curl loop — sourced from a second sister-project bug run, generalized so nothing app-specific carried over
- Added `window._selftest()` optional block to `template.html` (asserts `localISODate` and an `lsGet`/`lsSet` round-trip; callable from DevTools) — mentioned in README's optional-blocks list and checklist
- Added two hard rules to `CLAUDE.md`: pure-logic-in/out under Architecture; "grep the deployed file for a change marker" as the concrete alternative to trusting the version footer, under What NOT to do
- Added four more PATTERNS.md entries: `<input type="month">`/`week` unsafe for round-tripped data (Dates), GitHub Pages one-commit-behind + `.nojekyll` (Stack), conditional `## Derived data` (version-stamp cached results; bumping the version costs a full re-derivation), `## Showing numbers` (blank beats wrong; one-word provenance label)
- Verified `template.html` still renders past "Loading…" with zero console errors (headless Firefox screenshot)
- Public-repo security check: grepped the current tree and full `git log -p` history for secret/token/private-key patterns and for PII — clean on both. Flagged the missing `LICENSE` file as a deferred item (see Decision log)

### Session 5 — 2026-07-12
- Added `HANDOFF-INSTRUCTIONS.md`: the user's standard spec for handoff docs, to be copied into every new project so all repos document themselves the same way
- Restructured this file to that spec (7 sections + appendices); wired the instruction file into README's file table/checklist and CLAUDE.md

### Session 4 — 2026-07-12
- Backported hardening lessons from the Løpelogger audit (github.com/psvadev/running, July 2026 — same architecture philosophy, vanilla JS), one commit per item; lessons React already covers (output escaping, listener hygiene) were deliberately not carried over
- Drive sync: upload gating via `driveLoadConfirmed` ref — never upload before a successful read; also fixes the mount-time auto-save racing a slow startup load. `modifiedTime` pre-check before PATCH catches the stale-device sequential conflict; conflict prompt now shows item counts for both sides
- Data safety: `snapshotData()` keeps a rolling last-3 under `snapshots` before import restore / Drive conflict load (Clear All Data exempt by design — its confirm promises permanence); `lsSet` now returns success/failure and the primary data effect raises a persistent quota toast (`showToast` gained a duration param, 0 = sticky)
- Added `localISODate()` (user-entry dates — `toISOString().slice(0,10)` is UTC and mis-dates post-midnight entries) and the optional `VersionFooter` block (deploy confirmation via GitHub's public API, with the stale-cache caveat documented)
- PATTERNS.md: `dangerouslySetInnerHTML` anti-pattern (was a stored XSS with token theft in reach in the sister project) and the Cloudflare Worker proxy pattern for apps that ever need a server-side secret
- Verified after each item by rendering `template.html` headless (home + `#settings` views) — zero console errors, React past the "Loading…" fallback
- Corrected a docs/code mismatch found during the backport: `driveToken`/`driveFileId` are stored prefixed via `lsGet`/`lsSet` (docs claimed unprefixed since Session 1); only `drive_pkce_verifier` is raw `localStorage`

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
