# Reference — Daily Operator Loop

A live account drifts every day: new search terms appear, budgets cap out, an ad gets disapproved, a conversion source breaks. The daily loop is a short, repeatable pass that catches problems while they're small. It is mostly reading; the few writes it produces are proposed and confirmed, never auto-applied.

Read-heavy, v21. Keep the whole pass to a handful of `run_gaql` calls (use the parallel `queries` form to fan out).

## The pass (read → triage → propose)

### 1. Read — one wide pull
Fan out the day's checks in a single `run_gaql` call with `queries`:
- **Spend & pacing** (campaign cost, conversions, CTR over the last 1–3 days vs the prior period) — is anything spending sharply more or converting sharply less?
- **Budget-capped campaigns** (`metrics.search_budget_lost_impression_share`) — which converting campaigns ran out of budget?
- **Yesterday's search terms** (`search_term_view`, last 1–2 days, sorted by cost) — new wasteful queries?
- **Disapprovals / non-serving ads** (`ad_group_ad.policy_summary.approval_status`, `ad_group_ad.status`) — anything blocked from serving?
- **Conversion tracking pulse** (`customer.conversion_tracking_setting.conversion_tracking_status`) — still active?
- **Recent changes** (`get_changes`) — what did you or anyone else change in the last day, for context.

### 2. Triage — sort into three buckets
- **Act today** — clear, small, reversible: a junk search term to negate, a disapproved ad to look at, a budget-capped profitable campaign. 
- **Watch** — a metric moving but not yet conclusive; note it, re-check tomorrow (see `references/change-impact.md`).
- **Ignore** — noise within normal day-to-day variation. Don't manufacture work; a quiet day is a valid outcome.

### 3. Propose — confirmed writes only
For each "act today" item, propose the smallest reversible action with the evidence behind it, and wait for approval:
- Junk search terms → `add_negative_keywords` (or `add_keywords_to_negative_list` if you keep a shared block list); use the n-gram pass (`references/search-term-mining.md`) when the waste is a pattern.
- Budget-capped & profitable (has headroom, `references/ppc-math.md`) → a small `update_campaign_budget` step.
- A clearly bad keyword → `update_keyword` to PAUSED, or `bulk_pause_keywords` for several.
- A disapproved ad → inspect; a fix is usually a new `create_responsive_search_ad`, not a tool flick.

## Discipline
- **Daily ≠ daily changes.** Most days produce zero writes. The value is catching the day that does need action.
- **Don't tune on noise.** One bad day of CPA is not a trend; bid/budget thrashing hurts more than it helps.
- **Confirm and verify.** Every write is approved first and read back after (the operating contract's rules 4–5). Use `get_changes` so tomorrow's pass can see today's actions.
- **Escalate, don't guess.** A disapproval reason you don't understand or a tracking outage is a "tell the user" item, not a silent fix.
