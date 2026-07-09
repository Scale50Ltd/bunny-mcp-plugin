---
name: bunny-deep-researcher
description: Run Phase 3 (EVIDENCE) on a screened-in bunny — write the three deliverable Phase-3 documents (deep_research, marketing_plan, value_ladder) from their templates, run the Expert QA Panel against them, present the panel's Unresolved Questions at a human decision gate, revise all three documents per the panel + human decisions, then score against the final_evaluation_rubric and hand off to a human. Use when the user asks to run Phase 3, do deep research, run the expert panel, or generate/revise evidence docs for a bunny.
---

# Running Phase 3 (EVIDENCE)

## 1. What this is / when to use

Phase 3 is the **EVIDENCE** phase of the Bunny Trail: a screened-in candidate solution gets its research, go-to-market, and offer-architecture evidence base built out, so a human can decide whether it deserves to advance to Phase 4 (SHAPING).

Use this skill when the bunny is `screened_in` and sitting at **phase 3**, and the human wants Phase-3 documents written, QA'd, revised, or scored.

This skill **inherits `bunny-operator`'s DNA** — if you haven't internalized that skill, do so first. In particular:
- Read live context before writing anything.
- You only ever produce **drafts**.
- You never bless your own work and never advance the bunny.
- No fabrication — ground every claim in the bunny's real evidence or real research you actually did.

