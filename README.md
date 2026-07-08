# Bunny MCP Plugin

Orientation + tools for operating **Project Bunny** from Claude Code (and a documented path for Codex). Installing this plugin gives you the **Bunny MCP server** (find / read / draft bunnies) and the **bunny-operator skill** (how to operate it well).

> The plugin contains **no secret**. You supply your Bunny token yourself via an environment variable (below).

## Skills

- **`bunny-operator`** — core orientation: find/add/promote/delete bunnies, the research loop, the hard rules. Read this first.
- **`bunny-deep-researcher`** — Phase-3 (EVIDENCE) playbook: writes the four Phase-3 documents (`deep_research`, `marketing_plan`, `value_ladder`, `final_evaluation_rubric`) from live server templates, one Sonnet sub-agent per document, then runs the Expert QA Panel → human decision gate → revision loop, then hands off to a human for review/finalize and an advisory `score_bunny`.
- **`expert-qa-panel`** — convenes named marketing/business-design experts (Value Ladder & Offer Architecture wing: Brunson, Hormozi, Godin, Kern, Cialdini; Marketing wing: Kern, Berger) to critique a bunny's Phase-3 drafts and produce a Synthesis with top-3 recommendations and Unresolved Questions. Invoked by `bunny-deep-researcher`; can also be run standalone.

## Tools

The MCP server exposes (non-exhaustive — treat the live `tools/list` response as the source of truth):

- **`find_bunnies`**, **`get_bunny_context`**, **`get_documents`** — read tools. `get_bunny_context` also returns a `playbook` object (`missingInOrder`, `nextEligible`) describing what Phase-3 document to write next.
- **`get_template({ kind })`** — fetches a document template. Returns `{ found: true, id, name, phase, outputType, version, body }` on a hit (`body` is the section structure to follow exactly) or `{ found: false, kind, resolvedOutputType }` on a miss (normal — not every kind has a template). Pass the returned `id`/`version` back to `save_document` as `templateId`/`templateVersion`.
- **`save_document({ bunnyId, kind, phase, markdown, title?, confirmOverwrite?, templateId?, templateVersion? })`** — writes a draft only. `templateId`/`templateVersion` record provenance against the template used. If a `final` document of this kind already exists, the save is refused with `would_overwrite_final` unless `confirmOverwrite: true` — only pass that after a human explicitly asks to replace an approved doc.
- **`score_bunny({ bunnyId })`** — advisory scoring, re-runnable, never advances anything.
- **`add_bunny`**, **`promote_bunny`**, **`delete_bunny`**, **`restore_bunny`** — lifecycle tools (see `bunny-operator`).
- **`finalize_document`**, and advancing a bunny past phase 3, are **human-only** — no skill in this plugin calls them.

## Claude Code — install

1. **Set your token** in your shell profile (`~/.zshrc` or `~/.bashrc`), then restart your shell:
   ```bash
   export BUNNY_MCP_TOKEN="<your 64-char Bunny service token>"
   ```
   (Setting it in the shell profile keeps it out of per-command shell history. Do **not** put it in any project file.)

2. **Remove any old manual registration** (if you previously ran `claude mcp add bunny`), so you don't end up with two `bunny` servers (collisions resolve silently to one winner):
   ```bash
   claude mcp remove bunny 2>/dev/null; claude mcp list
   ```
   Confirm `bunny` is no longer listed.

3. **Add the marketplace and install:**
   ```
   /plugin marketplace add Scale50Ltd/bunny-mcp-plugin
   /plugin install bunny@bunny-mcp-plugin
   ```

4. **Restart Claude Code**, then verify exactly one connected server:
   ```bash
   claude mcp list   # expect: bunny  ✔ Connected   (exactly one)
   ```
   (A first-run trust prompt may appear before it connects.)

## Verify your token works (self-test)

A wrong/missing token produces a 401 that can look like "no auth." Test it directly:
```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST \
  https://ads-dashboard-scale50.vercel.app/api/agent/mcp \
  -H "Authorization: Bearer $BUNNY_MCP_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```
- `200` → token is valid.
- `401` → token is wrong or `BUNNY_MCP_TOKEN` is unset. Re-check step 1.

If the `bunny` server shows disconnected/missing in Claude Code, it's almost always the token: confirm `echo $BUNNY_MCP_TOKEN` is the right value in the shell that launched Claude Code.

## Codex CLI

Codex has no plugins or skills; it only reads the MCP server's built-in `instructions`. Add the server to `~/.codex/config.toml`:
```toml
[mcp_servers.bunny]
url = "https://ads-dashboard-scale50.vercel.app/api/agent/mcp"
bearer_token_env_var = "BUNNY_MCP_TOKEN"
```
(Verify these key names against current Codex docs.) Export `BUNNY_MCP_TOKEN` in your shell as above.

## Updating

Bug fixes and playbook changes ship by updating this repo. In Claude Code, re-running the install (or your version's plugin-update flow) pulls the latest skill + MCP config together.

## Security notes

- This is a **shared service token**: there is no per-user revocation. If it leaks, rotate `REPORT_SERVICE_TOKEN` in Vercel — that invalidates **all** operators at once.
- Never commit the token. Never put it in a project `.claude/settings.json`.
- The token grants the bearer the ability to call every Bunny tool, including `finalize_document` (which advances a bunny). By process, **only a human blesses** documents — keep a human in the loop.
