---
name: bunny-operator
description: Operate Project Bunny — find, add, screen, research, draft, greenlight, score, and delete "bunnies" (candidate solutions to build) via the Bunny MCP tools. Use when the user asks to work on a bunny, add/create a bunny from an idea, screen a bunny (generate its Phase-2 docs + run the initial rubric), do Phase-3 / deep research on a bunny, greenlight a bunny to Phase 3, score/re-score a bunny, delete/restore a bunny, look at Project Bunny ideas, or mentions Project Bunny / the Bunny Trail.
---

# Operating Project Bunny

## What Bunny is
Project Bunny is a product-factory pipeline. Upstream it captures real user problems (e.g. app-store reviews), "smelts" them into candidate **solutions to build**, then screens and scores them. **A "bunny" is a candidate solution**, always grounded in real captured evidence — not an abstract idea.

Bunnies travel an 11-phase trail. The early phases are now **gated stations**, not passive holding:

| # | Phase | Status |
|---|-------|--------|
| 0 | INTAKE | capture (finder pull or `add_bunny`) |
| 1 | TRIAGE | **the dossier / waiting room** — a bunny rests here after a cheap structural prefilter; nothing is generated yet |
| 2 | SCREENING | **gated** — `screen_bunny` generates the first-pass docs + runs the initial screening rubric |
| 3 | EVIDENCE | **live** — deep research happens here |
| 4 | SHAPING | **live** |
| 5–11 | SPEC, BUILD, HARDENING, LAUNCH, GROWTH, SCALE, EXIT | **sealed** (not built; never pretend otherwise) |

## How a bunny moves (the gated flow)
```
capture → cheap PREFILTER (structural hard-kills)
            ├─ finder pull that trips a hard-kill → auto SCREENED-OUT ($0 spent)
            └─ manual add_bunny → never killed, just FLAGGED
        → PHASE 1 (dossier / waiting room)
        → screen_bunny → PHASE 2: generate 3 first-pass docs + run the INITIAL screening rubric
            ├─ verdict kill              → screened out
            └─ verdict build | research  → PARKED, awaiting a human greenlight
        → greenlight_bunny → PHASE 3 (EVIDENCE) → your deep-research loop
        → human finalizes all Phase-3 docs (incl. required final_evaluation_rubric) → PARKED at Phase 3, awaiting a human greenlight
        → score_bunny(kind:'final') → greenlight_bunny → PHASE 4 (SHAPING)
```
Whether a captured bunny **auto-enters** screening is governed by a global **Auto-screen** valve (default **OFF**) — when off, a human sends bunnies into screening deliberately. You don't toggle that; you just screen the specific bunny the user asks you to.

## Your research loop (the core job — Phase 3)
1. **Read first.** Given a bunny id (or use `find_bunnies` to locate one), call **`get_bunny_context`** — it returns the bunny's problem, audience, solution, rubric scores, real provenance quotes, its documents, what's still missing, and its timeline. Use **`get_documents`** for full document text.
2. **Do the work** grounded in that context (and real research).
3. **`save_document`** — this writes an **agent-authored draft**. That is the only document write you make.
4. **Stop and hand off.** Tell the human it's ready for review. Once the required Phase-3 documents (including `final_evaluation_rubric`) are finalized and a final score exists (`score_bunny({ kind: 'final' })`), **a human** fires `greenlight_bunny` to advance the bunny 3 → 4. You never call `greenlight_bunny` yourself — not even on your own evaluation.

## Managing the bunny lifecycle
Beyond research, these tools drive the trail. The screen/greenlight/promote-style and delete tools act **on the user's behalf** — **ask the user first and only act on an explicit yes** (the server cannot verify consent; the discipline is yours).

