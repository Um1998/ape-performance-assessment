# APE Performance Report — Power BI Assessment

A four-page Power BI report for a life insurance agency, covering sales performance
(APE), target achievement, agent-level detail, and policy persistency.

## Pages

| Page | Purpose |
|---|---|
| Executive Summary | Company-wide KPI strip, monthly APE trend vs target, Top 10 agents by APE. Filterable by Region, Channel, Year. |
| Agent Performance Grid | Every agent's APE vs target status at a glance, via a bullet-style Deneb visual. Drillthrough into an individual agent's profile. |
| Agent Profile Card | A single agent's current standing — YTD APE, rank, month-over-month trend, top product, persistency — built as a dynamic HTML visual. |
| Persistency Cohort | Policies grouped by issue year, with renewal rate tracked across subsequent years, plus the portfolio's current Active Policy Count. |

## Data Model

Star schema, `fact_sales` at the centre, with `fact_target` and `fact_persistency`
as secondary fact tables sharing the same conformed dimensions:
`dim_agent`, `dim_date`, `dim_product`.

### Key decisions

**Unresolved `agent_id` values in the fact tables.**
`fact_sales' contain `agent_id` values with no match in
`dim_agent`. Rather than let Power BI silently drop these into a blank row, an
explicit **"Unknown Agent"** member (`agent_id = -1`) was added to `dim_agent` in
Power Query, and any unmatched `agent_id` in the fact table is remapped to it. This
keeps every dollar of APE and every policy attributable and visible in totals,
rather than disappearing from the report.

**Agent Name is not a unique identifier.**
More than one agent can share the same `agent_name`. A derived column,
`agent_identifier` (`"<agent_name> (<agent_id>)"`), was added to `dim_agent` and is
used wherever a single agent needs to be unambiguously selected or filtered —
the Agent Performance Grid visual and the Page 2 → Page 3 drillthrough both key off
this field instead of the raw name.

**Dual date relationships on `fact_persistency`.**
Each policy has both an `issue_date_key` and a `renewal_due_date_key`, both
relating to `dim_date`. `renewal_due_date_key` is the active relationship — the
default read of "persistency in year X" is *policies whose renewal came due in
year X*, which matches how the business tracks renewal performance day to day.
The `issue_date_key` relationship is kept inactive and switched on explicitly via
`USERELATIONSHIP()` inside the cohort measures on Page 4, where the question
being asked is different: *how do policies from a given issue-year cohort perform
over time*, regardless of when their renewals fell due.

**`fact_target` grain.**
Target data is provided at year + month grain with no natural date key, joined to
`dim_date` via a calculated `date_key_target` column. This currently produces a
many-to-many relationship against the daily `dim_date` table (every day in a month
shares the same target key). It's functional for month-and-above reporting but is
flagged under Known Limitations below, since a proper month-grain bridge table
would be the more robust long-term design.

**Page 3 date logic — "current," not "as selected."**
The Agent Profile Card deliberately does not depend on any date filter arriving
from Page 2. YTD APE, Rank, and Month-over-Month trend are all self-anchoring:
each measure independently finds *that specific agent's* most recent activity and
builds its own date window from there. This was a deliberate design choice — a
profile card reads more naturally as "this agent's current standing" than as a
number that silently changes meaning depending on what a previous page happened
to have selected. The tradeoff, and the alternative (locking to whatever year was
selected upstream), is discussed in more detail in code comments on the relevant
measures.

## DAX Approach: Active Policy Count (the semi-additive measure)

This is the one measure on the Persistency Cohort page that cannot be a simple
`SUM` or `COUNTROWS` over a date range — a policy's "active" status is a *point-in-time
state*, not a value that accumulates across a period. Summing a naive "is this
policy active this month" flag across twelve months would count the same
long-lived policy twelve times.

The measure instead:
1. Anchors to a single snapshot date — the latest date present in the calendar
   table, independent of whatever filter context the visual provides.
2. Re-evaluates, from scratch, which policies were **issued on or before** that
   date **and have not lapsed as of** that date (a policy that hasn't reached its
   first renewal yet is still active by default; one whose most recent renewal
   outcome was `"grace"` or `"renewed"` also still counts; only `"lapsed"` removes
   it).
3. Returns a count of policies satisfying that condition *at the snapshot instant*
   — not a sum of monthly counts.

The practical test: the measure's value for a full year should equal its value
for December alone, not the sum of all twelve months. If a total row is roughly
12x a single month's figure, the measure has regressed to additive (summed)
behaviour.

## Known Limitations / What I'd Improve With More Time

- **`fact_target` ↔ `dim_date` relationship** is many-to-many due to the month-only
  grain of the target data. A dedicated month-grain bridge table would restore
  standard single-direction filtering and avoid the weaker filter propagation
  many-to-many relationships carry.
- **Row-Level Security** (`Territory Level Access` role, filtering via
  `dim_security`) has not been exhaustively tested with *View As* across every
  territory. The `dim_agent.territory` ↔ `dim_security` relationship cardinality
  should be double-checked if territories can have more than one assigned user.
- The Deneb bullet-chart visual on the Agent Performance Grid page does not yet
  provide click-to-highlight visual feedback (selection is wired for drillthrough,
  but the spec doesn't yet dim non-selected points).
- Given more time, the Persistency Cohort matrix would be extended with a
  product-category breakdown, since persistency is known to vary meaningfully by
  product type in this book of business.
