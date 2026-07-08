---
name: bunny-deep-researcher
description: Run Phase 3 (EVIDENCE) on a screened-in bunny — write the four Phase-3 documents (deep_research, marketing_plan, value_ladder, final_evaluation_rubric) from their templates, save each as a draft, and hand off to a human. Use when the user asks to run Phase 3, do deep research, or generate evidence docs for a bunny.
---

# Running Phase 3 (EVIDENCE)

## 1. What this is / when to use

Phase 3 is the **EVIDENCE** phase of the Bunny Trail: a screened-in candidate solution gets its research, go-to-market, and offer-architecture evidence base built out, so a human can decide whether it deserves to advance to Phase 4 (SHAPING).

Use this skill when the bunny is `screened_in` and sitting at **phase 3**, and the human wants Phase-3 documents written or advanced.

This skill **inherits `bunny-operator`'s DNA** — if you haven't internalized that skill, do so first. In particular:
- Read live context before writing anything.
- You only ever produce **drafts**.
- You never bless your own work and never advance the bunny.
- No fabrication — ground every claim in the bunny's real evidence or real research you actually did.

This skill is the Phase-3-specific playbook on top of that foundation: it encodes the proven process for producing the four evidence documents in the right order, with the right sub-agent model, from **server-owned templates**.

## 2. Orient (always first)

Before writing anything:

1. Call **`get_bunny_context({ id })`**. Read:
   - The problem statement, audience, and solution.
   - The **provenance quotes** — this is the evidence base. Ground everything in it.
   - Any original submission/source material carried on the bunny, so Phase 3 research doesn't drift from the original idea.
   - `bunny.currentPhase` — confirm it's phase 3. If not, stop and say so.
   - The `playbook` object: `playbook.missingInOrder` (doc kinds still needed, in dependency order) and `playbook.nextEligible` (the first doc whose prerequisites are all `final` — the one to write next).
2. Call **`get_documents({ bunnyId })`** to read anything already drafted or finalized, so new work is consistent with it.

`playbook.nextEligible` is the live source of truth for what to write next — not a hardcoded list in this skill.

## 3. The loop

While `playbook.nextEligible` is one of the four Phase-3 kinds:

1. Call **`get_template({ kind })`** for that kind.
   - `{ found: true, id, name, phase, outputType, version, body }` → `body` is the exact section structure. Follow it exactly; produce every section in order; never invent section names.
   - `{ found: false, kind, resolvedOutputType }` → normal. Some kinds have no template yet. Proceed with a sensible structure grounded in the bunny's evidence, and say plainly in the draft that no template was found for this kind.
2. Spawn a **sub-agent** (Claude Code Task tool) with the matching prompt from Section 4, filled in with this bunny's context and the template body.
3. The sub-agent calls **`save_document({ bunnyId, kind, phase, markdown, title?, confirmOverwrite?, templateId, templateVersion })`** — pass the template's `id`/`version` back as `templateId`/`templateVersion` for provenance. `confirmOverwrite: true` is used ONLY if a human has explicitly asked to replace an already-`final` document (the save is otherwise refused with `would_overwrite_final`).
4. Re-read context (`get_bunny_context`) to advance to the next eligible kind.

**MODEL RULE — non-negotiable, stated explicitly:** every document sub-agent runs on **Sonnet** (`claude-sonnet-5`), **never Opus**. A prior run burned 100+ Opus agents on this loop — don't repeat it. Set the Task tool's model explicitly to Sonnet on every spawn; do not let it default.

**Order:**
- `deep_research` first — it has no dependencies and everything else reads it.
- `marketing_plan` and `value_ladder` both depend only on `deep_research`, so once it's drafted, they **may run as two parallel Sonnet sub-agents**.
- `final_evaluation_rubric` last — it needs all three.

Treat this order as the expected shape, not gospel — always defer to what `playbook.nextEligible` actually says.

## 4. The four sub-agent prompts

Fill in `{bunny_id}`, `{handle}`, `{product_name}`, and the template `body` before spawning. Each sub-agent runs on **Sonnet**.

### Deep Research