- **`add_bunny`** — create a new bunny from a person's submission (a messy blob, a solution idea, or a problem/complaint). An AI structures it into a rich bunny and creates it at stage `captured` (Phase 1). It runs the cheap prefilter but **never auto-kills a manual entry** — a tripped structural hard-kill is recorded as a **flag**, surfaced so a human can decide. If it looks like an existing bunny, the tool **warns** instead of creating — relay that, and only re-call with `confirmDuplicate: true` if the user wants it added anyway.
- **`screen_bunny`** — **ask the user first.** Moves a captured/Phase-1 bunny into **Phase 2**: generates the 3 first-pass documents and runs the **initial screening rubric** over them. A `kill` verdict screens the bunny out; a `build` or `research` verdict leaves it **parked at Phase 2 awaiting a human greenlight**. This is the paid step (it spends AI on documents), so never run it autonomously.
- **`greenlight_bunny`** — **ask the user first.** The **unified** human gate: it advances a bunny past whichever human gate it's currently parked at — **Phase 2 → 3** (EVIDENCE), for a parked `build`/`research` screening verdict, **or Phase 3 → 4** (SHAPING), the **final-evaluation gate**, once the Phase-3 documents (including the required `final_evaluation_rubric`) are finalized and a final score exists. Which gate opens depends on the bunny's current phase — you don't choose, the tool does. Pass `overrideKill: true` only if the user explicitly wants to push a `kill`-verdict bunny through anyway (screening or final). At Phase 3, without an override a `kill` final verdict returns `blocked_kill`; if the preconditions aren't met yet it returns `not_greenlightable` with a reason — `needs_final_docs` or `needs_final_score`. Never greenlight to self-advance your own work, and never greenlight an evaluation you just wrote yourself.
- **`score_bunny`** — a **separate, advisory** rubric run over a bunny's live documents. It takes an optional `kind` (`"screening"` | `"final"`, default `"screening"`). The **screening** score is normally produced by `screen_bunny`; call `score_bunny` to **re-run** it (e.g. after documents change). It is **re-runnable** and **never advances, screens out, or blesses anything** — the verdict is advice for a human, not a gate. Scoring before the docs are finalized returns a **provisional** result. A bunny now carries **two** scores — its screening score and (later) its final-evaluation score — so be clear which one you're reading or running.
- **`delete_bunny`** — **confirm with the user first.** Soft-deletes (archives) a bunny: it disappears from search, the cockpit views, and duplicate detection, but **no data is destroyed**. Fully reversible.
- **`restore_bunny`** — un-archives a previously deleted bunny so it reappears.

## The hard rules (non-negotiable)
1. **Read `get_bunny_context` before doing or writing research on an existing bunny.**
2. **For documents, you only write drafts.** `save_document` is your single document write; it always produces a draft.
3. **You never bless your own work and never self-advance.** Every gate is the human's: `screen_bunny` (enter Phase 2), `greenlight_bunny` (the unified gate — Phase 2 → 3, **or** Phase 3 → 4 at the final-evaluation gate), and `finalize_document → final` are all allowed **only on the user's explicit command — never autonomously, and never to advance your own research**. The token can reach these; the discipline that they're human-decided is yours.
4. **No fabrication.** Ground every claim in the bunny's real evidence/provenance or in real research you actually did. If something is unknown, say so plainly. Never invent reviews, numbers, competitors, or sources.
5. **Phase-3 documents follow templates fetched live from the MCP.** Call the **`get_template`** tool for the doc kind you're about to write and use its returned `body` as the section structure — never a local DOCX, and never a structure you invented. A `found: false` response is normal for some kinds; don't treat it as an error. Templates are server-owned and versioned, so read them fresh each time. For Phase-3 deep-research work specifically, use the **`bunny-deep-researcher`** skill, which builds the full EVIDENCE-phase loop on top of this rule.

## Defer to the live system
Do **not** rely on a hardcoded list of tools, document kinds, or per-phase requirements from this skill — those evolve. Always read the **live** answers from the MCP: the available tools and their arguments, and `get_bunny_context`'s `missing` list for "what this bunny still needs." This skill teaches the stable process; the server is the source of truth for specifics.
