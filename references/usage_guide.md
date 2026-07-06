# PrivCo MCP — Full Usage Reference

Consolidated reference for using the **PrivCo MCP tools**
(`mcp__privco-data-mcp__*`) and the underlying PrivCo v3 REST API they wrap.
Captures field shapes, filter semantics, gotchas, and practical workflows
discovered while building real fixtures.

This file is the deep reference. The compact distillation lives in
`../SKILL.md`. Install/setup lives in `../INSTALL.md`.

## MCP tool inventory

The PrivCo MCP server registers **17 tools** — the same inventory whether
you use the hosted remote connector (`https://mcp.privco.com/mcp`, OAuth)
or the local `privco-data-mcp` npm server (see `../INSTALL.md` for both
paths). Two of them — `funding_search` and `deal_search` — are documented
for forward compatibility and are **coming soon as expanded features**;
those rows below are marked *(coming soon)*. The rest of the inventory is
live today.

Tools group naturally by usage stage:

### Stage 1 — Entity resolution (name/details → profile_id)

These two tools look superficially similar but solve **different problems**.
The framing below corrects a common misconception (often inherited from
older docs) that `match` is the fuzzy/flexible option and `identification`
is the strict one. The truth is the inverse.

#### `match` — strict, name/website-only

| Property | Value |
|----------|-------|
| Method / path | `GET /match` |
| Inputs | `name`, `website`, `includeCityState` (boolean) |
| Matching strictness | **Strict / exact-ish.** Designed for clean recognizable inputs. Returns 0 results if the input differs even moderately from the canonical name/URL. |
| Output cardinality | Up to 20 ranked candidates as a flat list (`{data: [...], meta}`). |
| Confidence info | **None.** No score field. You judge the result by row position, name similarity, and disambiguator fields. |
| Use when | The user provided a clean, well-known company name or URL and you want the canonical row(s). |
| Cost | One cheap call. |

Row shape:

```jsonc
{
  "name": "Stripe, Inc.",
  "profile_id": 44612,
  "profile_type": "Company",                  // or "Investor", "Advisor"
  "profile_classification": "Private",        // or "Public", "PREQ"
  "permalink": "/company/stripe",
  "location": "United States",
  "industry": "Software & Internet Services",
  "website": ["stripe.com"]
}
```

**Failure mode**: `match` silently returns 0 rows when the input doesn't
closely match canonical data — `"Sequoia Capital"` returns 0 even though
Sequoia has a profile, because the registered name is different. If
`match` returns 0 or visibly-wrong rows for what should be a known
entity, **escalate to `identification`** — that's the forgiving path.

#### `identification` — flexible, multi-field, scored

| Property | Value |
|----------|-------|
| Method / path | `POST /identification/company` |
| Inputs | `name` (required) + many optional fuzzy hints: `dba`, `url`, `country`, `state`, `city`, `address`, `zip`, `phone`, `industry`, `social_facebook`, `social_linkedin`, `social_twitter`, `linkedin_url`, `classification`, `clean_name`, `fn` (formal name), `aka`, `reference_id`, `company_only`, `index` |
| Matching strictness | **Flexible / tolerant of incomplete or fuzzy inputs.** More inputs = more signals to score against, but every field is optional. |
| Output cardinality | Ranked list with **mapping-strength scores** on every row. Optional `threshold` query param (default 0.75) caps at low-confidence. |
| Confidence info | `mapping_strength` (int 1–5), `matching_probability` (float 0–1), `score_total` (raw matching score), `name_similarity` (0–1), `leading_distance` (0–1), `match_summary` / `mismatch_summary` (human-readable strings), `mapping_strength_need_review` (`"Yes"` / `"No"`). |
| Use when | The user has messy, partial, or fuzzy data (CRM dump, partner feed, prospect list), or you need a **probability** to gate downstream automation. |
| Cost | One cheap call. |

Row shape:

