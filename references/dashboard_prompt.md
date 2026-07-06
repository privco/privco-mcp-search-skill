# Company Intelligence Dashboard — presentation template

A ready-to-use prompt template for turning PrivCo MCP data into a single,
polished HTML **company intelligence dashboard**. Use it when the user asks
to "build a dashboard / one-pager / intelligence view" for a company, or
wants PrivCo data presented visually rather than as prose.

Replace `[WEBSITE URL]` with the target company's domain, then follow the
steps. The data-gathering sequence reuses the standard named-entity workflow
from `SKILL.md`; the output section is the PrivCo house dashboard design.

---

Research the company at [WEBSITE URL] using PrivCo MCP tools,
then enrich with web sources, and render a single intelligence dashboard.

## Step 1 — PrivCo MCP (run in sequence)

1. `match` by website domain → get profileId (fall back to
   `identification` if `match` returns 0 rows)
2. `profile` (profileType="company") → full company details
3. `financials` → all revenue + employee history by year
4. `vc_deals` + `ma_deals` → funding history
5. `people` → leadership team

## Step 2 — Web enrichment (optional but recommended)

Search the open web for anything recent that post-dates the PrivCo record:

- Recent revenue milestones (rankings lists, founder interviews)
- Current headcount (LinkedIn, company careers page)
- Latest leadership changes (company blog, LinkedIn, news)
- Recent funding or M&A (press releases)
- Customer count, pricing, product news (company website, blog)

## Output — single intelligence dashboard widget

Build one HTML widget with these sections in order:

### Header
- Company initials avatar (background-info, text-info)
- Company name + domain
- One-line descriptor: industry · city · founded year · PrivCo ID
- Top-right badge: ownership type (bootstrapped / VC-backed / PE-backed)

### Key metrics row (6 cards, background-secondary, no border)
- Latest revenue (with YoY % badge in success color)
- Employees (with trend note vs prior year)
- Estimated valuation (with multiple basis note)
- Total funding raised
- Revenue per employee
- 3-year revenue CAGR

### Two side-by-side charts (Chart.js, or inline SVG where external scripts are unavailable)
- Left — Revenue history: bar chart, year labels on x-axis, $M on y-axis
- Right — Headcount history: line chart with fill, same year range
- Both: disable default legend, build custom HTML legend below each chart
- Both: role="img" + aria-label on canvas, responsive,
  maintainAspectRatio false

### Valuation range bar
- Three horizontal bars: bear / base / bull, using revenue multiples
  appropriate to the company's sector applied to latest revenue — anchor to
  the last `vc_deals` round valuation when one exists
- Teal color ramp: lightest for bear, darkest for bull
- Dollar values on right, multiple labels on left
- One-line methodology note below stating the multiples used and why

### Annual financial summary table
- Columns: Year | Revenue | YoY Growth | Employees | Rev/Employee
- Growth column: pill badges in success color
- Most recent year row in font-weight 500
- Full border grid, 0.5px border-tertiary

### Company profile table
- Two-column key-value layout
- Rows: Full legal name, AKA/former name, Founded, Founder, CEO,
  Industry, Ownership, Customers (est.), Key competitors
- Competitors as small badge pills (background-secondary)

### Funding history table
- Columns: Date | Round | Amount | Post-money valuation
- Round type as pill badge (info color)
- If no additional rounds, italicized note explaining bootstrapped status

### Footer / disclaimer
- One line: sources used, data as-of date, confidence level note

## Style rules

- Never hardcode hex colors on any text — always use CSS variables
- CSS variables only: --color-text-primary/secondary/tertiary,
  --color-background-primary/secondary, --color-border-tertiary,
  --color-background-success/info, --color-text-success/info/warning
  (define them at the top of the widget with light/dark values if the host
  page doesn't provide them)
- Metric cards: background-secondary, border-radius-md, padding 14px 16px
- Grids and tables: 0.5px solid border-tertiary, border-radius-lg
- Label columns in tables: background-secondary, font-weight 500
- Chart.js colors: use hardcoded hex only inside Chart.js config
  (CSS vars don't resolve on canvas) — use mid-range blues and teals
- No gradients, no box-shadows, no emoji, no dark outer backgrounds
- Sentence case everywhere, including chart labels and table headers
- Font weights: 400 regular, 500 bold only — never 600 or 700
- All content visible on load — no tabs, no hidden sections

## Data-mapping notes (PrivCo-specific)

- `latestValuation` is **not** on `profile` — take the last round's
  valuation from `vc_deals`.
- Dollar fields on `profile` are VARCHAR strings (`"19400000000"`) —
  parse before charting.
- `vc_deals` dates are split fields `{year, month, day}`; `day` may be
  empty — format defensively.
- `profile.investors[]` contains duplicates across rounds — dedupe
  before display.
- If a metric is genuinely absent (e.g. pre-revenue company), show the
  card with an em-dash and a footnote rather than omitting the card.
