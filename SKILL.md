---
name: privco-mcp-search-skill
description: >
  Use the PrivCo MCP tools (`mcp__privco-data-mcp__*`) correctly for company /
  people / funding queries. Covers the non-obvious filter semantics (full state
  name, industry vs keyword, sorting enum, summary-row gaps, includeMissing),
  the standard match → profile → vc_deals workflow, and the two-stage pattern
  for "top N by valuation". Activate when the user wants to search/filter
  PrivCo data, look up a specific company, or build a multi-entity dataset.
  Trigger phrases: "find companies that…", "search PrivCo for…", "who are the
  X companies in Y", "look up <company> in PrivCo", "valuations for…", "build
  a fixture / dataset of companies matching…".
---

# PrivCo MCP Search

Setup / access (remote connector or local npm server): see `INSTALL.md` in this skill folder.
Full per-tool field reference: see `references/usage_guide.md` in this skill folder.
Dashboard presentation template: see `references/dashboard_prompt.md` in this skill folder.

The bullets below are the high-value-per-line distillation — load them once,
do not re-derive by trial and error.

## Tool inventory (17 tools)

All tools are exposed under the `mcp__<server-name>__*` namespace once the
MCP server is registered — `mcp__privco-data-mcp__*` in the standard setup;
a remote connector carries whatever name it was registered under (see
"Access & auth" below and `INSTALL.md`).

> **Coming soon as expanded features:** `funding_search` and `deal_search`
> are documented below for forward compatibility but are not yet generally
> available. Until they're released, approximate via `company_search` +
> per-candidate `vc_deals` / `ma_deals`. Once released, calls will require
> the API key to carry the `funding_search` / `deal_search` permission.

**Stage 1 — entity resolution (name/details → profile_id)**

These two tools look similar but differ in **strictness**, **inputs accepted**,
and **whether they tell you how confident the match is**. Pick deliberately.

- `match` — **strict, exact-ish matching** on company name and/or website
  only. Returns up to 20 ranked candidates as a flat list with no confidence
  score. Designed for: *"the user gave me a recognizable company name, give me
  the canonical row(s)."* Because the criteria are intentionally strict, it
  can **silently miss results when the input is incomplete or differs even
  slightly from the source data** (e.g., "Stripe" vs "Stripe, Inc.",
  "stripe.io" vs "stripe.com"). First row is usually correct for unique
  brands; common-word names need disambiguation by `profile_type`,
  `profile_classification`, `industry`. Pass `includeCityState: true` to
  surface HQ city/state in the result rows for co-located collisions.
  Inputs: `name`, `website`, `includeCityState`. GET endpoint, response shape:
  `{data: [{name, profile_id, profile_type, profile_classification, permalink, location, industry, website[]}], meta}`.

- `identification` — **flexible, fuzzy / tolerant** matching that accepts
  many partial inputs (DBA, URL, address, country/state/city/zip, phone,
  industry, Facebook/LinkedIn/Twitter URLs, alternate names, reference IDs)
  and returns ranked candidates **with a confidence score**. Designed for:
  *"I have messy, partial, or fuzzy fields from an external source — give
  me the best mapping with a probability so I can decide whether to trust
  it."* POST endpoint to `/identification/company`. Each result row carries:
  - `mapping_strength` — integer 1–5 (higher = stronger). `mapping_strength_need_review: "Yes"` flags rows that should be human-reviewed before being trusted.
  - `matching_probability` — float 0–1 (raw confidence).
  - `match_summary` / `mismatch_summary` — human-readable strings explaining
    *why* the engine scored it that way (e.g. `"Exact Name Match"` /
    `"Reference ID Mismatch|Phone Mismatch"`).
  - Optional `threshold` query param (default 0.75) filters out
    low-confidence candidates server-side.
  Inputs (all optional except `name`): `name`, `dba`, `url`, `country`,
  `state`, `city`, `address`, `zip`, `phone`, `industry`, `social_facebook`,
  `social_linkedin`, `social_twitter`, `linkedin_url`, `classification`,
  `clean_name`, `fn` (formal/legal name), `aka`, `reference_id`,
  `company_only`.