```jsonc
{
  "mapped_profile_id": 44612,
  "mapped_profile_type": "Company",
  "mapped_profile_name": "Stripe, Inc.",
  "mapped_profile_url": "stripe.com",
  "mapped_published": "Yes",

  // ── Scoring fields — the reason to use identification ──
  "mapping_strength": 4,                       // int 1-5; 4+ usually trustworthy
  "mapping_strength_need_review": "Yes",       // human-review flag
  "matching_probability": 0.176,               // float 0-1
  "score_total": 783.446,                      // raw matching score
  "name_similarity": 1,                        // 0-1
  "leading_distance": 0.5,                     // 0-1
  "match_summary": "Exact Name Match",
  "mismatch_summary": "Reference ID Mismatch|Phone Mismatch",

  // ── Address / classification ──
  "mapped_profile_classification": "Private",
  "mapped_pic": "Software & Internet Services",
  "mapped_profile_city": "San Francisco",
  "mapped_profile_state": "CA",
  "mapped_profile_country": "US",
  "mapped_profile_address": "354 Oyster Point Boulevard",
  "mapped_reference_id": "EIN:12-3456789|CIK:0001234567|...",
  // ... plus mapped_parent_company, mapped_clean_name, etc.
}
```

**Reading the score fields**: `mapping_strength` is the headline 1–5
ordinal you'd show a human. `matching_probability` is the raw model
output — useful for thresholding (`>0.75` is the server default for
auto-accept). `match_summary` / `mismatch_summary` are pipe-delimited
strings that explain *what* matched / *what didn't* (e.g. `"Exact Name
Match"` is good; `"Reference ID Mismatch|Phone Mismatch"` flags that the
name aligns but other identifiers disagree — typical pattern when a
domain or phone changed). `mapping_strength_need_review: "Yes"` is the
explicit signal to **not** auto-trust without human review.

#### Decision rule

| Situation | Pick |
|-----------|------|
| Clean company name or canonical URL, looking for the row | `match` |
| Messy / partial input (missing URL, ambiguous name, foreign address) | `identification` |
| Need a probability/confidence to gate downstream automation | `identification` |
| Bulk pipeline mapping external records to PrivCo IDs | `identification` (use `/identification/batch` for >a few records) |
| `match` returned 0 rows or visibly-wrong rows for a known entity | Retry with `identification` |

### Stage 2 — Discovery (criteria → list of candidates)

| Tool | Purpose | Returns |
|------|---------|---------|
| `company_search` | Multi-filter search across location, industry, keyword, revenue, valuation, funding, employees, growth, year founded, PE/VC inclusion | Up to 50 summary rows per page; max 129 pages on broad queries |
| `people_search` | Equivalent for people/contacts | Summary rows |
| `funding_search` *(coming soon)* | Round-centric search: filter by `fundingTypes` (13-token enum incl. `equity_funding_A`–`H`, `seed_angel`, `debt funding_financing`, etc.), `timestamp` (unix-ms range), `totalInUsd`, `industry`, `location`, `investorTypes`, `keyword`. `keyword.condition` REQUIRED. Sort by `companyName` / `timestamp` / `totalInUsd` / `fundingType`. **Coming soon as an expanded feature.** | Summary rows of funding rounds: `{id, companyName, companyUrl, timestamp, industryName, fundingType, roundDescription, totalInUsd, investors[], industryKeywords[]}` |
| `deal_search` *(coming soon)* | M&A-deal-centric search: `query` (free-text on buyer/seller/target names), `timestamp`, `totalInUsd`, `buyersIndustries`, `sellersIndustries`, `targetsIndustries`, `targetsIndustryKeywords`, `isPeDeal`, `inclusionExclusion.{targetsIsPeBacked, targetsIsVcBacked}`, `buyersAllOfType` ∈ {Financial, Private, Public}, `location`. **`isPeDeal` SUPPRESSES `targetsIsPeBacked`/`targetsIsVcBacked`/`buyersAllOfType` when set.** Sort: `timestamp` only; default = relevance if `query` set, else `timestamp desc`. **Coming soon as an expanded feature.** | Summary rows of deals: `{id, timestamp, totalInUsd, targetIndustries[], buyers[{name,urlSlug}], targets[{name,urlSlug}]}`. `totalInUsd: "--"` (string) when missing. |
| `industry_keyword` | Autocomplete keyword IDs (used in `company_search.keyword.selection`) | Keyword id+name list |
| `industry_pics` | List of PICS top-level industry codes | Industry id+name list |

### Stage 3 — Enrichment (profile_id → full record)

