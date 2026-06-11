# Reference — Budget Pacing, Reallocation & Seasonality

Budget is the throttle on everything else. This reference is how to read whether a campaign is starved or coasting, move money toward what earns it, and handle predictable demand swings — within our toolset, and honest about where the toolset stops. The only write tool here is `update_campaign_budget`; everything else is `run_gaql` reads and decisions. All queries validated against Google Ads API **v21**.

---

## When to use

- **A campaign is capped:** it converts profitably but runs out of budget before the day ends, leaving impressions on the table.
- **Money is mis-allocated:** a loser is funded while a winner is throttled.
- **A predictable demand swing is coming** (a sale, a holiday peak, an off-season trough) and you want bidding and budget to track it.

Don't touch budgets on noise. A single slow day, or a campaign with only a handful of conversions, isn't a pacing signal — apply the sample-size discipline from `references/ppc-math.md` before acting.

---

## Decision framework

**1. Is the constraint budget, or something else?** Budget-lost impression share answers this. High `search_budget_lost_impression_share` on a campaign converting *under* break-even CPA is the clean case for more budget. High `search_rank_lost_impression_share` is **not** a budget problem — that's quality/bid, and adding budget just spends more inefficiently (fix via `references/quality-score.md`). Separate the two before recommending a single euro.

**2. Reallocate before you add.** If the account's total budget is fixed, fund winners by pulling from losers, not by inventing new spend. The winner/loser test is profit-aware: capped *and* converting under break-even = winner (give it room); over break-even with no headroom = loser (pull back). This is the **budget-reallocation loop** in `references/optimization-loops.md` — run it weekly, one move at a time so you can attribute the effect (`references/change-impact.md`).

**3. Scale in steps, never lurches.** When a winner has headroom (`references/ppc-math.md`), raise its budget incrementally and re-measure, rather than doubling it overnight. On Smart Bidding campaigns a large, abrupt budget or target change can re-trigger the algorithm's learning period — performance gets noisy while it re-stabilises (Google, *How our bidding algorithms learn*). The headroom math sizes *whether* to scale; small steps protect *how*.

**4. Seasonality: let Smart Bidding handle the routine; flag the exceptional — honestly.** Smart Bidding already adapts to ordinary seasonal patterns. For a *major*, short, predictable conversion-rate swing (a 3-day sale, say), Google offers **seasonality adjustments** that tell Smart Bidding to expect the change. **We have no tool for this** — there is no seasonality-adjustment write in our surface. Be honest: that setting lives in the Google Ads UI, and the right move is to advise the user to create it there, scoped to the event. We *can* help by reading the historical trend (below) so the adjustment is grounded in real data, and by pre-positioning budget for the peak with `update_campaign_budget`.

---

## GAQL / tool examples

**Pacing — is budget the binding constraint?**

```
SELECT campaign.name,
       metrics.search_impression_share,
       metrics.search_budget_lost_impression_share,
       metrics.search_rank_lost_impression_share,
       metrics.conversions, metrics.cost_per_conversion, metrics.cost_micros
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.search_budget_lost_impression_share DESC
```

**Decision it supports:** the top rows are campaigns losing the most impressions *to budget*. Cross-check each against break-even CPA (`references/ppc-math.md`) — only raise budget where the campaign is both budget-capped and profitable.

**End-of-day exhaustion — does it spend out early?** Segment by hour to see the spend curve:

```
SELECT campaign.name, segments.hour, metrics.cost_micros, metrics.conversions
FROM campaign
WHERE campaign.id = <CAMPAIGN_ID>
  AND segments.date DURING LAST_14_DAYS
ORDER BY segments.hour
```

**Decision it supports:** if cost flatlines after midday, the budget is exhausting before evening demand — the campaign is missing later conversions. Raise the daily budget (if profitable) and re-check the curve.

**Act — change the budget (reversible):**

```
update_campaign_budget { campaignId: "<CAMPAIGN_ID>", dailyBudget: <new amount> }
```

Backed by the budget guardrail and a snapshot — reversible with `undo_change`. Confirm the new number with the user first; budget is the spend lever.

**Seasonality — read the trend so any advice is data-grounded:**

```
SELECT campaign.name, segments.date,
       metrics.conversions, metrics.conversions_from_interactions_rate,
       metrics.cost_micros
FROM campaign
WHERE campaign.id = <CAMPAIGN_ID>
  AND segments.date DURING LAST_90_DAYS
ORDER BY segments.date
```

**Decision it supports:** quantify how conversion rate actually moved around past instances of the event. If the user then sets a seasonality adjustment in the UI, that history — not a guess — is the basis for the magnitude they enter.

---

## Pitfalls

- **Adding budget to a rank-lost campaign.** Budget-lost vs rank-lost is the first split; conflating them wastes spend on a quality/bid problem. Always read both IS-lost columns.
- **Scaling on thin data or noise.** A few conversions, or one slow day, is not a pacing signal. Widen the window or wait (`references/ppc-math.md`).
- **Abrupt budget lurches on Smart Bidding.** Large sudden changes can reset learning and make the next week's numbers unreadable. Step, then measure.
- **Inventing seasonality magnitudes.** Don't quote a conversion-rate uplift you didn't read from the account. Google's own guidance: seasonality adjustments suit **short events of 1–7 days** and "may not work as well" for periods **longer than 14 days** (Google Ads Help, accessed 2026-06-11) — and they're for *major* expected swings only, since Smart Bidding handles routine seasonality. We can't set them; say so rather than implying we can.
- **Promising a seasonality tool we don't have.** The adjustment is UI-only for us. Advisory + trend-read is the honest scope.

---

## Sources

- Google Ads Help — *About seasonality adjustments*, accessed 2026-06-11. https://support.google.com/google-ads/answer/10369906 (ideal 1–7 day events; "may not work as well" beyond 14 days; supported campaign types/strategies; Smart Bidding already handles routine seasonality; for major expected conversion-rate changes only).
- Google Ads Help — *Create a seasonality adjustment*, accessed 2026-06-11. https://support.google.com/google-ads/answer/9352512 (the setting is created in the Google Ads UI; conversion-rate adjustment input).
- Google Ads Help — *How our bidding algorithms learn*, accessed 2026-06-11. https://support.google.com/google-ads/answer/10970825 (significant changes can re-enter a learning period).
- Tool surface: `src/tools/campaigns.ts` — `update_campaign_budget` (daily budget, guardrail + snapshot/undo). GAQL fields `metrics.search_budget_lost_impression_share`, `metrics.search_rank_lost_impression_share`, `segments.hour`, `metrics.conversions_from_interactions_rate` (Google Ads API v21).
- Related: `references/ppc-math.md` (break-even, headroom, impression-share opportunity), `references/optimization-loops.md` (budget-reallocation loop), `references/change-impact.md` (measure each move).