This skill is the Phase-3-specific playbook on top of that foundation. It encodes the proven process, reverse-engineered from a real Phase-3 session (Eloise's playbook), for:
1. producing the three deliverable evidence documents (`deep_research`, `marketing_plan`, `value_ladder`) from **server-owned templates**,
2. running the **Expert QA Panel** (`expert-qa-panel` skill) against them,
3. presenting the panel's **Unresolved Questions** at a **human decision gate**,
4. running a **revision loop** that folds the panel's top-3 recommendations and the human's decisions back into all three documents, and
5. only then scoring against `final_evaluation_rubric` — which itself now reads the panel output too.

## 2. Orient (always first)

Before writing anything:

1. Call **`get_bunny_context({ id })`**. Read:
   - The problem statement, audience, and solution.
   - The **provenance quotes** — this is the evidence base. Ground everything in it.
   - Any original submission/source material carried on the bunny, so Phase 3 research doesn't drift from the original idea.
   - `bunny.currentPhase` — confirm it's phase 3. If not, stop and say so.
   - The `playbook` object: `playbook.missingInOrder` (doc kinds still needed, in dependency order) and `playbook.nextEligible` (the first doc whose prerequisites are all `final` — the one to write next).
2. Call **`get_documents({ bunnyId })`** to read anything already drafted or finalized, so new work is consistent with it.

`playbook.nextEligible` is the live source of truth for what to write next — not a hardcoded list in this skill. Note that `playbook.missingInOrder`/`nextEligible` track the four *templated* document kinds (`deep_research`, `marketing_plan`, `value_ladder`, `final_evaluation_rubric`); the Expert QA Panel, the human decision gate, and the revision loop are orchestration steps this skill inserts around that sequence, not separate `nextEligible` kinds.

## 3. The loop

While `playbook.nextEligible` is one of the four Phase-3 kinds:

1. Call **`get_template({ kind })`** for that kind.
   - `{ found: true, id, name, phase, outputType, version, body }` → `body` is the exact section structure. Follow it exactly; produce every section in order; never invent section names.
   - `{ found: false, kind, resolvedOutputType }` → normal. Some kinds have no template yet. Proceed with a sensible structure grounded in the bunny's evidence, and say plainly in the draft that no template was found for this kind.
2. Spawn a **sub-agent** (Claude Code Task tool) with the matching prompt from Section 4, filled in with this bunny's context and the template body.
3. The sub-agent calls **`save_document({ bunnyId, kind, phase, markdown, title?, confirmOverwrite?, templateId, templateVersion })`** — pass the template's `id`/`version` back as `templateId`/`templateVersion` for provenance. `confirmOverwrite: true` is used ONLY if a human has explicitly asked to replace an already-`final` document (the save is otherwise refused with `would_overwrite_final`).
4. Re-read context (`get_bunny_context`) to advance to the next eligible kind.

**MODEL RULE — non-negotiable, stated explicitly:** every sub-agent in this entire skill — document writers, the Expert QA Panel, the revision agents, the rubric scorer — runs on **Sonnet** (`claude-sonnet-5`), **never Opus**. A prior run burned 100+ Opus agents on this loop — don't repeat it. Set the Task tool's model explicitly to Sonnet on every spawn; do not let it default.

**Order:**
- `deep_research` first — it has no dependencies and everything else reads it.
- `marketing_plan` and `value_ladder` both depend only on `deep_research`, so once it's drafted, they **may run as two parallel Sonnet sub-agents**.
- Once all three (`deep_research`, `marketing_plan`, `value_ladder`) are drafted: run the **Expert QA Panel** (Section 5), present the **human decision gate** (Section 6), then run the **revision agents** (Section 7) that fold the panel's and the human's decisions back into all three documents.
- `final_evaluation_rubric` last (Section 8) — it needs all three revised documents, plus the Expert QA Panel output.

Treat this order as the expected shape, not gospel — always defer to what `playbook.nextEligible` actually says.

## 4. The three deliverable sub-agent prompts

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
greenlight_bunny, and do not proceed to another document yourself.

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
greenlight_bunny, and do not proceed to another document yourself.

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
greenlight_bunny, and do not proceed to another document yourself.

NON-NEGOTIABLE RULES:
- Never invent a price, guarantee term, or bonus without a stated rationale.
- Never claim an unearned credential or advisor review to justify pricing.
- save_document only writes a draft. Never finalize, never advance the bunny.
```

## 5. Expert QA Panel

Runs once all three deliverable drafts (`deep_research`, `marketing_plan`, `value_ladder`) exist — **before** `final_evaluation_rubric`. This stage produces an internal artifact, not one of the four templated documents; there is no `get_template` call for it.

Spawn a single Sonnet sub-agent with this prompt:

```
You are running the Expert QA Panel for bunny {bunny_id} ({handle}: "{product_name}").
You are running on Sonnet. Do not attempt to switch models.

STEP 1 — Read the three drafts:
Call get_documents({ bunnyId: bunny_id }) and read the current deep_research,
marketing_plan, and value_ladder drafts in full.

STEP 2 — Invoke the expert-qa-panel skill:
Run BOTH wings:
- Wing 1 (Value Ladder & Offer Architecture): Brunson, Hormozi, Godin, Kern, Cialdini
- Wing 2 (Marketing): Kern (marketing lens), Berger
For each expert, write a 3–8 bullet block in first person, grounded only in the
bunny's actual evidence and the three drafts. Do not invent market facts.

STEP 3 — Write the Synthesis section:
1. Points of agreement (high-confidence recommendations)
2. Productive tensions (name each, propose a resolution)
3. Top 3 recommended changes (ranked by impact, specific, bunny-grounded)
4. Unresolved Questions — anything that needs a human decision or more research
   before revision can proceed

STEP 4 — Save immediately:
Call save_document({
  bunnyId: bunny_id, kind: "expert_qa_panel_output", phase: 3,
  markdown: <full panel output — every expert block plus the Synthesis>
}).
Save this BEFORE presenting anything to the human. This is an internal artifact
(drafts-only, same as every other document this skill produces), but persisting
it immediately protects the panel's work from being lost if the session's
context later compacts — a real failure mode Eloise's playbook documents.

STEP 5 — STOP:
Present the Synthesis's Unresolved Questions to the human. This is the decision
gate (Section 6). Do not proceed to revision, do not call finalize_document, and
do not advance the bunny.

NON-NEGOTIABLE RULES:
- Ground every expert observation in the bunny's actual evidence or the three
  drafts. Do not invent market facts.
- save_document only writes a draft. Never finalize, never advance the bunny.
```

## 6. Human decision gate

The panel's Unresolved Questions are answered by a human — never invented or defaulted by an agent without the table below saying so explicitly. From Eloise's playbook (Section 4: Decision Points), the recurring decision types are:

| Decision | Default handling | Who decides |
|---|---|---|
| A deferred rung of the value ladder (e.g. a high-ticket Rung 4) | **Defer by default** — launch without it, add later after validation | Pre-settable; human can opt in earlier |
| Hashtag / channel specifics | **Defer by default** — leave as a placeholder for the human to pin closer to launch | Pre-settable |
| Outside expert / self-advocacy reviewer | Flag it as a **recommended** review gate | Human decides whether to commission it |
| **Named mechanism** (the product's "how it works" name, e.g. Eloise's example "The Between-Sessions Plan") | **HUMAN-ONLY.** The agent must NOT invent it — this is a creative/brand decision. Prompt the human for it. | Human, always |
| **Credential / advisor claims** (e.g. "OT-reviewed") | **Default: REMOVE.** Never claim an unearned credential. Reframe as "grounded in published research" (or the domain-equivalent) unless the bunny's evidence explicitly confirms a real advisor/credential exists. This is a trust/accuracy risk, especially for a vulnerable audience. | Pre-settable default; only overridden with confirmed evidence |

Wait for the human's answer to every Unresolved Question before spawning the revision agents. For each decision, record: the question, the decision made, and which sections/documents it's expected to touch.

## 7. Revision agents (×3, Sonnet, may run in parallel)

Once the human has answered the decision gate, spawn **one revision sub-agent per deliverable document** (`deep_research`, `marketing_plan`, `value_ladder`) — they operate on different documents, so they **may run in parallel**.

```
You are revising the {document_type} document for bunny {bunny_id} ({handle}: "{product_name}").
You are running on Sonnet. Do not attempt to switch models.

STEP 1 — Read the current draft:
Call get_documents({ bunnyId: bunny_id, kind: "{document_type}" }) and read the
current draft in full.

STEP 2 — Apply the changes:
[List every one of the QA panel's top-3 recommendations and every human decision
from the decision gate here, explicitly. Do not summarize or paraphrase loosely —
list them as concrete instructions.]

CROSS-DOCUMENT SCOPE RULE — apply this explicitly:
Some decisions (credential removal, the named mechanism, risk-reversal framing)
are not a single find-and-replace in one paragraph. They can recur in hard
conditions, KILL triggers, objection handling, credibility positioning, build
gates, copy guidance, and risk rows — across ALL THREE documents, not just the
one you're editing. Check every section of {document_type} for every affected
mention before saving.

STEP 3 — Save the revision:
Call save_document({
  bunnyId: bunny_id, kind: "{document_type}", phase: 3, markdown: <full revised document>
}).
Then STOP. Hand off to the human for re-review — do not call finalize_document
or greenlight_bunny, and do not proceed to another document yourself.

NON-NEGOTIABLE RULES:
- Do not fabricate any new statistic, quote, or user count while revising.
- Do not add an unearned credential under any circumstance.
- Do not change the document's template section structure — only the content
  within it.
- save_document only writes a draft. Never finalize, never advance the bunny.
```

**Overwrite note:** all three documents are still **drafts** during revision, so re-saving is frictionless — this call needs no `confirmOverwrite`. If a document was already `final` (a human had approved it before the panel ran) and now needs revision, the **human** un-finalizes it in Bunny OS first, so it becomes a draft again. The revision agent never passes `confirmOverwrite: true` on its own initiative — that flag exists only for a human's explicit instruction to replace an already-approved document, and this stage doesn't have one.

## 8. Final Evaluation Rubric

```
You are writing the final_evaluation_rubric document for bunny {bunny_id} ({handle}: "{product_name}").
You are running on Sonnet. Do not attempt to switch models.

STEP 1 — Get the template:
Call get_template({ kind: "final_evaluation_rubric" }).
If found: true, use `body` as the exact section structure — including the exact
weights and gate definitions it specifies. Do not invent or modify weights.
If found: false, note that plainly and use a sensible weighted-scorecard structure
(dimensions, hard gates, synthesis, verdict, decision notes) instead.

STEP 2 — Read the prerequisite documents:
Call get_documents({ bunnyId: bunny_id }) and read the FINALIZED (post-revision)
deep_research, marketing_plan, and value_ladder in full. Also read
expert_qa_panel_output — use its Synthesis (points of agreement, tensions, top-3
recommendations) to inform any Panel Offer Recommendation (or similarly named)
section the template defines, and to sanity-check whether the revision agents
actually addressed the panel's top-3 recommendations.

STEP 3 — Complete the rubric per the template's Parts A–E (or your best-effort
equivalent structure if no template was found):
- Score each weighted dimension, citing specific evidence from the three
  documents (and, where relevant, expert_qa_panel_output) for every score.
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
finalize_document, score_bunny, or greenlight_bunny yourself.

