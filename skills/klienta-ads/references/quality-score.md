# Reference — Quality Score & Ad Rank: Components, Auction Mechanics, Remediation

Quality Score (QS) is Google's 1–10 estimate of how relevant your keyword, ad, and landing page are to a search. It is not a vanity metric: a higher QS lowers the price you pay for the same ad position, so a weak QS is a tax on every click. This reference is about reading the *three components* behind the number and fixing the specific one that is weak — not chasing the score itself — plus the auction layer above it (Ad Rank) and the diagnostic next to it (Ad Strength), so the three don't get conflated.

All queries below are read-only `run_gaql` against Google Ads API **v21**. Validated against a live account.

## When to use

- A keyword's CPC looks expensive for its position, or impression share is lost to **rank** (`metrics.search_rank_lost_impression_share` high while budget-lost is low — see `references/ppc-math.md`): QS is the first suspect.
- The user asks "why is my ad not showing?" or "why am I paying more than competitors?" — both are Ad Rank questions, and QS is the half of Ad Rank you can actually fix without paying more.
- After structural changes (new ad groups, new RSAs): re-read components to confirm the fix landed.
- **Not** when the problem is budget, tracking, or targeting — run `references/performance-triage.md` first to confirm rank/quality is actually the binding constraint.

## Decision framework

### The three components

Google exposes QS as one score plus three component ratings (each reported as `BELOW_AVERAGE`, `AVERAGE`, or `ABOVE_AVERAGE`):

- **Expected click-through rate** (`search_predicted_ctr`) — how likely the ad is to be clicked when it shows for this keyword, normalized for position. A weak rating here is a *relevance/appeal* problem at the keyword↔ad level.
- **Ad relevance** (`creative_quality_score`) — how closely the ad text matches the intent of the keyword. A weak rating means the keyword and the ad are talking about different things.
- **Landing page experience** (`post_click_quality_score`) — how relevant and usable the destination is for someone who searched this term. This is the only component that lives *off* the ad surface.

Two keywords can both score 4/10 for completely different reasons and need opposite fixes — never prescribe a remedy from the composite number alone.

### Ad Rank — the auction layer QS feeds

QS matters because it feeds Ad Rank, and Ad Rank decides whether you show, where, and what you pay. Google lists the Ad Rank factors as: **your bid amount; the quality of your ads and landing page; the Ad Rank thresholds; the competitiveness of the auction; the context of the search** (location, device, time, the nature of the search terms, other ads and results on the page, other user signals); **and the expected impact of assets and other ad formats** (per Google Ads Help, "About Ad Rank", accessed 2026-06). Ad Rank is recalculated **twice per auction** — once for eligibility against the thresholds, once for positioning against competitors (same source). Practical readings of that factor list:

- **Bid × quality is the core trade.** Google states explicitly that an advertiser with highly relevant keywords and ads can win a higher position at a lower cost than a higher-bidding competitor (same source). This is the economic case for fixing a weak component instead of raising bids: quality buys rank without buying clicks at a premium.
- **Thresholds explain "eligible but not showing."** A keyword can be ENABLED and approved yet sit out auctions because its Ad Rank doesn't clear the threshold for that query context. Symptom in data: very low impressions with healthy status fields, high `search_rank_lost_impression_share`. The fix is the same as for weak QS — quality or bid — not status debugging.
- **Assets affect rank.** "Expected impact of assets and other ad formats" is a listed factor, so populated sitelinks/callouts are not cosmetic — they participate in the rank calculation. Our write surface: `create_sitelink_asset`, `create_callout_asset`. When rank is marginal, shipping real assets is a cheap, legitimate lever.
- **Context means rank is per-auction, not a property of the keyword.** Position moving around during the day is normal; judge trends over windows, not single auctions.

### Component-level remediation

Read the weak component, then act on *that* component:

- **Expected CTR is `BELOW_AVERAGE`** → the keyword isn't earning clicks at its position. Tighten the keyword↔ad match: split the keyword into a more specific ad group so the ad can speak directly to it (`create_ad_group` + `add_keywords` + a dedicated `create_responsive_search_ad`), and consider a tighter match type via `update_keyword`. If the keyword is broadly off-intent, it may belong on a negative list instead (`add_negative_keywords`) — see `references/search-term-mining.md`.
- **Ad relevance is `BELOW_AVERAGE`** → the ad copy doesn't echo the keyword's intent. Write a new RSA whose headlines include the themes of the keywords in that ad group (`create_responsive_search_ad`, with the asset-pool craft in `references/rsa-writing.md`). If one ad group holds many unrelated keywords, the fix is structural: break it up so each ad group has a tight theme its ad can match.
- **Landing page experience is `BELOW_AVERAGE`** → the destination is the problem, and it lives outside this toolset. Surface it as a recommendation for the user's site/dev team (message match to the ad, load speed, mobile usability, a clear next action). You can point the ad at a more relevant existing page via the ad's final URL when one fits, but on-page changes are off-platform — say so plainly rather than implying you can edit the site.

### Ad Strength vs Quality Score — don't conflate them

They sound alike and get mixed up constantly; they answer different questions:

