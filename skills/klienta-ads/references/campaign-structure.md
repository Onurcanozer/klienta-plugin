# Reference — Campaign Structure & Naming

Structure is leverage: a well-organized account is easier to read, bid, and write ads for, and it earns better Quality Score because ads can stay relevant to tight keyword themes. This reference is how to organize Search campaigns and ad groups within our toolset — and an honest note on what restructuring we can and can't automate.

## The core principle: tight themes

The single most useful structural rule: **each ad group holds keywords that share one intent**, so its RSA can speak directly to them (`references/quality-score.md`, `references/rsa-writing.md`). A grab-bag ad group forces vague ads and drags expected-CTR and ad-relevance ratings down.

- **Campaign** = a budget + targeting + bidding boundary. Split into separate campaigns when you need *separate budgets or separate targeting* (e.g. different geos, or brand vs non-brand so brand traffic doesn't eat prospecting budget).
- **Ad group** = a tight keyword theme with its own ad(s). If you can't write one focused RSA that fits every keyword in the group, the group is too broad — split it.

How tight to go (a few specific keywords per group vs broader themed groups) is a trade-off, not a law: tighter groups give sharper relevance and control but more to manage; broader groups are simpler but vaguer. Lean tighter where relevance and CPC matter most, broader where volume is thin.

## Naming

Consistent names make an account legible at a glance and make `run_gaql` filtering easy. Pick a convention with the user and apply it uniformly — for example encoding the meaningful dimensions in order, like `Geo | Theme | MatchType` for campaigns and the keyword theme for ad groups. The exact scheme matters less than using it consistently. `update_campaign` can rename a campaign (name field); there's no ad-group rename tool, so set ad-group names correctly at creation (`create_ad_group`).

## Budget shape

Bias budget toward proven performers, but keep a slice for testing new themes and a buffer for scaling a winner — don't starve experiments or leave a profitable, budget-capped campaign capped (`references/ppc-math.md`). Treat any split as a hypothesis to measure (`references/change-impact.md`), not a one-way door.

## Honest limits — what we can and can't restructure

Our surface builds and edits **piece by piece**: `create_search_campaign`, `create_ad_group`, `add_keywords` / `bulk_add_keywords`, `create_responsive_search_ad`, plus the move/remove tools. There is **no bulk-restructure or move-keywords-between-ad-groups tool**, and no ad-group rename. So restructuring an existing account means rebuilding the target structure (new ad groups + keywords + ads) and pausing/removing the old — a deliberate, confirmed, multi-step job, not a one-click reshuffle.

Because of that, **scope restructures small and sequence them**: do one ad group at a time, verify each step (`run_gaql` read-back), and prefer pausing the old over removing it until the new structure proves out (removals aren't undoable). For a large reorganization, lay out the plan for the user and get approval before starting, rather than implying the account can be re-shaped in a single move.

## How tight is too tight — SKAG, STAG, and themed groups

The grouping question is really one decision: how many keywords share an ad group. Three named patterns sit on that spectrum.

- **SKAG (Single Keyword Ad Group)** — one keyword (across match types) per group. Maximal ad relevance, maximal management overhead.
- **STAG (Single Theme Ad Group)** — a handful of close synonyms expressing one intent. The current default for most accounts.
- **Themed / broad group** — many loosely related keywords. Simplest to run, vaguest ads.

**Decision framework: prefer STAG, reserve SKAG for proof, avoid grab-bags.** The historical case for SKAGs has weakened because exact match no longer means exact: Google matches exact-match keywords to close variants — misspellings, singular/plural and stemming, function-word changes, reordering, and same-meaning paraphrases (per Google Ads Help, "Keyword close variants," accessed June 2026). So two SKAGs built on near-identical terms now compete for the same queries, and which one serves is decided by Ad Rank, not your segmentation — the control SKAGs were meant to give is largely gone. Use STAG by default; escalate a single high-spend, high-intent keyword into its own group only when you have a genuinely distinct ad/landing message to justify it (`references/rsa-writing.md`). The test from the core principle still governs: **if one focused RSA can't speak to every keyword in the group, split it; if you're maintaining near-duplicate groups that serve interchangeably, consolidate them.**

The ceiling is not a structural concern: an account allows up to 10,000 campaigns, each campaign up to 20,000 ad groups, and each ad group up to 20,000 targeting items (keywords) (per Google Ads Help, "About your Google Ads account limits," accessed June 2026). Real structure decisions are driven by *legibility and relevance*, never by approaching those limits.

## Naming standard — make the account GAQL-legible

Names are the only free index you get. Because `run_gaql` filters on `campaign.name` and `ad_group.name` with `LIKE`, a disciplined convention turns the whole account into a queryable structure; an ad-hoc one forces you to eyeball every row.

**Pick one delimited convention and apply it without exception.** A workable campaign scheme encodes the dimensions you actually slice by, in fixed order — for example:

```
[Type] | [Brand|NonBrand] | [Geo] | [Theme] | [MatchType]
e.g.  Search | NonBrand | TR | running-shoes | Exact
```

The payoff is that any cross-section becomes one query. All non-brand campaigns:

```
SELECT campaign.name, metrics.cost_micros, metrics.conversions
FROM campaign
WHERE campaign.name LIKE '%| NonBrand |%'
  AND segments.date DURING LAST_30_DAYS
ORDER BY metrics.cost_micros DESC
```

Rules that keep it honest:

- **Fixed field order and a single delimiter** (`|` reads cleanly and rarely appears in real names). A convention only indexes if every campaign obeys it.
- **No live metrics in names** ("Best CPA", "Winner") — they go stale and lie. Encode *dimensions*, not *verdicts*.
- **Ad-group names get the keyword theme**, and get it right at creation: `create_ad_group` sets the name, and there is **no ad-group rename tool** (`update_campaign` renames campaigns only). A mis-named ad group can't be fixed in place — it's the rebuild path.

Agree the scheme with the user once, then enforce it on every `create_search_campaign` / `create_ad_group`. The exact fields matter less than that they never vary.

## Brand vs non-brand — separation and protection

Brand terms (your name, product names) and non-brand prospecting terms behave nothing alike: brand converts cheaply at high CTR and rates, non-brand is the expensive top-of-funnel work. **Put them in separate campaigns.** The reason is budget and measurement, not taste:

- A shared budget lets cheap brand clicks crowd out prospecting, or lets prospecting starve brand defense — separate campaigns give each its own `update_campaign_budget` lever.
- Blended reporting hides the truth: brand's low CPA flatters a non-brand campaign that's actually losing money. Separation lets `run_gaql` judge each on its own economics (`references/benchmark-calibration.md`).

Diagnose whether brand and non-brand are tangled in one campaign by reading where brand queries are landing:

```
SELECT campaign.name, ad_group.name, search_term_view.search_term,
       metrics.cost_micros, metrics.conversions
FROM search_term_view
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.cost_micros DESC
```

If brand search terms show up inside a prospecting campaign, that campaign is paying for traffic it didn't earn. **Brand protection** is the same problem from the spend side — non-brand or broad keywords poaching your own brand queries. The fix lives in the negative-keyword discipline, not here: add brand terms as negatives to non-brand campaigns so brand traffic is forced into the brand campaign (`references/shared-negatives.md`, `references/search-term-mining.md`). Note an honest limit: we can structure and add negatives, but we have no tool to file trademark complaints against competitors bidding on your brand — that's an off-platform Google Ads support process.

## Geographic and language structure

Geo and language are set **at campaign creation**: `create_search_campaign` takes `geoTargetIds` and `languageIds`. Resolve location names to IDs first with `search_geo_targets` (e.g. `['Istanbul','Ankara']` → constant IDs), then pass them in. Language IDs are constants (e.g. `1037` Turkish, `1000` English).

**Decision framework — split a campaign by geo only when the geo needs its own budget or bid level**, not for reporting alone:

- Different regions with different value, competition, or budget → separate campaigns, because budget and bidding are campaign-level boundaries (same rule as brand/non-brand).
- Same strategy, you only want a geo breakdown → keep one campaign and segment in reporting instead:
  ```
  SELECT campaign.name, geographic_view.country_criterion_id,
         metrics.cost_micros, metrics.conversions
  FROM geographic_view
  WHERE segments.date DURING LAST_30_DAYS
  ```

Language targeting should match the language of your ads and keywords — targeting a language you haven't written ads for wastes impressions. **Honest limit:** `update_campaign` only changes status and name; it does **not** edit geo or language targeting on an existing campaign. So a geo/language targeting change means the rebuild path below, the same as any other restructure — get it right at creation.

## Restructure red-flags — diagnose, then split or consolidate

Watch for these structural smells, each diagnosable with `run_gaql`:

- **Grab-bag ad groups** — one group, many unrelated search terms converting at wildly different costs. Symptom: high intra-group variance in `cost_per_conversion`. → **Split** by intent.
- **Near-duplicate groups** — several groups serving the same close-variant queries (visible when the same search term appears under multiple ad groups in `search_term_view`). → **Consolidate**; the segmentation buys nothing post-close-variants.
- **Brand bleed** — brand terms converting inside a non-brand campaign (the brand query above). → **Separate** + negatives.
- **One mega-campaign** — unrelated themes sharing a budget so the cheap theme starves the rest. Symptom: budget-constrained campaign (`references/optimization-loops.md`) where one ad group eats most spend. → **Split** by budget need.

Diagnostic pass — cost and conversion spread by ad group, to find the grab-bags and the starved themes:

```
SELECT campaign.name, ad_group.name, metrics.cost_micros,
       metrics.conversions, metrics.cost_per_conversion, metrics.ctr
FROM ad_group
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.cost_micros DESC
```

Then execute via the rebuild path from "Honest limits" above: new structure first (`create_ad_group` → `add_keywords` → `create_responsive_search_ad`), read back, pause the old, and only remove once the new structure proves out. **Pitfalls:** don't restructure on a thin sample (a few clicks isn't a verdict — `references/benchmark-calibration.md`); don't remove the old structure before the new one has data (removals aren't undoable); and don't split so far that no group accumulates enough conversions for its bidding to learn (`references/bidding-spectrum.md`).

## Sources

- Google Ads Help, "Keyword close variants" (close-variant matching: misspellings, plurals/stemming, function words, reordering, same-meaning paraphrases) — accessed June 2026: https://support.google.com/google-ads/answer/9342105
- Google Ads Help, "About your Google Ads account limits" (10,000 campaigns/account; 20,000 ad groups/campaign; 20,000 targeting items/ad group) — accessed June 2026: https://support.google.com/google-ads/answer/6372658
- Industry context on SKAG obsolescence post-close-variants (directional, not a sourced number): Marin Software, "SKAGs in the Time of Google Exact Match Close Variants"; Search Engine Journal, "Why [Google's keyword system] Is Becoming Obsolete."
