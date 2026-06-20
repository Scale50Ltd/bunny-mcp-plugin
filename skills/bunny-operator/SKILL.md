---
name: bunny-operator
description: Operate Project Bunny — find, research, and draft documents for "bunnies" (candidate solutions to build) via the Bunny MCP tools. Use when the user asks to work on a bunny, do Phase-3 / deep research on a bunny, look at Project Bunny ideas, or mentions Project Bunny / the Bunny Trail.
---

# Operating Project Bunny

## What Bunny is
Project Bunny is a product-factory pipeline. Upstream it captures real user problems (e.g. app-store reviews), "smelts" them into candidate **solutions to build**, then scores and screens them. **A "bunny" is a candidate solution**, always grounded in real captured evidence — not an abstract idea.

Screened-in bunnies travel an 11-phase trail:

| # | Phase | Status |
|---|-------|--------|
| 0 | INTAKE | upstream/automated |
| 1 | TRIAGE | upstream/automated |
| 2 | SCREENING | upstream/automated |
| 3 | EVIDENCE | **live** — deep research happens here |
| 4 | SHAPING | **live** |
| 5–11 | SPEC, BUILD, HARDENING, LAUNCH, GROWTH, SCALE, EXIT | **sealed** (not built; never pretend otherwise) |

You operate on the **live** phases (3 and 4). The conveyor advances a bunny 3 → 4 when a human blesses its required document.

## Your loop
1. **Read first.** Given a bunny id (or use `find_bunnies` to locate one), call **`get_bunny_context`** — it returns the bunny's problem, audience, solution, rubric, real provenance quotes, its documents, what's still missing, and its timeline. Use **`get_documents`** for full document text.
2. **Do the work** grounded in that context (and real research).
3. **`save_document`** — this writes an **agent-authored draft**. That is the only write you make.
4. **Stop and hand off.** Tell the human it's ready for review. **A human** decides whether to bless it; blessing is what advances the bunny.

## The hard rules (non-negotiable)
1. **Read `get_bunny_context` before doing or writing anything.**
2. **You only write drafts.** `save_document` is your single write.
3. **You never advance a bunny, and you never bless your own work.** Advancing happens only when a *human* approves a document (`finalize_document → final`). That is the human's step — **not part of your loop.** (Technically the tool is reachable with the token, but using it to self-advance violates the process; leave it to the human.)
4. **No fabrication.** Ground every claim in the bunny's real evidence/provenance or in real research you actually did. If something is unknown, say so plainly. Never invent reviews, numbers, competitors, or sources.
5. **There is no fixed document template yet.** The team (Eloise + Xander) is actively iterating on what a great Phase-3 `deep_research` doc looks like. **Follow the human's current direction. Do not invent a rigid structure and present it as the canonical format.**

## Defer to the live system
Do **not** rely on a hardcoded list of tools, document kinds, or per-phase requirements from this skill — those evolve. Always read the **live** answers from the MCP: the available tools and their arguments, and `get_bunny_context`'s `missing` list for "what this bunny still needs." This skill teaches the stable process; the server is the source of truth for specifics.
