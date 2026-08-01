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

**GitHub Pages occasionally serves one commit behind after a push.**
Fix with an empty commit (`git commit --allow-empty`) and check the Actions tab for deploy status. Add a `.nojekyll` file at the repo root so Jekyll doesn't process the site — without it, paths beginning with an underscore are dropped silently.

**`#root` fallback + `window.onerror` for blank-page diagnosis**
The template puts a "Loading…" message inside `<div id="root">` and registers a `window.onerror` handler *before* the Babel script tag. React replaces the fallback content immediately on a successful render. On failure the fallback remains visible. Three blank-page causes in order of likelihood:

1. **Babel pin stripped** → Babel 8 breaks JSX silently; "Loading…" persists, console shows the `import declarations` error
2. **CDN blocked** (corporate firewall, ad-blocker) → "Loading…" persists, Network tab shows failed requests
3. **Runtime error on first render** → `window.onerror` updates the fallback to "⚠ Script error — open DevTools (F12)"

Without the fallback all three look identical: a blank page.

---

## Verification

No build step means no test runner. That doesn't mean no verification — it means choosing techniques that cost nothing to set up and don't fight the single-file constraint.

**Keep pure logic pure — data in, data out.**
Business logic takes data and returns data: no DOM reads, no `fetch`, no module-scope mutation, no `Date.now()` inside the calculation (pass the time in as a parameter). This costs nothing to follow and is the only thing that makes logic testable when there's no test runner — a function that reaches into the DOM can only be exercised by driving a browser.

*When this applies:* always. It's free.

**Ship `window._selftest()` — a self-test harness that runs with the app.**
A function that asserts against the app's own pure functions and prints a pass/fail count to the console. Zero dependencies, works from `file://`, and — unlike a separate test suite — it tests the code that actually shipped, not a copy of it. That also answers a second question for free: "did the deploy really get the fix?" Run it from DevTools after a deploy, not just after a local edit. See the optional `[OPTIONAL: SELF-TEST HARNESS]` block in `template.html`.

*When this applies:* any project with more than trivial logic.

**Port pure logic to a language with a real test runner — an escalation, not a default.**
When logic gets numeric or algorithmic enough that hand-checked examples stop being convincing, transcribe the pure functions into a language with a test runner (Python needs no setup on most machines) and run that as the pre-commit gate.

The honest cost: **this is a hand transcription, not an import.** Nothing links the two, so the standing rule is *a change to a pure function updates its port in the same commit*. A port that silently tests superseded behavior is worse than no port — it returns false confidence. In the sister project this bit exactly once: a result-shape change needed a manual port patch, and four assertions failed and had to be corrected by hand.

Don't judge a port by how often it's edited, either — in that project four of seven ports were never touched again after the day they were written, but *running* them on every commit is what proved a later refactor changed no behavior.

*When this applies:* numeric/algorithmic logic where a wrong answer can look plausible. Skip it for CRUD and display code.

**Assert invariants, not examples.**
An example test says "for this input, expect this output" — which only ever checks the cases someone thought of. An invariant says "no output may ever violate this," and it holds for inputs nobody imagined. This is the highest-yield form of verification available in a no-test-runner stack.

Generic invariants worth reaching for:
- a part cannot exceed its whole (a subtotal ≤ its total; a filtered list ≤ its source)
- a derived total must equal the sum of its parts
- a percentage stays within 0–100; a probability within 0–1
- a "best over any window" cannot beat the best rate actually present in the data
- a curve that must be monotonic must actually be monotonic
- a residual must be physically possible — if removing a part leaves an impossible remainder, the part is wrong
- two numbers shown side by side must share a scale, a unit, and a clock

In the sister project, six bugs reached the UI, and every one was caught by an invariant the output could not satisfy — none by reading code. One of them survived 51 passing example tests, because every test used constant-rate data where both ends of the computed window shared one rate — precisely the condition that hid the bug. The invariant "no window can be faster than the fastest rate present in the data" caught it in one line.

The lesson: **self-similar tests inherit the blind spot of the code they were written against.** Adding more examples in the same shape doesn't help; adding a bound the output cannot violate does.

*When this applies:* always, wherever a bound on the output can be stated. It's the first thing to reach for, not the last.

