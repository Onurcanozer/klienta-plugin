# Reference — PPC Math: Make Every Recommendation Profit-Aware

A recommendation that ignores economics is a guess. This reference is the small set of calculations that turn account data into profit-aware decisions: what a conversion is allowed to cost, how much room you have to bid up, and whether spending more is worth it. The inputs Google can't know — **profit margin** and **lifetime value** — must come from the user; ask for them rather than assuming.

All metric pulls are read-only `run_gaql` (v21, validated live). The math is plain arithmetic; show your working to the user.

## Inputs you must get from the user

Google Ads knows clicks, cost, and conversions. It does **not** know what a sale is worth to the business. Before doing profitability math, ask for:

- **Average order value (AOV)** or revenue per conversion — if conversions don't already carry a value.
- **Gross margin %** — the share of revenue left after cost of goods/delivery. Break-even is meaningless without it.
- **Lifetime value (LTV)** and repeat behavior — only if the business has repeat purchases/subscriptions; otherwise treat the first sale as the whole value.

If the user can't give margin, say the profitability conclusion is unavailable and fall back to efficiency observations (CPA trend, impression share) without claiming a break-even.

## Break-even CPA — the ceiling on what a conversion may cost

```
Break-even CPA = AOV × gross margin %
```

This is the most you can pay for a conversion and still not lose money. A target CPA should sit *below* it to leave actual profit. Pull current CPA to compare:

```
SELECT campaign.name, metrics.cost_per_conversion, metrics.conversions, metrics.cost_micros
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.cost_micros DESC
```

**Decision it supports:** any campaign whose `cost_per_conversion` (÷1,000,000 for currency) is above break-even CPA is losing money on the margin — investigate or pull back. One comfortably below break-even has *headroom* (next).

## Headroom — how much room to scale or bid up

```
Headroom = Break-even CPA − current CPA
```

Positive headroom means each conversion currently earns margin, so paying a bit more per conversion (higher bids/budget) to win more volume can still be profitable — up to the point CPA approaches break-even. This is the economic justification for a budget step in `references/performance-triage.md`: raise budget on a converting, budget-constrained campaign *only when it has headroom*, and re-measure.

## ROAS and the value lens

When conversions carry value, use value directly:

```
SELECT campaign.name, metrics.cost_micros, metrics.conversions_value, metrics.conversions
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
```

```
ROAS = conversions_value ÷ cost
Break-even ROAS = 1 ÷ gross margin %   (e.g. 50% margin → break-even ROAS = 2.0)
```

**Decision it supports:** compare each campaign's ROAS to break-even ROAS. Above it = profitable; below = subsidizing sales. This is also why the conversion-value bidding strategies need real value data (see `references/conversion-tracking-first.md`) — without tracked value, ROAS is fiction.

## LTV:CAC — when first-sale break-even is too strict

If the business has repeat value, the conversion can justify a higher acquisition cost:

```
LTV = first-order value × (1 + expected repeat value over the customer's life, as a multiple)
LTV:CAC = LTV ÷ acquisition cost (CPA)
```

A healthy ratio means you can afford a CPA that looks unprofitable on the first order alone. Use this only with a real LTV figure from the user — never invent a repeat multiplier.

## Impression-share opportunity — sizing the upside before spending

Before recommending more budget, quantify what's actually available to capture:

```
SELECT campaign.name,
       metrics.search_impression_share,
       metrics.search_budget_lost_impression_share,
       metrics.search_rank_lost_impression_share,
       metrics.conversions, metrics.cost_per_conversion
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
```

**Decision it supports:** rough opportunity = current converting volume scaled by the share you're *losing to budget* (only meaningful if the campaign converts within break-even CPA). High budget-lost IS on a profitable campaign is the strongest case for scaling; high rank-lost IS is not a budget problem (fix quality/bids first — see `references/quality-score.md`). Present this as "there is room to roughly double impressions, currently lost to budget, on a campaign converting under break-even" — directional, grounded in the account's own numbers.

## Sample-size discipline

Don't compute CPA or ROAS conclusions off a handful of conversions. If a campaign has only a few conversions in the window, say the numbers are too thin to act on and either widen the date range or wait for more data. A confident recommendation on noise is worse than no recommendation.

## Calibrate against your own history

There are no industry numbers in this reference on purpose — we don't have a sourced benchmark set, and quoting one would be inventing data. The reliable baseline is the account's own history: compare a campaign to its prior periods and to other campaigns in the same account, and judge "good" relative to break-even CPA/ROAS (which come from the user's real margin), not an external table.