**Decision rule — when in doubt, prefer `identification`.** It is the full
entity-resolution engine: fuzzy and tolerant, accepts many combinable
signals beyond name/website, and returns a confidence score you can act
on. Use it whenever the input is fuzzy, partial, or from an external
source (CRM dump, prospect list, partner feed), when you hold extra
signals (address, phone, LinkedIn URL, reference ID), or when downstream
automation must gate on confidence. Reserve `match` for the quick first
try on a clean, recognizable name/URL — it matches on name/website ONLY,
strictly, and reports no confidence. **Always fall through to
`identification` when `match` returns 0 or visibly-wrong rows** — a
`match` miss does not mean the entity is absent from PrivCo.

**Stage 2 — discovery (criteria → candidate list)**
- `company_search` — multi-filter (location, industry, keyword, revenue,
  valuation, funding, employees, growth, year founded, PE/VC inclusion). Up to
  50 summary rows per page.
- `people_search` — equivalent for contacts.
- `funding_search` *(coming soon)* — Round-centric search (fundingTypes,
  total, timestamp, industry, location, investor types, keyword). Use when
  the question is *about rounds* rather than *about companies*. Same
  gotchas as `company_search`: `keyword.condition` is required (`"should"`
  / `"must"`). Until released, approximate via `company_search` (with
  funding filters) + per-candidate `vc_deals`.
- `deal_search` *(coming soon)* — M&A-deal-centric search
  (target/buyer/seller industries, deal value, query, isPeDeal, target
  PE/VC-backed flags). **Filter precedence quirk**: setting `isPeDeal` (any
  boolean) SUPPRESSES
  `inclusionExclusion.targetsIsPeBacked`/`targetsIsVcBacked`/`buyersAllOfType`
  — don't mix. Default sort: relevance if `query` set, else `timestamp desc`.
  Until released, approximate via `company_search` + per-candidate `ma_deals`.
- `industry_keyword` — autocomplete for keyword IDs (use before `keyword`
  filter when unsure).
- `industry_pics` — list of PICS top-level industry codes.

**Stage 3 — enrichment (profile_id → full record)**
- `profile` — full **company** profile (revenue, employees, funding, founders,
  contacts, investors, competitors, subsidiaries).
- `general_profile` — full profile for **non-company** entities (Investor,
  Person, etc.). Use when `profile_type != "Company"` from `match`.
- `people` — execs & board on a company.
- `financials` / `revenue_financials` — multi-year statements / revenue-only
  time series.
- `vc_deals` — round-by-round funding (per-round valuation, amount, date,
  investors). Use this for `latestValuation` since `profile` doesn't carry it.
- `ma_deals` — M&A history.

**Stage 4 — contact reveal (hash → details)**
- `company_contact` — resolve a contact hash to full businessEmail / phone /
  LinkedIn.
- `investor_contact` — same for investor-associated contacts.
- Both take the **opaque hash** from `people_search` row's `hash` field.
  **NOT** from `profile.contacts[]` — that block has no hash, just
  `{title, name, email}` summary.
- `associatedType` on the people_search row tells you which reveal endpoint
  to use: `Company` → `company_contact`, `Investor` → `investor_contact`.
- To find Investor-associated rows, you **must** pass
  `{"filters": {"investorOnly": true}}` to people_search — default is
  Company-only and title filters like `["Partner"]` are not effective.
- A successful call **consumes a contact-reveal credit** on the account —
  don't call speculatively.

## Filter semantics — the gotchas

### 1. `location.state` requires the full state name

```jsonc
// WRONG — silently returns 0 hits
{"location": {"selection": [{"type": "state", "value": "NY"}]}}
// RIGHT
{"location": {"selection": [{"type": "state", "value": "New York"}]}}
```

The response payload returns `"hq_state": "NY"`, so the response shape does not
telegraph the required input shape. Verified empirically: `"NY"` → 0 hits,
`"New York"` → 6,444 hits on the same query.

### 2. `industry` vs `keyword` are distinct filters

| Filter | Vocabulary | Example values | When |
|--------|-----------|----------------|------|
| `industry.rawIndustries` (or `.selection`) | PICS top-level | `"Software & Internet Services"`, `"Biotechnology"` | Broad industry buckets |
| `keyword.rawKeywords` (or `.selection`) | Tagged terms from `industry_keyword` autocomplete | `"Fintech"`, `"Payments"`, `"Generative AI"` | Specific business-model / tech tags |

