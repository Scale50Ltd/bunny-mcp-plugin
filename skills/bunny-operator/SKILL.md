---
name: bunny-operator
description: Operate Project Bunny — find, add, research, draft, promote, and delete "bunnies" (candidate solutions to build) via the Bunny MCP tools. Use when the user asks to work on a bunny, add/create a bunny from an idea, do Phase-3 / deep research on a bunny, promote a bunny or generate its Phase-2 docs, delete/restore a bunny, look at Project Bunny ideas, or mentions Project Bunny / the Bunny Trail.
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

## Your research loop (the core job)
1. **Read first.** Given a bunny id (or use `find_bunnies` to locate one), call **`get_bunny_context`** — it returns the bunny's problem, audience, solution, rubric, real provenance quotes, its documents, what's still missing, and its timeline. Use **`get_documents`** for full document text.
2. **Do the work** grounded in that context (and real research).
3. **`save_document`** — this writes an **agent-authored draft**. That is the only document write you make.
4. **Stop and hand off.** Tell the human it's ready for review. **A human** decides whether to bless it; blessing is what advances the bunny 3 → 4.

## Managing the bunny lifecycle (create / promote / delete)
Beyond research, four tools manage a bunny's lifecycle. The create/promote/delete tools act **on the user's behalf** — for promote and delete you **ask the user first and only act on an explicit yes** (the server cannot verify consent; the discipline is yours).

- **`add_bunny`** — create a new bunny from a person's submission (a messy blob, a solution idea, or a problem/complaint). An AI structures it into a rich bunny and creates it at stage `captured`. If it looks like an existing bunny, the tool **warns** instead of creating — relay that, and only re-call with `confirmDuplicate: true` if the user wants it added anyway. No fabrication still applies: the AI grounds the bunny in what the user actually wrote.
- **`promote_bunny`** — **ask the user first.** Generates the bunny's solution, rubric-scores it, and (if it screens in) moves it to screening + **phase 3 (EVIDENCE)** and kicks off the Phase-2 documents (which generate asynchronously — fetch them with `get_documents` a moment later). If the rubric would screen the bunny **out**, the tool returns `screened_out_pending` and changes nothing — relay the verdict to the user and only re-call with `force: true` if they confirm. **Never promotes past phase 3.**
- **`delete_bunny`** — **confirm with the user first.** Soft-deletes (archives) a bunny: it disappears from search, the cockpit views, and duplicate detection, but **no data is destroyed**. Fully reversible.
- **`restore_bunny`** — un-archives a previously deleted bunny so it reappears.

## The hard rules (non-negotiable)
1. **Read `get_bunny_context` before doing or writing research on an existing bunny.**
2. **For documents, you only write drafts.** `save_document` is your single document write; it always produces a draft.
3. **You never advance a bunny past phase 3, and you never bless your own work.** The phase 3 → 4 conveyor is **human-only** (`finalize_document → final`) — that is the human's step, never yours, even though the token can reach it. Screening-promotion (`promote_bunny`, which enters phase 3) and `delete_bunny` are allowed, but **only on the user's explicit command — never autonomously, and never to self-advance your own research.**
4. **No fabrication.** Ground every claim in the bunny's real evidence/provenance or in real research you actually did. If something is unknown, say so plainly. Never invent reviews, numbers, competitors, or sources.
5. **Phase-3 documents follow templates fetched live from the MCP.** Call the **`get_template`** tool for the doc kind you're about to write and use its returned `body` as the section structure — never a local DOCX, and never a structure you invented. A `found: false` response is normal for some kinds; don't treat it as an error. Templates are server-owned and versioned, so don't assume today's structure is permanent — read it fresh each time. For Phase-3 deep-research work specifically, use the **`bunny-deep-researcher`** skill, which builds the full four-document EVIDENCE-phase loop on top of this rule.

## Defer to the live system
Do **not** rely on a hardcoded list of tools, document kinds, or per-phase requirements from this skill — those evolve. Always read the **live** answers from the MCP: the available tools and their arguments, and `get_bunny_context`'s `missing` list for "what this bunny still needs." This skill teaches the stable process; the server is the source of truth for specifics.
