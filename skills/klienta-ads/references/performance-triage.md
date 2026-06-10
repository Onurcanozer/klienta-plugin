# Reference — Performance Triage: Waste vs Budget vs Rank

A campaign that is "doing badly" is doing badly in one of three distinct ways, and each has a different fix. Mislabeling the problem wastes money. Always classify before recommending.

All queries below were validated against Google Ads API **v21** via `run_gaql`. Run them read-only first; let the numbers pick the diagnosis.

---

## Step 1 — Get the account picture

```
SELECT campaign.name, campaign.status, campaign.advertising_channel_type,
       metrics.cost_micros, metrics.clicks, metrics.conversions,
       metrics.ctr, metrics.average_cpc
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.cost_micros DESC
```

**Decision it supports:** where is the money going, and is it converting? Sort by spend and read top-down. A high-spend / zero-conversion row is the first thing to investigate. Divide `cost_micros` by 1,000,000 for the real amount.

---

## Step 2 — Waste: spend with nothing to show for it

```
SELECT campaign.name, metrics.cost_micros, metrics.conversions, metrics.clicks
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
  AND metrics.conversions = 0
  AND metrics.cost_micros > 0
ORDER BY metrics.cost_micros DESC
```

**Decision it supports:** which campaigns spent money and produced zero conversions over a meaningful window. Caveat: this is only trustworthy if conversion tracking is active — verify with the conversion-tracking-first gate before calling something "waste." If tracking is missing, the diagnosis is *invalid*, not "zero conversions."

Drill into keywords with impressions but poor engagement:

```
SELECT ad_group_criterion.keyword.text, metrics.ctr,
       metrics.impressions, metrics.clicks
FROM keyword_view
WHERE segments.date DURING LAST_30_DAYS
  AND metrics.impressions > 100
ORDER BY metrics.ctr ASC
```

**Decision it supports:** low-CTR keywords (lots of impressions, few clicks) signal a relevance mismatch — the keyword, ad copy, and intent are misaligned. Likely actions: tighten match type via `update_keyword`, add negatives (`add_negative_keywords`), or rewrite the RSA (`create_responsive_search_ad`). A low-CTR keyword is rarely fixed by raising its bid.

---

## Step 3 — Budget vs Rank: the two faces of "not enough traffic"

When a campaign could spend more but isn't, impression share tells you *why*. This single query separates the two bottlenecks:

```
SELECT campaign.name,
       metrics.search_impression_share,
       metrics.search_budget_lost_impression_share,
       metrics.search_rank_lost_impression_share
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
```

**Decision it supports — the core triage split:**

- **Budget-constrained** → `search_budget_lost_impression_share` is high. The campaign is profitable/healthy but capped by its daily budget. *Fix:* if (and only if) it converts at an acceptable cost, propose a **small** budget step with `update_campaign_budget` (e.g. +20%), then re-measure. Never jump the budget on a campaign you haven't confirmed is converting.
- **Rank-constrained** → `search_rank_lost_impression_share` is high. The campaign is losing auctions to quality/bid, not budget. *Fix:* raising the budget does nothing here. Look at Quality Score and bids/relevance instead:

```
SELECT ad_group_criterion.keyword.text,
       ad_group_criterion.quality_info.quality_score,
       metrics.impressions
FROM keyword_view
WHERE segments.date DURING LAST_30_DAYS
  AND metrics.impressions > 0
```

**Decision it supports:** low Quality Score keywords are the rank leak. Actions: improve ad relevance (new RSA), tighten keyword→ad→landing alignment, or adjust CPC with `update_keyword`. Pouring budget into a rank-constrained campaign just raises the cost of the same lost auctions.

---

## The triage in one line

> High budget-lost IS → it's a **budget** problem → consider `update_campaign_budget` (small step, only if converting).
> High rank-lost IS → it's a **quality/bid** problem → fix relevance / bids, not budget.
> Spend with zero conversions (tracking confirmed) → it's **waste** → pause, negate, or rewrite — smallest reversible move first.

Whatever you recommend, show the query and the numbers behind it, propose the smallest reversible action, and confirm before writing.
