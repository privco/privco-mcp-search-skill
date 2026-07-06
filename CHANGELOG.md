# Changelog

All notable changes to the `privco-mcp-search-skill` are recorded here.
The skill teaches Claude how to use the
[`privco-data-mcp`](https://www.npmjs.com/package/privco-data-mcp) MCP
tools effectively; releases track significant updates to the documented
inventory, gotchas, and workflows.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Date format is `YYYY-MM-DD`.

## [1.4.0] — 2026-07-04

Remote-connector release. The PrivCo MCP server is now reachable two ways;
the skill documents both.

> Skill versions no longer track the `privco-data-mcp` npm package version:
> this release documents the hosted remote connector, which is live
> independently of the npm release cadence.

### Added

- **Remote connector documentation** — the hosted PrivCo MCP endpoint at
  `https://mcp.privco.com/mcp` (MCP Streamable HTTP + OAuth 2.1 sign-in
  with a PrivCo account; no API key in config files). `INSTALL.md` is
  restructured around the remote-vs-local choice with per-client steps
  for claude.ai, Claude Desktop, Claude Code, and ChatGPT (developer
  mode), plus a transport-aware troubleshooting table.
- **`references/dashboard_prompt.md`** — PrivCo's house company
  intelligence dashboard template: the standard `match → profile →
  financials → vc_deals/ma_deals → people` pull rendered as a single
  HTML dashboard widget (metric cards, revenue/headcount charts,
  valuation range bar, financial/profile/funding tables), with style
  rules and PrivCo-specific data-mapping notes. `SKILL.md` points to it
  for "present this company" requests.
- `SKILL.md` "Access & auth" section covering both transports and the
  tool-name-prefix note (same 17 tools either way).

### Changed

- The `match` vs `identification` decision rule now leads with **"when in
  doubt, prefer `identification`"** — the fuzzy, multi-signal,
  confidence-scored resolver — reserving `match` for the quick first try
  on clean, recognizable names/websites, and mandating the fall-through
  when `match` returns zero or wrong-looking rows.
- README quick-start prompt now **installs** the skill persistently
  (`git clone` into `~/.claude/skills/`) instead of just reading it into
  one session's context, and prefers the remote connector when no API
  key is provided.

## [1.3.3] — 2026-05-13

Initial public release. Matches the `privco-data-mcp@1.3.3` npm package.

### Tool coverage

- **All 17 tools** registered by `privco-data-mcp@1.3.3` are documented,
  grouped into four stages:
  - **Stage 1 — Entity resolution**: `match`, `identification`.
  - **Stage 2 — Discovery**: `company_search`, `people_search`,
    `funding_search`, `deal_search`, `industry_keyword`, `industry_pics`.
  - **Stage 3 — Enrichment**: `profile`, `general_profile`, `people`,
    `financials`, `revenue_financials`, `vc_deals`, `ma_deals`.
  - **Stage 4 — Contact reveal**: `company_contact`, `investor_contact`
    (each call consumes a contact-view credit on the account).
- The deliberate distinction between strict `match` and fuzzy/scored
  `identification` is called out with a decision-rule table in `SKILL.md`
  and a deeper per-field score-reading guide in
  `references/usage_guide.md`.

### Filter-semantics gotchas documented

These silent-failure modes are the highest-leverage content in the
skill. Each is documented in both `SKILL.md` (terse) and
`references/usage_guide.md` (with examples and "wrong vs right" diff):

- `location.state` requires the full state name (`"New York"`, not `"NY"`).
- `industry` vs `keyword` are distinct vocabularies — broad terms like
  `"Software"` belong in `industry.rawIndustries`, not in
  `keyword.rawKeywords`.
- `inclusionExclusion` is a requirement, not an addition
  (`{"pe": "include"}` means *must be* PE-backed).
- `sorting.field` enum is small — `latestValuation`, `latestRevenue`, and
  `latestEbitda` are **not** sortable; documented two-stage pattern.
- `revenue.includeMissing` quietly changes result composition.
- `keyword.condition` is **required** when the `keyword` filter is used
  (values `"should"` / `"must"`).
- `profileType` path param is lowercase only (`'company'`, `'investor'`,
  `'advisor'`).
- `people_search` returns Company-associated rows by default; pass
  `{"filters": {"investorOnly": true}}` to source Investor-associated
  rows.
- Contact-reveal hashes come from `people_search` row `hash` fields, not
  from `profile.contacts[]`.
- `deal_search.isPeDeal` suppresses `targetsIsPeBacked` /
  `targetsIsVcBacked` / `buyersAllOfType` when set.
- `deal_search` default sort = relevance if `filters.query` is set, else
  `timestamp desc`.
- `deal_search` response `totalInUsd` is the literal string `"--"` when
  missing (not null).
- POST endpoints return `201 Created`, not `200`.

### Workflows documented

- **Named-entity lookup**: `match → profile → vc_deals` (3–4 calls per
  entity for full enrichment).
- **Filter-criteria candidate discovery**: `industry_keyword →
  company_search → profile()` per candidate.
- **Top-N-by-valuation two-stage pattern** (the API filters by
  valuation but doesn't sort by it).
- **Recent-funding queries** with caveats about `latestFundingYear` vs
  cumulative `funding.min`.

### Coming soon

- `funding_search` and `deal_search` are documented for forward
  compatibility and will be released as expanded features. Until then,
  approximate via `company_search` with funding/deal filters +
  per-candidate `vc_deals` / `ma_deals`.
- Once released, both tools will require the API key to carry the
  matching permission (`funding_search` / `deal_search`, or
  `everything`).
