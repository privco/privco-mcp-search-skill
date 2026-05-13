# Changelog

All notable changes to the `privco-mcp-search-skill` are recorded here.
The skill teaches Claude how to use the
[`privco-data-mcp`](https://www.npmjs.com/package/privco-data-mcp) MCP
tools effectively; releases track significant updates to the documented
inventory, gotchas, and workflows.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Date format is `YYYY-MM-DD`.

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
  `latestEbitda` are **not** sortable; documented two-stage workaround.
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
- **Top-N-by-valuation two-stage workaround** (the API filters by
  valuation but doesn't sort by it).
- **Recent-funding queries** with caveats about `latestFundingYear` vs
  cumulative `funding.min`.

### Known issues

- `funding_search` and `deal_search` are declared in the MCP package
  ahead of the corresponding backend rollout. Calls to these two tools
  may return HTTP 404 until the backend catches up. Approximate via
  `company_search` with funding/deal filters + per-candidate `vc_deals`
  / `ma_deals` in the meantime.
- Both tools, once live, require the API key to carry the matching
  permission (`funding_search` / `deal_search`, or `everything`) — a key
  with generic auth alone will get HTTP 403.