| Tool | Returns |
|------|---------|
| `profile` | Full **company** profile: address, summary, revenue, employees, total funding, growth rates, founders, contacts, investors (all rounds aggregated), competitors, subsidiaries, former names |
| `general_profile` | Full profile for **non-company** entities (Investor, Person, etc.) — distinct endpoint shape from `profile`. Use when `match` returns `profile_type != "Company"` (Investor / Person / etc.) |
| `people` | Executives & board members on a company |
| `financials` | Multi-year financial statements |
| `revenue_financials` | Revenue-only time series |
| `vc_deals` | Round-by-round VC funding history (target/investor role); includes per-round valuation, amount, date, investor list |
| `ma_deals` | M&A history (target/buyer/seller role) |

### Stage 4 — Contact reveal (hash → details)

| Tool | Purpose |
|------|---------|
| `company_contact` | Resolves a Company-associated contact hash to full businessEmail / directNumbers / mobileNumbers / personalLinkedin |
| `investor_contact` | Resolves an Investor-associated contact hash to the same shape |

**Where the hash comes from**: each row of a `people_search` response
carries a `hash` field plus an `associatedType` of `"Company"` or
`"Investor"`. That `hash` is the input to the reveal endpoints — pick the
endpoint based on the row's `associatedType`. **A profile's `contacts[]`
block does NOT carry hashes** (only `{title, name, email}` summaries), so
you cannot start from a profile and reveal directly.

**Finding an Investor-associated row** requires passing
`{"filters": {"investorOnly": true}}` to `people_search`. The default
behavior returns Company-associated rows only, and `title` filters like
`["Managing Partner", "General Partner"]` empirically do not narrow to
Investor-associated rows.

**Credit cost**: a successful reveal consumes one contact-view credit on
the account (separate counters for `company_contact_view` and
`investor_contact_view`). Don't call speculatively.

Response shape (PII fields shown as `<redacted>` for documentation):

```jsonc
{
  "data": {
    "hash": "<the same hash you passed in>",
    "id": 10000001,
    "yearLastSeen": "2025",
    "firstName": "Jane",                  // example values — fictional
    "lastName": "Doe",
    "jobTitles": ["Regional Vice President"],
    "seniorityLevel": "Vp",
    "department": "Sales",
    "businessEmail": "<redacted>",
    "personalCity": "<redacted>",
    "personalState": "<redacted>",
    "personalCountry": "<redacted>",
    "directNumbers": [],
    "mobileNumbers": ["<redacted>"],
    "personalLinkedin": "<redacted>",
    "companyName": "Databricks, Inc."     // or "investorName" for investor_contact
  },
  "meta": { "call": "company_contact", "status": "success", "current_time": 1778692429873 }
}
```

## Filter semantics — the non-obvious bits

### 1. `location.state` requires the full state name

```jsonc
// WRONG — silently returns 0 hits
{"location": {"selection": [{"type": "state", "value": "NY"}]}}

// RIGHT
{"location": {"selection": [{"type": "state", "value": "New York"}]}}
```

The response payload returns the **abbreviated** form (`"hq_state": "NY"`),
so the response shape does not telegraph the required input shape. Verified
empirically: `"NY"` → 0 hits, `"New York"` → 6,444 hits on the same query.

### 2. `industry` vs `keyword` are distinct filters

| Filter | Vocabulary | Example values | When to use |
|--------|-----------|----------------|-------------|
| `industry.rawIndustries` (or `.selection`) | PICS top-level categories | `"Software & Internet Services"`, `"Biotechnology"`, `"Aerospace & Defense"` | Broad industry buckets |
| `keyword.rawKeywords` (or `.selection`) | Tagged keywords from the `industry_keyword` autocomplete dictionary | `"Fintech"`, `"Payments"`, `"Generative AI"` | Specific business-model / tech tags |

Passing a broad word like `"Software"` to `keyword.rawKeywords` returns **0
hits** — it isn't a registered keyword. Always call
`industry_keyword(query: "...")` first if you're unsure whether a term is a
keyword or an industry.

### 3. `inclusionExclusion` is a requirement, not an addition