**Reproduce a number the other system already computed.**
When integrating with an external API that publishes a computed aggregate, make your own computation reproduce it. That's ground truth authored by someone else from the same data — worth more than any self-written test.

In the sister project, a threshold was settled by checking that the derived figure reproduced the provider's own published aggregate to within one unit, on two independent records. Before that check, three different plausible thresholds all "looked right."

*When this applies:* any integration where the provider exposes a summary you also derive.

**Two copy-paste snippets:**

*Structural check standing in for a compiler* — when this applies: single-file app, no build step, nothing to parse the file. It doesn't prove validity; it proves an edit didn't change structural balance, which is what hand-editing a large file actually breaks.

```python
"""Bracket/tag balance vs git HEAD.  Usage: python balcheck.py <file>"""
import subprocess, sys
from pathlib import Path

REPO = Path(__file__).resolve().parent          # adjust if the script lives in a subfolder
TARGET = sys.argv[1] if len(sys.argv) > 1 else "index.html"

head = subprocess.run(["git", "show", f"HEAD:{TARGET}"], cwd=REPO,
                      capture_output=True).stdout.decode("utf-8")
cur = (REPO / TARGET).read_text(encoding="utf-8")
counts = lambda t: {c: t.count(c) for c in "{}()[]"}
a, b = counts(head), counts(cur)

bad = False
for o, cl in ("{}", "()", "[]"):
    net_head, net_cur = a[o] - a[cl], b[o] - b[cl]
    ok = net_head == net_cur
    bad |= not ok
    print(f"net {o}{cl}: HEAD {net_head:+d} -> now {net_cur:+d}   {'OK' if ok else 'CHANGED'}")
for tag in ("<script", "</script>"):
    print(f"{tag:<12} HEAD {head.count(tag):>3}   now {cur.count(tag):>3}")

print("RESULT:", "STRUCTURE UNCHANGED" if not bad else "*** IMBALANCE INTRODUCED ***")
sys.exit(1 if bad else 0)
```

*Deploy verification* — when this applies: any static host. Pick a string that exists only in the new code — a new function name works well.

```bash
# Confirm the change is actually being served, not just pushed.
MARKER="myNewFunction"
for i in $(seq 1 30); do
  if curl -s "https://USER.github.io/REPO/index.html?cb=$RANDOM" | grep -q "$MARKER"; then
    echo "DEPLOYED after ~$((i*10))s"; exit 0
  fi
  sleep 10
done
echo "NOT DEPLOYED after 300s"; exit 1
```

---

## Storage

**`const PFX = 'app_'` prefix on all keys**
Multiple apps can be open in the same browser origin (e.g. `file://` or `localhost`) without their `localStorage` keys colliding. Renaming the prefix is the one edit that scopes an app.

**`lsGet` / `lsSet` helpers**
Every localStorage access goes through these two functions. They centralise the JSON parse/stringify, and `lsGet` returns the fallback on any parse error. Direct `localStorage.getItem` calls bypass the prefix and error handling.

**`lsSet` tolerates write errors but never hides them.**
`lsSet` is non-throwing — a full quota must not crash the app mid-interaction — but it returns `false` on failure instead of swallowing it. The distinction matters: an earlier version of this template used `catch {}`, and the lesson (from the Løpelogger audit) is that a swallowed `QuotaExceededError` means the user keeps editing while believing data is saved, and finds out when they close the tab. The primary data persistence effect checks the return and raises a *persistent* error toast ("Local storage full — changes are NOT being saved" — `showToast(msg, 'error', 0)`, duration 0 = no auto-dismiss). Every subsequent failed write re-raises it, so the warning stands exactly as long as the problem does.

Only the primary `data` effect surfaces the toast: it's the write that grows with use and hits quota first, and the one whose loss hurts. Settings-sized writes failing means quota is already blown — and the data toast is already up.

**Lazy `useState` init + paired `useEffect` persistence**
```js
const [items, setItems] = useState(() => lsGet('items', []));
useEffect(() => lsSet('items', items), [items]);
```
The lazy init reads from storage once on mount. The effect writes only when state actually changes — not on every render. This is the correct pattern; do not inline `lsSet` inside event handlers.

---

## Derived data (conditional — only if the app caches computed results)

