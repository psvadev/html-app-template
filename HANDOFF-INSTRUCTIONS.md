# Handoff doc — instruction

Copy this file into every new project alongside `CLAUDE.md`. It defines how that
project's `HANDOFF.md` must be written and maintained, so all projects document
themselves the same way.

---

Write (or update) HANDOFF.md so that a brand-new Claude Code session with ZERO
context — no memory of this or any prior conversation — can read it and pick up
exactly where we left off. Assume the reader is capable but knows nothing about
this project's history. The test for every line: "would a fresh session make a
mistake or re-do work without this?" If yes, it belongs. If it's derivable from
the code itself, it doesn't.

The doc must capture, in this order:

1. **WHAT THIS IS** — two or three sentences: what the app does, for whom, and the
   architecture in one breath (stack, storage, hosting, integrations).

2. **CURRENT STATE** — what works today, verified how. Include exact deploy/run
   steps and how to test (tools, paths, commands). Note anything currently
   half-done with its next concrete step.

3. **HOW THINGS FIT TOGETHER** — the systems view: data flow, sync/auth flows,
   which functions own which surfaces. Focus on the connections a reader can't
   see by reading one function at a time (e.g. "all charts receive ONE shared
   filtered array, but X and Y also re-render independently — so filtering must
   live inside each function, not at call sites").

4. **QUIRKS & GOTCHAS** — every non-obvious fact that cost time to learn, each with
   the WHY: workarounds for platform behavior, misleading symptoms and their
   real cause (e.g. "footer shows newest commit even on a stale cached page —
   it proves 'latest exists', not 'latest is running'"), naming traps, external
   API behaviors confirmed by real testing rather than docs.

5. **DECISION LOG** — every idea discussed, in three buckets, each entry with the
   reasoning, not just the verdict:
   - **ACCEPTED/SHIPPED:** what shipped, why this shape rather than the
     alternatives considered, and any user-stated principle behind it (quote
     the user's own words for load-bearing principles).
   - **REJECTED:** what was proposed, why it was declined, and whether that's
     permanent ("don't re-propose") or conditional ("revisit if X becomes a
     pain point"). This is what stops a future session from re-pitching a
     settled question.
   - **DEFERRED/PARKED:** what it is, why it's parked (not worth it yet ≠ bad
     idea), the rough shape it should take when built, and the concrete
     UN-PARK TRIGGER — the observable event that means it's time.

   When a parked item ships, don't delete its row — replace it with a short
   ship-note (commit ref + what the parked sketch got wrong, if anything), so
   the reasoning trail survives.

6. **WORKFLOW RULES** — how the human wants to work, stated as rules: commit/push
   policy (e.g. "commit freely, NEVER push without explicit OK"), how decisions
   get made (e.g. "discussion in chat first; plan files record landed decisions,
   they are not the conversation"), verification expectations (e.g. "every
   user-facing change gets a Playwright check before commit"), and which docs
   must be updated when a feature ships.

7. **LEARNING LOG** — lessons about the WORK itself, not the code: scoping lessons
   ("re-verify a parked item's scope against what the user actually wants
   before treating the old sketch as the plan"), debugging lessons ("stale
   cached JS mimics logic bugs — check the deployed version first"), process
   lessons. One line each, with the incident that taught it.

**Style rules:** prefer WHY over WHAT (the code shows what); use exact identifiers,
file paths, commit refs, and dates (absolute, never "yesterday"); quote the
user verbatim where their wording IS the requirement; keep it one file; update
it in the SAME session a feature ships — a handoff written from memory a week
later loses exactly the details that matter.