```
You are writing the deep_research document for bunny {bunny_id} ({handle}: "{product_name}").
You are running on Sonnet. Do not attempt to switch models.

STEP 1 — Get the template:
Call get_template({ kind: "deep_research" }).
If found: true, use `body` as the exact section structure. Produce every section it
defines, in order. Do not invent, skip, or reorder sections.
If found: false, note that plainly in the draft and use a sensible research-report
structure grounded in the bunny's evidence instead.

STEP 2 — Read bunny context:
Call get_bunny_context({ id: bunny_id }) if not already provided.
Call get_documents({ bunnyId: bunny_id }) for anything already drafted.
Ground every claim in the bunny's provenance quotes and any original submission
material, plus real research you actually conduct (cite it).

STEP 3 — Write the document:
Follow the template's section structure exactly, using its headings.
Do not invent statistics, competitor user counts, or quotes.
If something is unknown, say so plainly rather than guessing.

STEP 4 — Save:
Call save_document({
  bunnyId: bunny_id, kind: "deep_research", phase: 3, markdown: <full document>,
  templateId: <template id from Step 1>, templateVersion: <template version from Step 1>
}).
Then STOP. Hand off to the human for review — do not call finalize_document or
promote_bunny, and do not proceed to another document yourself.

NON-NEGOTIABLE RULES:
- Never fabricate statistics, quotes, or user counts.
- Never cite competitor user counts (adversarially indefensible).
- This is sensitive content for a real audience — clinical/factual accuracy matters;
  never claim an unearned credential or advisor review.
- save_document only writes a draft. Never finalize, never advance the bunny.
```

### Marketing Plan

```
You are writing the marketing_plan document for bunny {bunny_id} ({handle}: "{product_name}").
You are running on Sonnet. Do not attempt to switch models.

STEP 1 — Get the template:
Call get_template({ kind: "marketing_plan" }).
If found: true, use `body` as the exact section structure. Produce every section it
defines, in order. Do not invent, skip, or reorder sections.
If found: false, note that plainly in the draft and use a sensible marketing-plan
structure grounded in the bunny's evidence instead.

STEP 2 — Read prerequisite documents:
Call get_documents({ bunnyId: bunny_id, kind: "deep_research" }) and read the
finalized (or, if not yet final, draft) deep_research document in full. The
marketing plan must be consistent with its findings.

STEP 3 — Write the document:
Follow the template's section structure exactly. Cover funnel architecture, channel
strategy, copy angles, lead magnet design, and (if applicable) virality mechanisms.
Do not invent engagement rates, follower counts, or platform metrics.

STEP 4 — Save:
Call save_document({
  bunnyId: bunny_id, kind: "marketing_plan", phase: 3, markdown: <full document>,
  templateId: <template id from Step 1>, templateVersion: <template version from Step 1>
}).
Then STOP. Hand off to the human for review — do not call finalize_document or
promote_bunny, and do not proceed to another document yourself.

NON-NEGOTIABLE RULES:
- Never fabricate metrics, testimonials, or user counts.
- Ground every channel/copy claim in the deep_research evidence or stated rationale.
- save_document only writes a draft. Never finalize, never advance the bunny.
```

### Value Ladder

```
You are writing the value_ladder document for bunny {bunny_id} ({handle}: "{product_name}").
You are running on Sonnet. Do not attempt to switch models.

STEP 1 — Get the template:
Call get_template({ kind: "value_ladder" }).
If found: true, use `body` as the exact section structure. Produce every section it
defines, in order. Do not invent, skip, or reorder sections.
If found: false, note that plainly in the draft and use a sensible offer-architecture
structure grounded in the bunny's evidence instead.

STEP 2 — Read prerequisite documents:
Call get_documents({ bunnyId: bunny_id }) and read deep_research and marketing_plan
in full. The value ladder's offer tiers must align with the funnel defined in
marketing_plan.

STEP 3 — Write the document:
Follow the template's section structure exactly. Cover offer tiers, pricing
rationale, guarantee structure, and bonus/continuity design as the template
requires. Ground all pricing in evidence or explicit stated rationale — never
invent a price point without one.

STEP 4 — Save:
Call save_document({
  bunnyId: bunny_id, kind: "value_ladder", phase: 3, markdown: <full document>,
  templateId: <template id from Step 1>, templateVersion: <template version from Step 1>
}).
Then STOP. Hand off to the human for review — do not call finalize_document or
promote_bunny, and do not proceed to another document yourself.

NON-NEGOTIABLE RULES:
- Never invent a price, guarantee term, or bonus without a stated rationale.
- Never claim an unearned credential or advisor review to justify pricing.
- save_document only writes a draft. Never finalize, never advance the bunny.
```

### Final Evaluation Rubric

