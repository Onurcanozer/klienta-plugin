# Reference — Quality Score: Components & Component-Level Remediation

Quality Score (QS) is Google's 1–10 estimate of how relevant your keyword, ad, and landing page are to a search. It is not a vanity metric: a higher QS lowers the price you pay for the same ad position, so a weak QS is a tax on every click. This reference is about reading the *three components* behind the number and fixing the specific one that is weak — not chasing the score itself.

All queries below are read-only `run_gaql` against Google Ads API **v21**. Validated against a live account.

## The three components

Google exposes QS as one score plus three component ratings (each reported as `BELOW_AVERAGE`, `AVERAGE`, or `ABOVE_AVERAGE`):

- **Expected click-through rate** (`search_predicted_ctr`) — how likely the ad is to be clicked when it shows for this keyword, normalized for position. A weak rating here is a *relevance/appeal* problem at the keyword↔ad level.
- **Ad relevance** (`creative_quality_score`) — how closely the ad text matches the intent of the keyword. A weak rating means the keyword and the ad are talking about different things.
- **Landing page experience** (`post_click_quality_score`) — how relevant and usable the destination is for someone who searched this term. This is the only component that lives *off* the ad surface.

The single GAQL pass that exposes all of it:

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

**Decision it supports:** rank keywords by impressions (where QS costs you the most money), then read *which component* is `BELOW_AVERAGE`. Two keywords can both score 4/10 for completely different reasons and need opposite fixes — never prescribe a remedy from the composite number alone.

## Why the score moves the price

QS feeds Ad Rank, and Ad Rank determines both whether you show and what you pay. The practical takeaway is directional, not a formula to quote: improving a genuinely weak component tends to reduce the cost of the auctions you were already winning and can unlock auctions you were losing on rank. Confirm the effect on the account itself — pair this with `references/change-impact.md` and compare cost-per-click for the affected keywords before and after the fix. Do not promise a specific percentage; QS-to-CPC behavior is account- and auction-specific.

## Component-level remediation

Read the weak component, then act on *that* component:

- **Expected CTR is `BELOW_AVERAGE`** → the keyword isn't earning clicks at its position. Tighten the keyword↔ad match: split the keyword into a more specific ad group so the ad can speak directly to it (`create_ad_group` + `add_keywords` + a dedicated `create_responsive_search_ad`), and consider a tighter match type via `update_keyword`. If the keyword is broadly off-intent, it may belong on a negative list instead (`add_negative_keywords`) — see `references/search-term-mining.md`.
- **Ad relevance is `BELOW_AVERAGE`** → the ad copy doesn't echo the keyword's intent. Write a new RSA whose headlines include the themes of the keywords in that ad group (`create_responsive_search_ad`). If one ad group holds many unrelated keywords, the fix is structural: break it up so each ad group has a tight theme its ad can match.
- **Landing page experience is `BELOW_AVERAGE`** → the destination is the problem, and it lives outside this toolset. Surface it as a recommendation for the user's site/dev team (message match to the ad, load speed, mobile usability, a clear next action). You can point the ad at a more relevant existing page via the ad's final URL when one fits, but on-page changes are off-platform — say so plainly rather than implying you can edit the site.

## Calibrate against your own account, not benchmarks

There is no universal "good QS." Instead, baseline within the account: pull the query above for the whole account, look at the distribution, and treat the high-impression keywords sitting below the account's own median as the priority list. Re-pull after each remediation and watch the component rating flip from `BELOW_AVERAGE` toward `AVERAGE`/`ABOVE_AVERAGE`. That movement — measured on this account's own data — is the evidence, not an external number.

> Note: QS is a diagnostic lens, not a goal. The goal is profitable conversions (see `references/ppc-math.md`); QS earns its place only when a weak component is demonstrably raising the cost of traffic you want.
