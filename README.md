# privco-mcp-search-skill

A [Claude](https://claude.com/claude-code) **skill** that teaches Claude how to use the PrivCo MCP tools (`mcp__privco-data-mcp__*`) effectively — without trial-and-error on the non-obvious filter semantics, response-shape gaps, and multi-tool workflows.

Drop this skill into Claude Code or Claude Desktop, connect the PrivCo MCP server — either the **hosted remote connector** at `https://mcp.privco.com/mcp` (OAuth, nothing to install) or the **local [`privco-data-mcp`](https://www.npmjs.com/package/privco-data-mcp) npm server** with your own PrivCo API key — and Claude becomes immediately effective at company / people / funding queries against PrivCo data.

## Two ways to connect

| | Remote connector (recommended) | Local npm server |
|---|---|---|
| Endpoint | `https://mcp.privco.com/mcp` | runs on your machine via `npx`/npm |
| Auth | OAuth — sign in with your PrivCo account | `PRIVCO_API_KEY` env var |
| Clients | claude.ai, Claude Desktop, Claude Code, ChatGPT | Claude Code, Claude Desktop, any stdio client |

Both expose the same 17 tools. Full steps for each: [`INSTALL.md`](./INSTALL.md).

## Quickest start — hand it to your AI agent

The skill is self-teaching: it carries its own setup steps, tool inventory, and filter gotchas. In Claude Code, paste this prompt so the skill is **installed persistently** (not just read once):

```
Install the PrivCo Claude skill so it's available in future sessions:
git clone https://github.com/privco/privco-mcp-search-skill ~/.claude/skills/privco-mcp-search-skill

Then read its INSTALL.md and connect the PrivCo MCP server — use the remote
connector (https://mcp.privco.com/mcp) unless I give you an API key for the
local npm server. Tell me what you need from me.
```

(If the skill folder already exists from an earlier run, update it with `git -C ~/.claude/skills/privco-mcp-search-skill pull` instead of cloning.)

The agent will clone the skill into your skills directory (so it auto-activates in future sessions), set up the MCP connection, and tell you exactly what credentials go where. Prefer to do it by hand? Follow the [Install](#install) steps below.

## What it does

When activated, the skill briefs Claude on:

- **All 17 PrivCo MCP tools** grouped by stage (entity resolution → discovery → enrichment → contact reveal) so Claude reaches for the right tool, including the deliberate distinction between strict `match` and fuzzy/scored `identification`.
- **The filter-semantics gotchas** that silently break queries — full state name required, `industry` vs `keyword` vocabulary split, `inclusionExclusion` semantics, the small `sorting.field` enum, `revenue.includeMissing` behavior, `keyword.condition` requirement, lowercase `profileType` path param, `people_search` Company-only default, `deal_search.isPeDeal` precedence trap, and more.
- **Summary-row gaps** — which fields `company_search` omits and require a follow-up `profile()` call.
- **Standard workflows** — named-entity lookup, criteria-driven discovery, "top N by valuation" two-stage pattern, recent-funding queries.
- **Field-shape gotchas** — VARCHAR dollar strings, split-VARCHAR dates, deduping aggregated investor arrays.
- **Guided workflow prompts** — the skill points at the MCP server's built-in prompts (`company_dashboard`, `research_company`, `discover_companies_by_criteria`, `top_companies_by_valuation`, `recent_funding_query`) and the fetchable `privco://docs/*` reference resources, rather than bundling its own copies. `company_dashboard` renders a company's data as a single HTML intelligence dashboard (metric cards, revenue/headcount charts, valuation range, financial tables) in PrivCo's house style.

## Auto-activation triggers

The skill auto-activates when you say things like:

- "find companies that…"
- "search PrivCo for…"
- "who are the X companies in Y"
- "look up &lt;company&gt; in PrivCo"
- "valuations for…"
- "build a fixture / dataset of companies matching…"

You can also invoke it explicitly with `/privco-mcp-search-skill` in some clients.

## Install

### 1. Connect the MCP server

**Remote (recommended — no install, no key file):** add a custom connector pointing at `https://mcp.privco.com/mcp` and sign in with your PrivCo account:

- **claude.ai / Claude Desktop:** Settings → Connectors → Add custom connector → URL `https://mcp.privco.com/mcp`, leave OAuth Client ID/Secret blank.
- **Claude Code:** `claude mcp add --transport http privco-data-mcp https://mcp.privco.com/mcp`, then `/mcp` to sign in.
- **ChatGPT:** enable Developer mode (Settings → Apps & Connectors → Advanced settings), then create an app with the same URL and OAuth.

Your PrivCo account needs API access enabled — contact <support@privco.com>.

**Local (npm, API-key auth):**

```bash
npm install -g privco-data-mcp     # Node.js ≥ 18 required
```

Or use `npx -y privco-data-mcp@latest` in your config without installing globally. See [`INSTALL.md`](./INSTALL.md) for the config-file details.

### 2. Install this skill

Clone or download into your Claude skills directory:

```bash
git clone https://github.com/privco/privco-mcp-search-skill.git \
  ~/.claude/skills/privco-mcp-search-skill
```

Or download the latest zipped release from the [Releases page](https://github.com/privco/privco-mcp-search-skill/releases) and unzip into `~/.claude/skills/`.

### 3. (Local path only) Register the server with your API key

If you chose the local npm server, add it to your MCP client config with your **PrivCo API key** in the `env` block. **Never commit your key.**

Claude Code (`~/.claude.json` or project-local `.mcp.json`):

```json
{
  "mcpServers": {
    "privco-data-mcp": {
      "command": "privco-data-mcp",
      "args": [],
      "env": {
        "PRIVCO_API_KEY": "your_api_key_here"
      }
    }
  }
}
```

Restart Claude Code, then `claude mcp list` should show `privco-data-mcp`.

For full setup (Claude Desktop, npx alternative, troubleshooting for both remote and local), see [`INSTALL.md`](./INSTALL.md).

### 4. Try it

> "Look up Stripe in PrivCo and give me the latest valuation."

Claude will call `mcp__privco-data-mcp__match`, then `profile` / `vc_deals`, and answer.

## Repository layout

```
privco-mcp-search-skill/
├── SKILL.md                  # the skill itself — auto-loaded
├── INSTALL.md                # full setup guide (remote connector + local npm)
├── CHANGELOG.md              # per-version changes
├── README.md                 # this file
├── LICENSE                   # MIT
└── .gitignore
```

The deep per-tool reference and the guided workflow prompts (including the
company dashboard) now live in the `privco-data-mcp` server's own contract —
the `privco://docs/overview` + `privco://docs/usage-guide` resources and the
built-in prompts — so they stay in lockstep with the tools and reach every
MCP client, not just skill-aware ones. `SKILL.md` points at them.

## Updating

The skill mirrors knowledge discovered while using PrivCo's API in real-world queries. If you find a new gotcha or a missing workflow, open an issue or PR.

When the upstream [`privco-data-mcp`](https://www.npmjs.com/package/privco-data-mcp) MCP server adds new tools or changes input schemas, update `SKILL.md`; the deep per-tool reference and workflow prompts live in the server's own contract (`privco://docs/*` resources + built-in prompts), so they travel with the server release.

## License

[MIT](./LICENSE) — use it however you like.

## Links

- PrivCo remote MCP connector: `https://mcp.privco.com/mcp`
- PrivCo MCP server (npm): <https://www.npmjs.com/package/privco-data-mcp>
- PrivCo API documentation: <https://www.privco.com/api-documentation>
- PrivCo support: <support@privco.com>
- Claude Code: <https://claude.com/claude-code>
- Model Context Protocol: <https://modelcontextprotocol.io>