```jsonc
{"inclusionExclusion": {"pe": "include"}}   // REQUIRE PE-backed
{"inclusionExclusion": {"pe": "exclude"}}   // EXCLUDE PE-backed
```

`"include"` here means "must be PE-backed", not "also include PE-backed
companies in addition". Combining with other restrictive filters quickly
empties the result set — drop one filter at a time when debugging zero-result
queries.

### 4. `sorting.field` enum is small

Only these are valid: `name | state | employee | industry | yearFounded |
totalFunding | revenueGrowthRate1 | revenueGrowthRate3 | hasContact`.

**Not sortable**: `latestValuation`, `latestRevenue`, `latestEbitda`. For
"top N by valuation" workflows, filter by `latestValuation.min` and sort by
`totalFunding` desc as a rough proxy — then re-rank client-side after
fetching profiles for the top candidates.

### 5. `revenue.includeMissing` quietly changes result composition

Setting `includeMissing: true` includes companies with null/empty revenue —
common for early-stage, pre-revenue, and stealth companies, which often have
`latestRevenue: null` in their records. When testing "revenue > X" assertions,
leave it **false** (default) and explicitly pull missing-revenue companies via
a separate query when you need negative controls.

### 6. `keyword.condition` is required whenever the `keyword` filter is used

```jsonc
// WRONG — HTTP 400: "filters.keyword.condition must be one of the following values: "
{"filters": {"keyword": {"rawKeywords": ["Fintech"]}}}

// RIGHT — pick `"should"` (match any) or `"must"` (match all)
{"filters": {"keyword": {"rawKeywords": ["Fintech"], "condition": "should"}}}
```

The `condition` field is match-engine terminology — distinct from the
`include` / `exclude` vocabulary used by `inclusionExclusion`. Applies to
both `company_search` and `funding_search` (when it ships).

### 7. `profileType` path param is lowercase

```jsonc
// WRONG — HTTP 400: "profileType must be one of the following values: " (empty list in the error, confusing)
GET /profile/Company/44612

// RIGHT
GET /profile/company/44612
```

Allowed values: `'company' | 'investor' | 'advisor'`. Affects every tool
that takes a `profileType` in the URL — `profile`, `general_profile`,
`people`, `vc_deals`, `ma_deals`. The MCP handlers pass the string through
unchanged, so casing matters end-to-end.

### 8. `people_search` defaults to Company-associated rows

Without `{"filters": {"investorOnly": true}}`, every row in the response
has `associatedType: "Company"` — regardless of what title filter you
pass. This matters when you need a row's `hash` to call
`investor_contact`: you must explicitly opt into Investor results.

### 9. `deal_search.isPeDeal` suppresses sibling filters

Setting `filters.isPeDeal` (any boolean value) **suppresses**:

- `inclusionExclusion.targetsIsPeBacked`
- `inclusionExclusion.targetsIsVcBacked`
- `buyersAllOfType`

The precedence is:

1. `isPeDeal` is not boolean → apply `inclusionExclusion.*` and `buyersAllOfType`.
2. `isPeDeal` is boolean AND no `targetsIsPeBacked` AND no `buyersAllOfType`
   → `isPeDeal` takes effect.
3. `isPeDeal` is not boolean AND `buyersAllOfType` is set → buyer-type
   filter takes effect.

So a request that includes both `isPeDeal` and `targetsIsPeBacked` is
ambiguous and will silently drop one of them. Pick one per query.

### 10. `deal_search` default sort: relevance vs timestamp

When no `sorting` is supplied:

- `filters.query` is set → results ordered by **relevance scoring**.
- `filters.query` is absent → default to `timestamp desc`.

Explicit `sorting` always wins. Pass an explicit
`{"sorting": {"field": "timestamp", "order": "desc"}}` for deterministic
ordering regardless of whether `query` is present.

### 11. `deal_search` response: `totalInUsd` is the string `"--"` when missing

```jsonc
{"id": 12345, "timestamp": 1700000000000, "totalInUsd": "--", ...}
```

Don't expect `null`. Either coerce the string before numeric ops or filter
the missing rows out client-side. The same `includeMissing` toggle on the
input filter exists if you want to bound this at the source.

### 12. `funding_search` and `deal_search` permissions