Passing a broad word like `"Software"` to `keyword.rawKeywords` returns **0
hits** — it isn't a registered keyword. Call `industry_keyword(query: "...")`
first when unsure.

### 3. `inclusionExclusion` is a requirement, not an addition

```jsonc
{"inclusionExclusion": {"pe": "include"}}   // REQUIRE PE-backed
{"inclusionExclusion": {"pe": "exclude"}}   // EXCLUDE PE-backed
```

`"include"` means *must be* PE-backed, not *also include*. Combining with other
restrictive filters quickly empties the result set — drop one filter at a time
when debugging zero-result queries.

### 4. `sorting.field` enum is small

Valid: `name | state | employee | industry | yearFounded | totalFunding |
revenueGrowthRate1 | revenueGrowthRate3 | hasContact`.

**Not sortable**: `latestValuation`, `latestRevenue`, `latestEbitda`. For "top
N by valuation" workflows, see Workflow C below.

### 5. `revenue.includeMissing` quietly changes result composition

`includeMissing: true` includes companies with null/empty revenue — common for
early-stage, pre-revenue, and stealth companies, which often have
`latestRevenue: null`. When testing "revenue > X" assertions, leave it
**false** (default).

### 6. `keyword.condition` is required whenever `keyword` filter is used

```jsonc
// WRONG — 400 Bad Request
{"keyword": {"rawKeywords": ["Fintech"]}}
// RIGHT
{"keyword": {"rawKeywords": ["Fintech"], "condition": "should"}}
```

`condition` is `"should"` (match any keyword) or `"must"` (match all). It's
match-engine terminology — don't confuse with `inclusionExclusion`'s
`include` / `exclude` (different filter, different vocabulary).

### 7. `profileType` path param is lowercase

```jsonc
// WRONG — 400 with cryptic "profileType must be one of the following values: "
GET /profile/Company/44612
// RIGHT
GET /profile/company/44612
```

`'company' | 'investor' | 'advisor'`. Affects `profile`, `general_profile`,
`people`, `vc_deals`, `ma_deals`. The MCP handlers don't normalize casing.

### 8. `deal_search.isPeDeal` SUPPRESSES sibling filters

Setting `filters.isPeDeal` to any boolean **silently disables**
`inclusionExclusion.targetsIsPeBacked`, `inclusionExclusion.targetsIsVcBacked`,
and `buyersAllOfType` in the same request. The precedence is intentional —
pick one approach per query:

- Want *only* PE deals? → `{"isPeDeal": true}` (alone).
- Want deals where the *target* is PE-backed? → omit `isPeDeal`, use
  `{"inclusionExclusion": {"targetsIsPeBacked": "include"}}`.

### 9. `deal_search` sort default depends on whether `query` is set

When `sorting` is omitted: response is ordered by **relevance scoring** if
`filters.query` is set, otherwise by `timestamp desc`. Explicit `sorting`
always wins. Pass `{"sorting": {"field": "timestamp", "order": "desc"}}`
when you want deterministic ordering regardless of query presence.

## Summary rows: included vs omitted

`company_search` summary rows contain:
- ✅ `id`, `name`, `state`, `industry`, `url` (permalink), `keywords[]`,
  `classification` (Private/Public), `revenueGrowthRate1`,
  `revenueGrowthRate3`, `latestEbitda`
- ❌ `latestRevenue`, `latestValuation`, `totalFunding`, `yearFounded`,
  `ownersFounders`, `headquarters.city`, `summary`, funding-round detail, M&A
  detail

If you need any omitted field, follow up with `profile` per company.
Round-level data → also `vc_deals`. M&A detail → also `ma_deals`. **Budget the
round-trips when planning multi-candidate work** — a 22-company dataset is
~22+ profile calls plus targeted vc_deals/ma_deals.

## Practical workflows

### A. Named-entity lookup

```
match(name="Airbnb")            → profile_id 78
profile(78)                     → full record
vc_deals(profileType="company", profileId=78)   → all rounds + valuations
ma_deals(profileType="company", profileId=78)   → M&A history
```

3–4 calls per entity for full enrichment.

### B. Filter-criteria discovery

```
1. industry_keyword(query="fintech")    → keyword id (if using id-based filter)
2. company_search({
       location: {selection: [{type: "state", value: "New York"}]},
       industry: {rawIndustries: ["Software & Internet Services"]},
       revenue: {min: 1000000},
       inclusionExclusion: {pe: "include"}
   }, sorting: {field: "totalFunding", order: "desc"})
3. For each candidate of interest → profile(id) for dollar fields
```