**Version-stamp cached derived data.**
If the app caches a computed result (not raw user input), stamp each cached entry with a version constant, keep an `isCurrent()` check, and provide a backfill that re-derives stale entries.

The cost that's easy to miss: **bumping the version makes every cached entry stale, so the user pays a full re-derivation.** That makes a version bump a real cost to weigh, not a free flag — a refactor that changes no output must not bump it.

*When this applies:* only apps that cache computed data. Skip it entirely if everything stored is raw user input.

---

## Dates

**User entries are dated with `localISODate()`, never `toISOString().slice(0, 10)`**
`toISOString()` is UTC. In any UTC+ timezone, an entry logged after local midnight but before UTC midnight gets *yesterday's* date — in Oslo (UTC+1/+2), a run logged at 00:30 files under the previous day. The bug is invisible in testing (nobody tests at half past midnight) and users read it as "the app put my entry on the wrong day", which it did. `localISODate(d?)` builds `YYYY-MM-DD` from the local calendar fields directly, so the date is always the one on the user's wall clock. Machine timestamps (sync times, snapshot times) correctly stay full ISO — `localISODate` is for the *calendar day a human means*.

**Never use native `<input type="month">` (or `week`) for data that round-trips through storage.**
Firefox renders `month` as a plain unvalidated text box — a value typed in the wrong format is accepted silently and read back as garbage. Two `<select>`s (month + year) are uglier and correct. `<input type="date">` is safe; it's `month`/`week` that lack cross-browser pickers.

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