Once released, both endpoints will require the user's API key to carry
the matching permission (`funding_search` or `deal_search`, or
`everything`). A key without the permission will return 403. Contact
support@privco.com if your key needs these permissions added.

## What summary rows include vs. omit

`company_search` returns a paginated list of summary rows. Each row contains:

✅ **Included**: `id`, `name`, `state`, `industry`, `url` (permalink),
`keywords[]` (id+name), `classification` (Private/Public),
`revenueGrowthRate1`, `revenueGrowthRate3`, `latestEbitda`

❌ **Omitted**: `latestRevenue`, `latestValuation`, `totalFunding`,
`yearFounded`, `ownersFounders`, `headquarters.city`, `summary`,
funding-round detail, M&A detail

If you need any of the omitted fields, you must follow up with `profile` per
company. For round-level data, also `vc_deals`. For M&A details, also
`ma_deals`. **Budget the round-trips** when planning multi-candidate fixture
work — a 22-company dataset is ~22+ profile calls, plus targeted
vc_deals/ma_deals for entities that need history.

## Practical workflows

### Workflow A — Named-entity profile lookup

```
match(name="Airbnb")            → profile_id 78
profile(78)                     → full record (revenue, employees, founders, ...)
vc_deals(profileType="company", profileId=78)   → all rounds with valuations
ma_deals(profileType="company", profileId=78)   → M&A history
```