### C. "Top N by valuation" (two-stage pattern)

```
1. company_search(filters: {latestValuation: {min: 1000000000}},
                  sorting: {field: "totalFunding", order: "desc"})
2. Take top ~50 (totalFunding correlates roughly with valuation tier)
3. profile() each → real latestValuation
4. Sort client-side by latestValuation desc, take top N
```

The API can filter by valuation but not sort by it.

### D. Recent funding ("raised >X in year Y")

```
1. company_search(filters: {latestFundingYear: {min: 2023},
                            funding: {min: 10000000}})
2. For round-level confirmation, vc_deals() each → inspect round dates
```

`latestFundingYear` filters on the *most recent* round; `funding.min` on the
**cumulative total**. If the user means "raised >$10M in 2023 specifically",
confirm with `vc_deals` round dates.

## Field-shape gotchas

- Dollar fields are VARCHAR strings on `profile` (`"19400000000"`) — parse
  client-side.
- `profile.investors[]` is aggregated across rounds with duplicates — dedupe
  before display.
- `vc_deals` dates use split-VARCHAR pattern: `{"year": "2017", "month": "3",
  "day": "9"}`. `day` may be `""`. Compare as strings.
- `vc_deals` rows where target is a *different* company are returned when the
  queried company acted as an investor — filter by `target_company_id ==
  queried_id` for self-funding only.
- `profile.summary` is often empty for non-flagship entities — don't promise
  it in assertions.
- There is no `latestValuation` field on `profile`. Get it from
  `vc_deals` (last round's `valuation`).
- Public companies have null/no private-valuation field; their market cap
  isn't in PrivCo.

## When building a fixture / assertion dataset

- Every "filter must exclude X" assertion needs a representative X **inside
  the dataset**, otherwise the test passes vacuously. Pair each positive slot
  with at least one negative control on the same axis (CA-software vs
  NY-software, low-rev vs rev>1M, pre-2015 vs post-2015).
- Reuse rich anchor entities across many slots to keep the dataset small.
- Track coverage in an explicit matrix doc so the fixture set can be
  re-validated when the assertion list changes.

## Presenting company data — the dashboard template

When the user wants a company's data **presented** (a dashboard, one-pager,
or visual intelligence view) rather than answered in prose, use the
ready-made template in `references/dashboard_prompt.md`. It chains the
standard Workflow A data pull (`match → profile → financials → vc_deals /
ma_deals → people`) into a single HTML dashboard widget: header + 6 metric
cards + revenue/headcount charts + valuation range + financial/profile/
funding tables, with the PrivCo house style rules (CSS variables, Chart.js
conventions, sentence case). Follow its data-mapping notes — they encode the
same field-shape gotchas listed above.

## Access & auth — two transports, same 17 tools

- **Remote connector (recommended):** PrivCo-hosted endpoint at
  `https://mcp.privco.com/mcp` (MCP Streamable HTTP). Auth is OAuth 2.1 —
  the user signs in with their PrivCo account in a browser; no API key in
  any config. Works in claude.ai, Claude Desktop, Claude Code
  (`claude mcp add --transport http`), ChatGPT (developer mode), and any
  spec-compliant client. Requires API access enabled on the PrivCo account.
- **Local server (npm `privco-data-mcp`, stdio):** reads `PRIVCO_API_KEY`
  from its environment (set in the MCP client config) and sends it to the
  PrivCo v3 API as the `x-api-key` header. For automation/CI or key-based
  setups.
- Tool names and schemas are identical on both; only the client-visible
  prefix differs (it carries whatever server/connector name was registered).
- Full setup steps + troubleshooting for both paths: `INSTALL.md`.
- For the full per-tool field reference, common pitfalls in one-line form, and
  expanded workflow detail, see `references/usage_guide.md` in this skill
  folder.

## When this skill applies — and when not

**Apply** when the user is doing entity lookup, criteria-driven discovery,
fixture/dataset building, or any multi-tool MCP workflow against PrivCo data.

**Skip** when the user is editing the `privco-data-mcp` server source itself,
or when they're talking to the PrivCo REST API directly without the MCP
wrapper.