**Drive key storage: prefixed, except the PKCE verifier**
`driveToken`, `driveFileId`, and `driveModifiedTime` go through `lsGet`/`lsSet` like everything else, so they carry the app prefix — each app connects and holds its own token and Drive file association independently. (Earlier versions of these docs claimed the keys were unprefixed for cross-app credential reuse; the code never did that, and per-app tokens are the safer default anyway — disconnecting one app can't break another.) Only `drive_pkce_verifier` is written with raw `localStorage`: a one-shot value created immediately before the OAuth redirect and consumed immediately after it.

**`'TOKEN_EXPIRED'` string sentinel from `getValidAccessToken`**
The function returns three distinct values: `null` (no token), a token string (valid), or the literal string `'TOKEN_EXPIRED'` (token existed but `invalid_grant` from the refresh). The sentinel avoids throwing and lets callers branch cleanly: disconnect the UI, don't try to call Drive, don't silently swallow the error.

**2-second debounce on auto-save**
Prevents hammering the Drive API on rapid edits (e.g. typing in a text field). Short enough to feel live — 2s after the last keystroke, not 2s after the first. The debounce timer is stored in a `useRef` so it survives re-renders without causing them.

**`pendingDriveLoad` flag for post-OAuth load**
After the OAuth redirect back to the app, the component re-mounts fresh with no in-memory state. The OAuth callback `useEffect` sets `lsSet('pendingDriveLoad', true)` and a second effect watches for `driveConnected && pendingDriveLoad` to trigger the load. Using a flag rather than a closure avoids the stale-closure problem.

**If an app ever needs a real server-side secret: tiny Cloudflare Worker proxy**
Drive sync works client-side because PKCE + `drive.file` scope are designed for browser apps. Some APIs aren't — they only offer a key that must stay secret, and a secret shipped in a single HTML file is public. The pattern that preserves this stack: a ~30-line Cloudflare Worker (free tier) holds the key as an environment secret, forwards requests to the API, and sets CORS headers from an explicit origin allowlist (your GitHub Pages origin, maybe `localhost`). The app fetches the Worker instead of the API; the single-file, no-build property of the app itself is untouched. **The honest limit:** CORS only stops *browsers on other origins* — anyone can `curl` the Worker directly. It hides the key, not the capability; it is not a hard gate. If abuse matters, the Worker also needs its own auth or rate limiting.

---

## Drive conflict detection

When local data and Drive data differ on reconnect, silently overwriting local is data loss. `loadFromDrive` compares `JSON.stringify(local) !== JSON.stringify(driveData)` before calling `setData`. If they differ and local is non-empty, `window.confirm` lets the user pick:

- **OK** → load Drive data (replaces local)
- **Cancel** → keep local data; pushes local → Drive immediately so both sides re-sync

`window.confirm` is appropriate here — it's a no-build, no-custom-modal app. Both branches end with both sides in sync.

**The prompt states item counts on both sides** (`Local: 3 items · Drive: 248 items`). A user can't choose between two opaque labels — and the costly mistake has a shape: keeping a near-empty local copy (fresh browser, cleared storage), which then pushes over the full cloud copy. With counts, that choice looks as wrong as it is *before* it happens. The count falls back to "unknown size" when the state isn't an array — apps that reshape `data` should adapt the label, not remove it.

**When to adapt:** if you rename the primary state key from `data` (e.g. to `playthroughs`), update `lsGet('data', [])` in `loadFromDrive` and the `saveToDrive({ version: 1, data: local })` call to match. The nav/tab-bar Settings ⚠ badge (triggered on `driveStatus === 'expired'` or `'error'`) should be removed if you remove the Drive block entirely.

---

## Drive upload gating

**Never upload over cloud data you haven't successfully read first — a failed load must block sync loudly, not let the next edit win silently.**

The failure mode without the gate: the startup `loadFromDrive()` fails (network error, transient API failure → status `'error'`), the user makes one local edit, and 2 seconds later the auto-save uploads whatever partial local state exists — overwriting the intact Drive copy that was never read. The device that *failed to sync down* wins over the device that synced up. This was the highest-severity finding in the Løpelogger audit.

`driveLoadConfirmed` is a `useRef(false)` set to `true` in exactly two places:

1. A successful read of the Drive file's content (set *before* the conflict prompt — the read succeeded regardless of which side the user picks)
2. The first-run path where no Drive file exists yet — there is nothing on Drive to protect, and uploads must be allowed to create the file

`saveToDrive` returns immediately while the flag is false. The refusal is silent by design: the *loud* part is the failed load itself (`driveStatus 'error'` → "⚠ Sync error" in Settings plus the nav/tab ⚠ badge), and the retry is the existing **Load now** button. A blocked save during a normal connect flow (load still in flight) should not flash an error.

The gate also fixes a subtler race: the auto-save effect runs on mount, so on every startup it schedules an upload of the *initial local state* at T+2s. If the startup load takes longer than 2 seconds, the upload used to win — local overwrote Drive before the load ever completed. With the gate, that first save is refused and the post-load save (triggered by `setData`) uploads instead.

It's a ref rather than state because the flag never drives rendering, and because `loadFromDrive`'s "keep local" branch calls `saveToDrive` directly — with state, that call would close over the stale pre-load value; a ref is always current.

The flag is in-memory and per-session: every page load must re-earn the right to upload by reading Drive first. `disconnectDrive` resets it.

---

## Remote-change pre-check before upload

**The realistic multi-device failure is sequential, not simultaneous:** phone edits and syncs; laptop — loaded this morning, now stale — makes one edit, and its auto-save silently overwrites the phone's version. Upload gating (above) doesn't catch this: the laptop's startup load *succeeded*, it's just old.

Before PATCHing an existing file, `saveToDrive` makes one cheap metadata call (`fields=modifiedTime`) and compares it against `driveModifiedTime` — the timestamp this device recorded at its last successful load or save (stored via `lsGet`/`lsSet`). If Drive is newer than what this device last saw, someone else wrote in between: a conflict prompt offers **load Drive** / **overwrite anyway** instead of silently letting last-write-win.

Details that matter:

- **Timestamps are Drive's own** (RFC 3339 UTC), on both sides of the comparison — device clock skew is irrelevant, and plain string comparison is correct for the fixed-width format.
- **`driveModifiedTime` is recorded on every successful load and save.** On load it's recorded *before* the conflict prompt, so a "keep local" push doesn't immediately re-prompt against its own read.
- **Fail-open when there's no recorded timestamp** (pre-existing installs, first save after connect): the check is skipped and behavior matches the old template. The first successful save records a baseline.
- **Choosing "load Drive" calls `loadFromDrive({ skipConfirm: true })`** — the user already made the choice; re-showing the local-vs-Drive prompt would be a double prompt. The call goes through a `loadFromDriveRef` because `loadFromDrive` (keep-local branch) calls `saveToDrive` and `saveToDrive` (pre-check) calls `loadFromDrive` — a circular `useCallback` dependency that a latest-value ref breaks.
- This is deliberately **not a merge**. Simultaneous editing needs CRDTs or field-level merging, which this stack intentionally avoids. Detect-and-ask covers the sequential case, which is the one that actually happens with one user and two devices.

---

## Version footer (optional block)

**Problem it solves:** after pushing, "did the deploy actually go live?" — GitHub Pages deploys lag by a minute or two, and without a marker you're left comparing behavior by eye. `VersionFooter` asks GitHub's public API for the latest commit touching the deployed file (`/repos/OWNER/REPO/commits?path=index.html&per_page=1`) and renders short-SHA + date at the bottom of Settings.

**The honest caveat:** it proves the newest commit *exists on GitHub* — not that the browser is *running* it. The API call is live, the page may be cached: a stale page happily shows the newest SHA next to old behavior. When the SHA looks right but the app looks wrong, hard-refresh; the footer narrows the question to "is it deployed?" vs "am I cached?", it doesn't answer both.

Fails silently to nothing (renders `null`) when offline or rate-limited — the unauthenticated GitHub API allows 60 requests/hour/IP, which one call per page load fits comfortably. The template ships placeholder `REPO_OWNER`/`REPO_NAME` consts and skips the request entirely until they're set. Delete-if-unwanted, same as the Drive block.

---

## Snapshot before destructive replace

Two operations replace the entire dataset in one move: restoring a backup file from disk, and choosing "load Drive data" in a conflict prompt. Both are *intentional* actions with *unintended* outcomes available — the wrong file, the wrong prompt button. `snapshotData(current)` writes the outgoing data to a rolling `snapshots` key (newest first, last 3, timestamped) immediately before either replace.

Empty datasets are not snapshotted — a snapshot of nothing is noise, and with only 3 slots it would evict a snapshot worth having.

**Clear All Data is deliberately not snapshotted.** Import and conflict-load replace data as a *side effect* of another intent, so the user deserves an undo. Clear's intent *is* deletion — its confirm says "cannot be undone", and keeping a hidden copy would make that a lie (and defeat clearing storage to free space).

**Manual recovery** (DevTools console, substitute your `PFX`):

```js
JSON.parse(localStorage.getItem('app_snapshots'))          // inspect: [{ at, data }, …] newest first
const s = JSON.parse(localStorage.getItem('app_snapshots'))[0];
localStorage.setItem('app_data', JSON.stringify(s.data));
location.reload();
```

Deliberately manual — a restore UI for a safety net that fires rarely isn't worth its surface area in a template. Apps where restores are routine should build one.

---

## Showing numbers

**An empty cell beats a wrong number.**
A wrong figure ends a question; a blank one prompts it. Where a displayed value can come from more than one source, show which — a one-word provenance label next to the number (e.g. "manual" vs. a date) resolves "where did this come from?" in a glance and costs one string.

*When this applies:* any UI showing a computed or user-editable number that could plausibly be missing or overridden.

---

## Anti-patterns (seen across audited repos, avoided here)

**Inline styles with hardcoded colors** (seen in FreezerBox)
`style={{ background: '#0e0f11' }}` makes theming, dark/light toggle, and future design changes impossible without touching every element. Always use `var(--token-name)` and define it in `:root`.

**All styling via inline `style` props** (seen in FreezerBox)
Harder to read, impossible to override with CSS, prevents pseudo-selectors (`:hover`, `:focus`). Use class names with CSS rules; reserve inline styles for truly dynamic values (e.g. a width derived from state).

**Using `localStorage` directly without helpers**
Bypasses the prefix, loses error handling, scatters JSON.parse/stringify throughout the code.

**`dangerouslySetInnerHTML` with user-entered text**
React escapes JSX text automatically — `<p>{item.note}</p>` is XSS-safe no matter what the user typed. That guarantee is void the moment `dangerouslySetInnerHTML` (or a ref + `innerHTML`) is used: the string goes into the DOM as markup, and a note containing `<img src=x onerror=…>` executes. In the vanilla-JS sister project (Løpelogger), exactly this class of missing escaping was a stored XSS with the Drive token in reach — one crafted entry in a synced dataset runs script on every device that loads it. React apps inherit the protection only while nobody bypasses it; the prop's name is the warning. If you need rich text, render a constrained structure (e.g. split on newlines and map to elements), don't inject markup.