Use this for entity-specific queries ("funding info for Airbnb", "founders of
Airbnb", "summarize Databricks"). Three to four calls per entity for full
enrichment.

### Workflow B — Filter-criteria candidate discovery

```
1. industry_keyword(query="fintech")    → keyword id (if using id-based filter)
2. company_search(filters: {
       location: {selection: [{type: "state", value: "New York"}]},
       industry: {rawIndustries: ["Software & Internet Services"]},
       revenue: {min: 1000000},
       inclusionExclusion: {pe: "include"}
   }, sorting: {field: "totalFunding", order: "desc"})
3. For each candidate of interest → profile(id) to get the omitted dollar fields
```

Use this for population queries ("show me NY software companies with revenue
> $1M"). Expect to filter and re-rank client-side after stage 3 since
`latestRevenue` and `latestValuation` aren't sortable.

### Workflow C — "Top N by valuation" (two-stage pattern)

```
1. company_search(filters: {latestValuation: {min: 1000000000}},
                  sorting: {field: "totalFunding", order: "desc"})
2. Take top ~50 candidates (totalFunding correlates roughly with valuation tier)
3. profile() each → real latestValuation per company
4. Sort client-side by latestValuation desc, take top N
```

Two-stage because the API can filter by valuation but not sort by it.

### Workflow D — Recent funding ("raised >X from year Y")

```
1. company_search(filters: {
       latestFundingYear: {min: 2023},
       funding: {min: 10000000}
   })
2. For round-level confirmation, vc_deals() each candidate and inspect round dates
```

`latestFundingYear` filters on the *most recent* round; `funding.min` on the
**cumulative total** funding. If you need "raised >$10M in 2023 specifically"
(not "raised >$10M total and most recently in 2023"), confirm with
`vc_deals` round dates.

## Field shape reference (response payloads)

### `profile` (company) — key fields

```jsonc
{
  "profile_id": 44612,
  "name": "Stripe, Inc.",
  "classification": "Private",
  "address": {"city": "San Francisco", "state": "CA", "country": "United States"},
  "industry": "Software & Internet Services",
  "index_vc_backed": "Yes",          // "Yes" | "No"
  "index_pe_backed": "Yes",          // "Yes" | "No" — derive companyTags from these two
  "latest_revenues": "19400000000",  // VARCHAR-typed; parse to number client-side
  "latest_revenues_year": 2025,
  "latest_employees": 10000,
  "total_funding": "6500000000",     // VARCHAR
  "revenue_growth_rate_1yr": "7.7778",
  "founder": [{"name": "Patrick Collison"}],
  "contacts": [{"title": "CEO & Co-Founder", "name": "...", "email": "..."}],
  "investors": [/* aggregated across ALL rounds — may contain duplicates */],
  "competitors": [],
  "subsidiary": [],
  "summary": "Stripe, Inc., founded in 2010 ..."   // may be empty string
}
```

Notes:
- Dollar fields are VARCHAR strings (`"19400000000"`), reflecting the
  underlying database convention. Parse client-side.
- `investors[]` is aggregated across rounds and contains duplicates — dedupe
  before display.
- `summary` is often empty for non-flagship entities; don't promise it in
  assertions.
- There is no `latestValuation` field directly — get it from `vc_deals` (last
  round's `valuation`) or from press/research.

### `vc_deals` — round shape

```jsonc
{
  "target_company_id": 78,
  "target_company_name": "Airbnb, Inc.",
  "vc_deal_id": 215068,
  "round": "F",                            // "Series A" → "A", debt rounds → "Debt Funding_financing"
  "currency": "USD",
  "total": "447800000",                    // VARCHAR
  "valuation": "31000000000",              // VARCHAR; "" when undisclosed
  "date": {"year": "2017", "month": "3", "day": "9"},   // split-VARCHAR pattern; day may be ""
  "investors": [{"investor_name": "...", "investor_profile_id": "...", "investor_profile_type": "Investor|Company"}]
}
```

Notes:
- Date uses the split-VARCHAR pattern. For "this happened in 2023" filters,
  compare `date.year == "2023"` as strings.
- Rounds where the target is **a different company** (e.g., when the queried
  company is acting as an investor) are also returned — filter by
  `target_company_id == queried_id` for self-funding only.

### `match` — disambiguation tips

- Returns up to 20 results. First row is usually correct for well-known
  unique brands ("Airbnb", "Databricks").
- Common-word names ("Stripe") return many false positives — disambiguate by
  `profile_type == "Company"`, `profile_classification` (Private/Public/PREQ),
  and `industry`. Stripe Inc. is `profile_id=44612` and is **Private**
  (correct) — distinguish from `Stripes Group` (Investor) and `Stripes
  Convenience Stores` (Company, but TX retailer).
- Pass `includeCityState: true` to surface HQ city/state in the result rows —
  helps disambiguate when names collide.

## Common pitfalls (one-line form)

- Forgetting that `state` needs the full name (silent 0 hits)
- Using `keyword.rawKeywords` for broad terms instead of
  `industry.rawIndustries`
- Trying to sort by `latestValuation` (not in enum)
- Trusting `company_search` summary rows for dollar amounts (they aren't
  there)
- Reading `investors[]` from `profile` and presenting it as unique investors
  (it's per-round, with duplicates)
- Forgetting that public companies have `null` or no private-valuation field
  (their market cap isn't in PrivCo)
- Filtering `revenue.min` then being surprised that early-stage companies
  (null revenue) don't show up — that's correct; set `includeMissing: true`
  only when intentional
- Omitting `keyword.condition` when using the `keyword` filter (HTTP 400)
- Capitalized `profileType` in path params (HTTP 400 — must be lowercase)
- Trying to reveal contacts from `profile.contacts[]` (no hash exists
  there; reveal hashes come from `people_search` rows)
- Filtering `people_search` by `title: ["Partner"]` expecting Investor rows
  (use `{filters: {investorOnly: true}}` instead)
- Treating POST endpoints as failed on `201` — they're success; accept any
  `2xx` status
- On `deal_search`: combining `isPeDeal` with `targetsIsPeBacked` /
  `targetsIsVcBacked` / `buyersAllOfType` (the latter get silently
  suppressed)
- On `deal_search`: omitting explicit `sorting` and being surprised that
  results aren't time-ordered (with `query` set the default is relevance,
  not timestamp)
- Forgetting that `funding_search` / `deal_search` will require a
  separate permission on the API key once released — a generic key
  returns 403

## Auth & deployment

- **Remote connector** (`https://mcp.privco.com/mcp`): OAuth sign-in with
  your PrivCo account — no key to manage.
- **Local server**: `privco-data-mcp` reads `PRIVCO_API_KEY` from its
  environment and exits if missing; it connects to the PrivCo v3 API and
  sends the key as the `x-api-key` header.
- Setup and MCP-client registration for both paths: see `../INSTALL.md`.
- Package source: <https://www.npmjs.com/package/privco-data-mcp>
- Support: support@privco.com — API documentation: <https://www.privco.com/api-documentation>