| | Quality Score | Ad Strength |
|---|---|---|
| Scope | per **keyword** (keyword↔ad↔LP) | per **ad** (the RSA/RDA asset pool) |
| Basis | historical auction performance | best-practice conformance of the assets (relevance, quality, diversity) |
| Role | feeds Ad Rank → price and position | **feedback tool only** — Google's "About Ad Strength" page describes it as guidance for improving creatives and makes no claim that it enters the auction (per Google Ads Help, accessed 2026-06) |
| Ratings | 1–10 + three component ratings | `POOR` / `AVERAGE` / `GOOD` / `EXCELLENT` (a pending state can show before it's computed) |

Practical rule: a `POOR` Ad Strength is a *pre-flight* warning that the asset pool lacks variety — fix it at authoring time (`references/rsa-writing.md`). A weak QS component is a *post-flight* verdict from real auctions. Improving Ad Strength is worthwhile — Google reports advertisers improving RSA Ad Strength from "Poor" to "Excellent" see on average 15% more conversions (per Google Ads Help, "About Ad Strength for responsive search ads", accessed 2026-06) — but don't report Ad Strength to the user as if it were QS, and don't expect a better Ad Strength label to directly lower CPCs the way a fixed QS component does.

### Calibrate against your own account, not benchmarks

There is no universal "good QS." Instead, baseline within the account: pull the component query below for the whole account, look at the distribution, and treat the high-impression keywords sitting below the account's own median as the priority list. Re-pull after each remediation and watch the component rating flip from `BELOW_AVERAGE` toward `AVERAGE`/`ABOVE_AVERAGE`. That movement — measured on this account's own data — is the evidence, not an external number. The same applies to the price effect: improving a genuinely weak component tends to reduce the cost of auctions you were already winning and can unlock auctions you were losing on rank, but confirm it on the account itself — pair with `references/change-impact.md` and compare CPC for the affected keywords before and after. Do not promise a specific percentage; QS-to-CPC behavior is account- and auction-specific.

## GAQL/tool examples

**1. The single pass that exposes QS and all three components:**

```
SELECT ad_group_criterion.keyword.text,
       ad_group_criterion.quality_info.quality_score,
       ad_group_criterion.quality_info.creative_quality_score,
       ad_group_criterion.quality_info.search_predicted_ctr,
       ad_group_criterion.quality_info.post_click_quality_score,
       metrics.impressions
FROM keyword_view
WHERE metrics.impressions > 0
ORDER BY metrics.impressions DESC
```

**Decision it supports:** rank keywords by impressions (where QS costs you the most money), then read *which component* is `BELOW_AVERAGE` and apply that component's remediation above.

**2. Is the problem rank at all?** (run before doing QS work)

```
SELECT campaign.name,
       metrics.search_rank_lost_impression_share,
       metrics.search_budget_lost_impression_share,
       metrics.average_cpc
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
```

**Decision it supports:** high rank-lost IS with low budget-lost IS = quality/bid problem → this reference. The reverse = budget problem → `references/ppc-math.md`, not QS work.

**3. Ad Strength across the account** (authoring-quality sweep):

```
SELECT campaign.name, ad_group.name,
       ad_group_ad.ad.id, ad_group_ad.ad_strength, ad_group_ad.status
FROM ad_group_ad
WHERE ad_group_ad.status = 'ENABLED'
```

**Decision it supports:** list `POOR`/`AVERAGE` ads as candidates for a rebuilt asset pool via `create_responsive_search_ad` (then pause the old ad with `update_ad_status`). This is the pre-flight sweep; query 1 is the post-flight verdict.

## Pitfalls

- **Chasing the composite number.** Raising "QS 4 → 7" is not a goal; profitable conversions are (see `references/ppc-math.md`). QS work earns its place only when a weak component is demonstrably raising the cost of traffic you want.
- **Prescribing from the composite.** Always read the three components first; the same 4/10 can need a new ad, a new structure, or a landing-page conversation.
- **Treating Ad Strength as a ranking lever.** It's authoring feedback, not an auction input (sourced above). Fix `POOR` ads for the variety it signals, not to "raise rank" directly.
- **Promising CPC drops.** The bid×quality trade is real and sourced, but the magnitude is account-specific — verify with before/after CPC on the affected keywords (`references/change-impact.md`), never quote a percentage upfront.
- **QS work on null data.** Keywords with no recent impressions report stale or absent `quality_info` — that's why query 1 filters `metrics.impressions > 0`. Diagnose where money actually flows.
- **Ignoring assets when rank is marginal.** Sitelinks/callouts are listed Ad Rank factors; an account with zero assets is leaving free rank on the table — ship them before recommending bid raises.

## Sources

- Google Ads Help — "About Ad Rank": https://support.google.com/google-ads/answer/1752122 (accessed 2026-06). Factor list (bid, ad/LP quality, thresholds, competitiveness, search context, expected asset impact), twice-per-auction recalculation, better-quality-can-beat-higher-bid statement.
- Google Ads Help — "About Ad Strength": https://support.google.com/google-ads/answer/9142254 (accessed 2026-06). What Ad Strength measures (relevance/quality/diversity of assets), Poor→Excellent scale, feedback-tool positioning.
- Google Ads Help — "About Ad Strength for responsive search ads": https://support.google.com/google-ads/answer/9921843 (accessed 2026-06). Average 15% more conversions improving RSA Ad Strength from Poor to Excellent.
- Live v21 validation (this project): `quality_info` component fields and the `keyword_view` query shape; `ad_group_ad.ad_strength` field.
- Cross-references: `references/rsa-writing.md` (asset-pool craft), `references/search-term-mining.md` (negatives), `references/ppc-math.md` (economics, impression-share), `references/change-impact.md` (before/after verification), `references/performance-triage.md` (is rank even the problem?).
