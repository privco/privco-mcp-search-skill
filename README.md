# privco-mcp-search-skill

A [Claude](https://claude.com/claude-code) **skill** that teaches Claude how to use the PrivCo MCP tools (`mcp__privco-data-mcp__*`) effectively — without trial-and-error on the non-obvious filter semantics, response-shape gaps, and multi-tool workflows.

Drop this skill into Claude Code or Claude Desktop, register the [`privco-data-mcp`](https://www.npmjs.com/package/privco-data-mcp) MCP server with your own PrivCo API key, and Claude becomes immediately effective at company / people / funding queries against PrivCo data.

## What it does

When activated, the skill briefs Claude on:

- **All 17 PrivCo MCP tools** (as of `privco-data-mcp` v1.3.3) grouped by stage (entity resolution → discovery → enrichment → contact reveal) so Claude reaches for the right tool, including the deliberate distinction between strict `match` and fuzzy/scored `identification`.
- **The filter-semantics gotchas** that silently break queries — full state name required, `industry` vs `keyword` vocabulary split, `inclusionExclusion` semantics, the small `sorting.field` enum, `revenue.includeMissing` behavior, `keyword.condition` requirement, lowercase `profileType` path param, `people_search` Company-only default, `deal_search.isPeDeal` precedence trap, and more.
- **Summary-row gaps** — which fields `company_search` omits and require a follow-up `profile()` call.
- **Standard workflows** — named-entity lookup, criteria-driven discovery, "top N by valuation" two-stage workaround, recent-funding queries.
- **Field-shape gotchas** — VARCHAR dollar strings, split-VARCHAR dates, deduping aggregated investor arrays.

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

### 1. Install the MCP server

```bash
npm install -g privco-data-mcp     # Node.js ≥ 18 required
```

Or use `npx -y privco-data-mcp@latest` in your config without installing globally.

### 2. Install this skill

Clone or download into your Claude skills directory:

```bash
git clone https://github.com/privco/privco-mcp-search-skill.git \
  ~/.claude/skills/privco-mcp-search-skill
```

Or download the latest zipped release from the [Releases page](https://github.com/privco/privco-mcp-search-skill/releases) and unzip into `~/.claude/skills/`.

### 3. Register the MCP server with your client

You need a **PrivCo API key** — contact <support@privco.com>. Add it to your MCP client config in the `env` block. **Never commit your key.**

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

For full setup (Claude Desktop, npx alternative, troubleshooting), see [`INSTALL.md`](./INSTALL.md).

### 4. Try it

> "Look up Stripe in PrivCo and give me the latest valuation."

Claude will call `mcp__privco-data-mcp__match`, then `profile` / `vc_deals`, and answer.

## Repository layout

```
privco-mcp-search-skill/
├── SKILL.md                  # the skill itself — auto-loaded
├── INSTALL.md                # full setup guide
├── references/
│   └── usage_guide.md        # deep per-tool reference, all gotchas, workflows
├── CHANGELOG.md              # per-version changes
├── README.md                 # this file
├── LICENSE                   # MIT
└── .gitignore
```

## Updating

The skill mirrors knowledge discovered while using PrivCo's API in real-world queries. If you find a new gotcha or a missing workflow, open an issue or PR.

When the upstream [`privco-data-mcp`](https://www.npmjs.com/package/privco-data-mcp) npm package adds new tools or changes input schemas, both `SKILL.md` and `references/usage_guide.md` need updating.

## License

[MIT](./LICENSE) — use it however you like.

## Links

- PrivCo MCP server (npm): <https://www.npmjs.com/package/privco-data-mcp>
- PrivCo API documentation: <https://www.privco.com/api-documentation>
- PrivCo support: <support@privco.com>
- Claude Code: <https://claude.com/claude-code>
- Model Context Protocol: <https://modelcontextprotocol.io>