NON-NEGOTIABLE RULES:
- Never fabricate a citation — every dimension score must point to real content in
  one of the three documents or expert_qa_panel_output.
- Never soften a hard-gate FAIL to make the verdict look better.
- save_document only writes a draft. Never finalize, never advance the bunny.
```

## 9. Human gate

Once all four drafts — the three revised deliverable documents plus `final_evaluation_rubric` — exist, **STOP**. Tell the human:
- Which documents are drafted and ready for review (note that the three deliverable documents carry the panel's and the human's own revisions).
- That `final_evaluation_rubric` is now a **required** Phase-3 document — it, along with the other three, must be `final` before the Phase 3 → 4 gate can open.
- That they need to review and `finalize_document` each one in Bunny OS.

You (the orchestrating session, and every sub-agent) must **never** call `finalize_document` or `greenlight_bunny`. Those are human-only, always — even if the token you're holding is technically capable of calling them.

## 10. Scoring

After the human has finalized the documents — including `final_evaluation_rubric`:

1. Call **`score_bunny({ bunnyId, kind: 'final' })`**. This is the Phase-3 **final-evaluation** score, distinct from the Phase-2 screening score, and it's what the Phase 3 → 4 gate reads. It's advisory, re-runnable, and never advances anything. It reads finalized docs preferentially, falling back to drafts (marked PROVISIONAL) if some aren't final yet.
2. **Sync-lag is expected, not an error** (Eloise's playbook, Failure Mode 3): right after a human finalizes documents in Bunny OS, `score_bunny` may still return a PROVISIONAL score, or the score may fluctuate slightly between calls, for several minutes while finalization propagates. Do not treat this as a bug and do not aggressively retry — wait roughly 5–10 minutes and re-run once.
3. Present the score/verdict to the human plainly, including if it's still PROVISIONAL. The human decides whether and when to advance the bunny — you never do.
4. **Then STOP and hand off — do not call `greenlight_bunny` yourself.** Once `final_evaluation_rubric` is finalized and the final score exists, the bunny is eligible for the Phase 3 → 4 gate — but firing `greenlight_bunny` is always a human's (or a human's own operator session's) action, never the agent that just wrote the research, ran the panel, or scored the evaluation. You do not bless your own work, and that includes not greenlighting your own final evaluation.

## 11. Resilience

From Eloise's playbook (Failure Modes 1–2), adapted:

- **Mid-output stalls on large documents.** If a document sub-agent (or the QA panel, or a revision agent) stalls mid-write (alive but paused, no error), **resume it** — don't restart from scratch. It will continue from where it left off.
- **Keep documents in separate sub-agents.** Writing multiple large documents in a single long-running context risks compaction wiping earlier reasoning and increases stall risk. One sub-agent per document (and one for the panel) keeps each context lean and independently resumable.
- **Persist the panel output immediately.** This is why Section 5, Step 4 calls `save_document` for `expert_qa_panel_output` before presenting anything to the human — if context compacts between the panel finishing and the revision agents running, the panel's synthesis and top-3 recommendations are still retrievable via `get_documents` instead of being reconstructed from memory.

## 12. Hard rules

1. **Never fabricate** statistics, quotes, competitor user counts, or sources. If something is unknown, say so plainly.
2. **Never cite competitor user counts** — they're not verifiable and get adversarially refuted.
3. **Clinical/factual accuracy over persuasiveness** for sensitive audiences — never claim an unearned credential, advisor review, or professional endorsement the bunny's evidence doesn't actually support. Default to removing such claims (Section 6).
4. **`save_document` writes drafts only.** No sub-agent, and no orchestrating session, ever calls `finalize_document` or `greenlight_bunny`.
5. **Never advance the bunny.** A human finalizes each document and decides on advancement — that is never this skill's job, no matter how confident the draft, the panel, or the score.
6. **The named mechanism is never agent-invented.** If the Expert QA Panel flags a missing or weak named mechanism, that Unresolved Question goes to the human decision gate — an agent does not name it unilaterally.
7. **`confirmOverwrite` is a human-authorized action only.** No sub-agent in this skill — including the revision agents — passes `confirmOverwrite: true` on its own judgment.
8. **Never greenlight the evaluation you just wrote.** After `score_bunny({ kind: 'final' })` returns — even a clean, confident verdict — this skill never calls `greenlight_bunny`. The Phase 3 → 4 gate exists precisely so a human (not the agent that authored the research, the panel critique, or the final score) decides whether the evaluation holds up. Stop and hand off; do not self-approve.

## 13. v2 — implemented

The Expert QA Panel, the human decision gate, and the revision loop (Sections 5–7) described above are now part of this skill — they are no longer deferred. The `expert-qa-panel` skill is wired into this flow, `expert_qa_panel_output` is saved as a real document kind, and `final_evaluation_rubric` (Section 8) reads it.

What's still genuinely deferred, not built here:
- An autonomous remote runner that executes this whole sequence (drafts → panel → decision gate → revision → rubric → score) without a human present to answer the decision gate — the decision gate in Section 6 is designed to *require* a human turn, by design, and that isn't going away; "deferred" here means server-side scheduling/automation around invoking this skill, not the decision gate itself.
- Any additional expert-panel wings beyond the two `expert-qa-panel` already defines.