```
You are writing the final_evaluation_rubric document for bunny {bunny_id} ({handle}: "{product_name}").
You are running on Sonnet. Do not attempt to switch models.

STEP 1 — Get the template:
Call get_template({ kind: "final_evaluation_rubric" }).
If found: true, use `body` as the exact section structure — including the exact
weights and gate definitions it specifies. Do not invent or modify weights.
If found: false, note that plainly and use a sensible weighted-scorecard structure
(dimensions, hard gates, synthesis, verdict, decision notes) instead.

STEP 2 — Read the three prerequisite documents:
Call get_documents({ bunnyId: bunny_id }) and read the FINALIZED deep_research,
marketing_plan, and value_ladder in full. In v1 of this skill there is no
Expert QA Panel output to read — score from the three documents alone.

STEP 3 — Complete the rubric per the template's Parts A–E (or your best-effort
equivalent structure if no template was found):
- Score each weighted dimension, citing specific evidence from the three documents
  for every score.
- Assess every hard gate as PASS or FAIL. A single FAIL means no green light — be
  honest, do not soften a real gate failure.
- Write the narrative synthesis section(s).
- Write the final verdict (e.g. GREEN / AMBER / RED) with the weighted total.
- Write the decision notes — flags, risks, recommendations, deferred items.
- Leave any Human Sign-Off block blank.

STEP 4 — Save:
Call save_document({
  bunnyId: bunny_id, kind: "final_evaluation_rubric", phase: 3, markdown: <full document>,
  templateId: <template id from Step 1>, templateVersion: <template version from Step 1>
}).
Then STOP. Hand off to the human for review and sign-off — do not call
finalize_document, score_bunny, or promote_bunny yourself.

NON-NEGOTIABLE RULES:
- Never fabricate a citation — every dimension score must point to real content in
  one of the three documents.
- Never soften a hard-gate FAIL to make the verdict look better.
- save_document only writes a draft. Never finalize, never advance the bunny.
```

## 5. Human gate

Once all four drafts exist, **STOP**. Tell the human:
- Which documents are drafted and ready for review.
- That they need to review and `finalize_document` each one in Bunny OS.

You (the orchestrating session, and every sub-agent) must **never** call `finalize_document` or `promote_bunny`. Those are human-only, always — even if the token you're holding is technically capable of calling them.

## 6. Scoring

After the human has finalized the documents:

1. Call **`score_bunny({ bunnyId })`**. It's advisory, re-runnable, and never advances anything. It reads finalized docs preferentially, falling back to drafts (marked PROVISIONAL) if some aren't final yet.
2. **Sync-lag is expected, not an error** (Eloise's playbook, Failure Mode 3): right after a human finalizes documents in Bunny OS, `score_bunny` may still return a PROVISIONAL score, or the score may fluctuate slightly between calls, for several minutes while finalization propagates. Do not treat this as a bug and do not aggressively retry — wait roughly 5–10 minutes and re-run once.
3. Present the score/verdict to the human plainly, including if it's still PROVISIONAL. The human decides whether and when to advance the bunny — you never do.

## 7. Resilience

From Eloise's playbook (Failure Modes 1–2), adapted:

- **Mid-output stalls on large documents.** If a document sub-agent stalls mid-write (alive but paused, no error), **resume it** — don't restart from scratch. It will continue from where it left off.
- **Keep documents in separate sub-agents.** Writing all four documents in a single long-running context risks compaction wiping earlier reasoning and increases stall risk. One sub-agent per document keeps each context lean and independently resumable.

## 8. Hard rules

1. **Never fabricate** statistics, quotes, competitor user counts, or sources. If something is unknown, say so plainly.
2. **Never cite competitor user counts** — they're not verifiable and get adversarially refuted.
3. **Clinical/factual accuracy over persuasiveness** for sensitive audiences — never claim an unearned credential, advisor review, or professional endorsement the bunny's evidence doesn't actually support.
4. **`save_document` writes drafts only.** No sub-agent, and no orchestrating session, ever calls `finalize_document` or `promote_bunny`.
5. **Never advance the bunny.** A human finalizes each document and decides on advancement — that is never this skill's job, no matter how confident the draft or the score.

## 9. v2 (deferred) — not in this skill

This v1 skill is **documents-first only**: the four documents plus an advisory score. Eloise's full playbook also describes an **Expert QA Panel**, **5 structured decision points**, and a **revision loop** that incorporates panel + human decisions back into the three documents before the rubric is scored. None of that is implemented here — it's explicitly **v2**.

The `expert-qa-panel` skill already exists and is intended to be wired into this flow in a future version of `bunny-deep-researcher`. Until then, the final_evaluation_rubric sub-agent in Section 4 scores from the three documents alone, with no panel critique as input.
