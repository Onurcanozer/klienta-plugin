# Reference — The Bidding Spectrum: From Manual CPC to Target ROAS

Bidding strategy is the single highest-leverage setting in a Google Ads account: it decides what the auction optimizes *for*. Most bidding mistakes are not bad targets — they are strategies chosen before the account has the conversion data to feed them. This reference is the full spectrum (manual_cpc → maximize clicks → maximize conversions → target CPA → maximize conversion value → target ROAS), when to move along it, how to calibrate targets from real economics, and how to make the transition without blowing up performance.

Write surface: `create_search_campaign` (campaign-level strategy at creation), and the portfolio tools `create_bidding_strategy`, `update_bidding_strategy`, `link_campaign_to_bidding_strategy`, `remove_bidding_strategy`. Diagnostics are read-only `run_gaql` against Google Ads API **v21**. Portfolio payload shapes and the unlink mask behavior below were validated against a live v21 account.

## When to use

Reach for this reference when:

- The user asks "should I use smart bidding?", "switch to target CPA/ROAS?", or "why is Google spending my budget badly?"
- A campaign is on `MANUAL_CPC` or `MAXIMIZE_CLICKS` and has been converting steadily — it may be leaving smart-bidding value on the table.
- A campaign on a conversion-based strategy is starving (few impressions, low spend) or hemorrhaging (high spend, CPA far above target) — the target or the signal is usually wrong, not the strategy.
- Several campaigns share one economic goal (same product line, same margin) — a **portfolio** strategy can pool their conversion data instead of each campaign learning alone.
- Performance dipped right after a strategy or target change — see the learning-period section before reverting anything.

**Hard prerequisite:** every conversion-based strategy (maximize conversions, tCPA, maximize conversion value, tROAS) requires working conversion tracking. Run the gate in `references/conversion-tracking-first.md` first. No tracking → the only honest recommendations are `MANUAL_CPC` or `MAXIMIZE_CLICKS` plus a tracking setup plan. This is enforced by the API itself: conversion-based portfolio strategies fail on accounts without conversion tracking — a prerequisite we confirmed against a live v21 account, and which the `create_bidding_strategy` tool description states explicitly.

## Decision framework

### The spectrum

Each step right hands Google more pricing freedom in exchange for optimizing a goal closer to profit. More freedom is only good when the conversion signal is strong enough to steer it.

| Strategy | Optimizes for | Needs | Our write path |
|---|---|---|---|
| `MANUAL_CPC` | Nothing — you set every bid | Nothing (works with zero data) | `create_search_campaign biddingStrategy:MANUAL_CPC`, bids via `bulk_update_bids` |
| `MAXIMIZE_CLICKS` | Most clicks within budget | Nothing | `create_search_campaign` (its default) |
| `MAXIMIZE_CONVERSIONS` | Most conversions within budget | Tracking ON, some conversion history | campaign-level at create, or portfolio type `MAXIMIZE_CONVERSIONS` |
| Target CPA | Conversions at a set cost | Steady conversion volume | portfolio type `TARGET_CPA` (also as a cap on `MAXIMIZE_CONVERSIONS`) |
| `MAXIMIZE_CONVERSION_VALUE` | Most conversion *value* within budget | Tracking with real values | campaign-level at create, or portfolio |
| Target ROAS | Value at a set return | Steady volume of *valued* conversions | portfolio type `TARGET_ROAS` (also as a target on `MAXIMIZE_CONVERSION_VALUE`) |

### Selection tree

Walk it top-down; stop at the first "no".

1. **Is conversion tracking live and trustworthy?** (gate query in `references/conversion-tracking-first.md`). **No** → `MANUAL_CPC` if the user wants tight control or the account is tiny; `MAXIMIZE_CLICKS` if the goal is traffic while tracking gets built. Stop here.
2. **Does the account have meaningful conversion history?** Smart bidding learns from conversions; with a trickle, its predictions are noise. Google's own eligibility floor for target ROAS on Search and Shopping is **at least 15 conversions in the past 30 days at the conversion-tracking level** (per Google Ads Help, "About Target ROAS bidding", accessed 2026-06). Below any meaningful volume → stay on `MAXIMIZE_CLICKS` (or `MANUAL_CPC`) and concentrate budget until conversions accumulate. Thin data is the most common reason smart bidding "doesn't work".
3. **Do conversions carry real, accurate values?** **No / all conversions worth roughly the same** (typical lead-gen) → the conversion-count branch: `MAXIMIZE_CONVERSIONS` first; add a **target CPA** once there's enough history to know what a conversion should cost. **Yes, values are real and differ between conversions** (e-commerce, mixed product lines) → the value branch: `MAXIMIZE_CONVERSION_VALUE` first; add a **target ROAS** once value history is stable. Google's guidance for value-based bidding: report values for **4 weeks or 3 conversion cycles, whichever is longer**, before setting a ROAS target (per Google Ads Help, "About Target ROAS bidding", accessed 2026-06).
4. **Does the user have a hard efficiency constraint?** A fixed allowable cost per lead → target CPA. A required return on spend → target ROAS. No hard constraint, just "get me the most" → stay on the un-targeted maximize variant; an arbitrary target only handcuffs the algorithm.
5. **Do several campaigns share the same goal and economics?** → make the target a **portfolio** strategy and link the campaigns, so they pool learning data instead of each clearing the volume bar alone. Different margins or different goals → separate strategies (one blended target mis-prices both sides).

### Field notes per strategy

- **`MANUAL_CPC`** — you own every bid; change them with `update_keyword` (cpcBid) or `bulk_update_bids` (max 100 per call, guardrail CPC caps apply to every item). Right when data is too thin for automation or the user insists on control; wrong as a permanent home for a converting account, because it cannot price each auction individually. Note that **Enhanced CPC is retired**: Google removed eCPC from Search and Display campaigns as of March 2025, migrating remaining users to plain manual CPC (per Search Engine Land / Google announcement, 2024; rollout completed March 2025). Don't recommend eCPC as a stepping stone — the modern step after manual is `MAXIMIZE_CLICKS` or straight to a conversion strategy. (Our unlink path sets `enhancedCpcEnabled: false` explicitly for exactly this reason.)
- **`MAXIMIZE_CLICKS`** (`targetSpend`) — fills the budget with the cheapest clicks available. Good for traffic-building and data accumulation phases; dangerous left unattended, because the cheapest clicks are often cheap for a reason. Pair it with the search-term discipline in `references/search-term-mining.md` while it runs.
- **`MAXIMIZE_CONVERSIONS`** — most conversions the budget can buy, no cost discipline unless you add a `targetCpa` cap (our `create_bidding_strategy` accepts it as an optional cap). Uncapped, it will happily spend the full budget even when marginal conversions are expensive — fine while learning, worth capping once you know the economics.
- **Target CPA** — the cap becomes the goal: average cost per conversion ≈ target. Some conversions cost more, some less; Google aims for the average (per Google Ads Help, "About Target CPA bidding", accessed 2026-06). Judge it on the average over a ≥30-conversion window, not on individual expensive conversions.
- **`MAXIMIZE_CONVERSION_VALUE`** — value-weighted twin of maximize conversions; requires real values on conversions or it degenerates into maximize conversions with extra steps. Optional `targetRoas` turns it into tROAS behavior.
- **Target ROAS** — strongest discipline, hungriest for data (the value signal must be both plentiful and accurate — see signal hygiene). The strictest eligibility floors of any strategy (15 valued conversions/30 days for Search/Shopping, sourced above) exist because it is the easiest strategy to starve with a bad target.

### Campaign-level vs portfolio — and what our tools can actually change

- **At creation**, `create_search_campaign` sets a campaign-level strategy: `MANUAL_CPC`, `MAXIMIZE_CLICKS` (default), `MAXIMIZE_CONVERSIONS`, or `MAXIMIZE_CONVERSION_VALUE`. PMax campaigns are created with `maximizeConversions` or `maximizeConversionValue` by `create_performance_max_campaign`; Display via `create_display_campaign`.
- **Mid-life**, `update_campaign` changes only status and name — it does **not** rewrite a campaign-level strategy. The supported way to change a live campaign's bidding with our tools is the portfolio path: `create_bidding_strategy` → `link_campaign_to_bidding_strategy`. This is honest scope: if the user wants a campaign-level (non-portfolio) tCPA added to an existing campaign, say that our path is the portfolio equivalent, which Google treats the same way at auction time.
- **Unlinking** (`link_campaign_to_bidding_strategy` with `unlink:true`) returns the campaign to a standard strategy — `MANUAL_CPC` (default) or `MAXIMIZE_CLICKS`. Conversion-based standard strategies aren't offered on unlink by design; keep those as portfolios.
- A campaign's `biddingStrategy` (portfolio link) and its own standard strategy live in the same oneof — linking replaces whatever was there, and unlinking must explicitly set a new standard strategy. There is no "remove the portfolio and keep what was before"; decide the landing strategy before unlinking.

### Calibrating the target — economics first, history second

Both numbers come from `references/ppc-math.md`; get **margin** (and AOV) from the user, never invent them.

- **Target CPA ceiling = break-even CPA = AOV × gross margin %.** The working target should sit *below* break-even to leave profit. Google's recommended starting point is performance history: its suggested target is "the average CPA from the last 30 days, adjusted for any conversion delays" (per Google Ads Help, "About Target CPA bidding", accessed 2026-06). The right opening target is therefore **near the campaign's own recent average CPA, capped by break-even** — not the number the user wishes were true. If recent CPA is far above break-even, the campaign has a profitability problem no bid strategy fixes; route to `references/performance-triage.md`.
- **Target ROAS floor = break-even ROAS = 1 ÷ gross margin %** (50% margin → 2.0). The working target sits *above* the floor. Google expresses tROAS as a percent — conversion value ÷ cost × 100 (per Google Ads Help, "About Target ROAS bidding", accessed 2026-06) — but our `create_bidding_strategy`/`update_bidding_strategy` take a **ratio**: `targetRoas: 4.0` means 400%. Confirm which convention the user is quoting before setting anything; a 4× error in either direction is catastrophic.
- **Set the target from history, then walk it toward the goal.** Open at (or slightly easier than) the recent actual CPA/ROAS, let a conversion cycle pass, then tighten in steps via `update_bidding_strategy`. Jumping straight to an aspirational target is the classic volume-killer (see Pitfalls).

### Signal hygiene — smart bidding is only as good as its food

Smart bidding does not see your business; it sees the conversion stream. Three properties decide whether that stream is trustworthy:

1. **Volume.** Google recommends evaluating performance over windows containing **at least 30 conversions** (per Google Ads Help, "About Target CPA bidding", accessed 2026-06), and won't even offer Search/Shopping tROAS below 15 conversions/30 days (source above). Pull the actual count per campaign (query below) before recommending any conversion-based strategy — and before judging one.
2. **Delay.** If conversions land days after the click (offline uploads, long sales cycles), the algorithm's recent picture is systematically incomplete, and so is yours. Google defines the conversion cycle as the click-to-conversion time and recommends waiting **at least one conversion cycle** before evaluating changes (per Google Ads Help, "How our bidding algorithms learn", accessed 2026-06). Measure the account's own lag with the `conversion_lag_bucket` query below; never judge a window shorter than the lag.
3. **Value accuracy.** Value-based strategies optimize whatever numbers you feed them. Default values stamped on every conversion (`alwaysUseDefaultValue`), missing currency codes, or duplicate offline uploads (no `orderId` dedup) all corrupt the signal — see `references/conversion-design.md`. If values are fiction, tROAS is fiction; say so and route to the count-based branch instead.

If any of the three is weak, the recommendation is **not** "try smart bidding and see" — it is to fix the signal first. This is the same gate philosophy as `references/conversion-tracking-first.md`, one level deeper.

### Learning periods and transition safety

Strategy changes cause turbulence; the job is to keep the user from panicking into a revert that resets everything.

- **A strategy change restarts learning; expect volatility "for a conversion cycle or two"** before judging it (per Google Ads Help, "How our bidding algorithms learn", accessed 2026-06). Tell the user *before* the switch that the first days may look worse — pre-warned users don't force a revert on day 3.
- **A target change does NOT reset learning.** Google states plainly that changing a target won't trigger a learning status and won't reset what Smart Bidding has learned, and bids adjust quickly (same source). Practical consequence: prefer *adjusting the target* on an existing strategy (`update_bidding_strategy`) over *swapping strategies* — the first is cheap, the second restarts the meter.
- **Move targets in steps, not leaps.** Walk tCPA down (or tROAS up) by modest increments per conversion cycle, watching volume at each step. The algorithm responds to targets continuously; a huge jump is just a slow-motion strategy swap.
- **Don't stack changes.** A new strategy plus a budget change plus new ads in the same week makes attribution of the result impossible. One change, one measurement window — then the next. Use `references/change-impact.md` to measure before/after, with the window sized to the conversion lag.
- **Transition checkpoint list:** (1) gate open (tracking live), (2) volume bar met on the account's own numbers, (3) target calibrated from history + break-even, (4) user warned about the turbulence window, (5) measurement date agreed. Only then write.

## GAQL/tool examples

**1. What is each campaign running today?** (always the first pull — never assume)

```
SELECT campaign.name, campaign.status,
       campaign.bidding_strategy_type,
       campaign.bidding_strategy,
       campaign.maximize_conversions.target_cpa_micros,
       campaign.maximize_conversion_value.target_roas,
       campaign.target_cpa.target_cpa_micros,
       campaign.target_roas.target_roas
FROM campaign
WHERE campaign.status != 'REMOVED'
```

**Decision it supports:** `bidding_strategy_type` tells you where each campaign sits on the spectrum; a non-null `campaign.bidding_strategy` resource name means it's linked to a portfolio. Targets in micros ÷ 1,000,000 for currency.

**2. Is the strategy healthy, learning, or limited?** — the system-status diagnostic

```
SELECT campaign.name,
       campaign.bidding_strategy_type,
       campaign.bidding_strategy_system_status
FROM campaign
WHERE campaign.status = 'ENABLED'
```

`campaign.bidding_strategy_system_status` is Google's own verdict on the strategy, exposed for both standard and portfolio strategies (per Google Ads API docs, "Bidding Strategy Status", developers.google.com, accessed 2026-06). Read it in three families:

- **`ENABLED`** — active, no issues detected. The strategy's results can be judged on their merits.
- **`LEARNING_*`** (e.g. `LEARNING_NEW` after creation/reactivation, `LEARNING_SETTING_CHANGE` after a settings change) — the algorithm is recalibrating. **Do not judge performance or make further changes while a learning status shows**; that's the turbulence window from the framework section, now visible as a field instead of a guess.
- **`LIMITED_*` / `MISCONFIGURED_*`** — the strategy is constrained (by data, budget, or bid ceilings) or broken by configuration. A limited status is the API telling you *which* pitfall you're in: data-limited → the volume bar isn't met (go back to the selection tree); budget-limited → the budget, not the strategy, is the binding constraint (see `references/ppc-math.md` impression-share section); ceiling-limited → a bid cap is fighting the automation. Fix the named constraint before touching the target.

**Decision it supports:** this single query separates "the strategy is bad" from "the strategy is still learning" from "something else is strangling it" — three situations that look identical in the performance numbers and need opposite responses.

**3. Is the volume bar met?** (conversions per campaign, last 30 days)

```
SELECT campaign.name, metrics.conversions, metrics.conversions_value,
       metrics.cost_micros, metrics.cost_per_conversion
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.conversions DESC
```

**Decision it supports:** compare `metrics.conversions` against the thresholds in the selection tree, and `cost_per_conversion` against break-even CPA from `references/ppc-math.md`. Campaigns individually under the bar but sharing economics → portfolio candidates (pool them).

**4. How long is the conversion lag?** (sets your measurement window)

```
SELECT segments.conversion_lag_bucket, metrics.conversions
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
```

**Decision it supports:** if a meaningful share of conversions sits in multi-day buckets, recent-window CPA/ROAS reads are systematically pessimistic, and the post-change evaluation window must be at least one full lag cycle.

**5. Inspect existing portfolio strategies**

```
SELECT bidding_strategy.id, bidding_strategy.name, bidding_strategy.type,
       bidding_strategy.status, bidding_strategy.campaign_count,
       bidding_strategy.target_cpa.target_cpa_micros,
       bidding_strategy.target_roas.target_roas
FROM bidding_strategy
WHERE bidding_strategy.status != 'REMOVED'
```

**6. Portfolio lifecycle with our tools** (each write: show the user, confirm, then call)

```
create_bidding_strategy   { customerId, name: "tCPA — lead gen TR", type: "TARGET_CPA", targetCpa: 180 }
                          → { resourceName, biddingStrategyId }     # targetCpa in account currency
link_campaign_to_bidding_strategy { customerId, campaignId, biddingStrategyId }
update_bidding_strategy   { customerId, biddingStrategyId, targetCpa: 160 }   # step the target, don't leap
link_campaign_to_bidding_strategy { customerId, campaignId, unlink: true, standardStrategy: "MAXIMIZE_CLICKS" }
remove_bidding_strategy   { customerId, biddingStrategyId }   # only after ALL campaigns are unlinked; permanent
```

For `TARGET_ROAS`: `type: "TARGET_ROAS", targetRoas: 4.0` (ratio = 400%). `MAXIMIZE_CONVERSIONS` accepts an optional `targetCpa` cap; `MAXIMIZE_CONVERSION_VALUE` an optional `targetRoas`. Creates/links/updates are reversible via `undo_change`; `remove_bidding_strategy` is **permanent** — confirm explicitly.

## Pitfalls

- **tCPA set too low → volume death.** Google's own warning: a too-low target makes the system forgo clicks that could have converted, yielding *fewer total conversions* (per Google Ads Help, "About Target CPA bidding", accessed 2026-06). Symptom: impressions and spend collapse right after the target is set. Fix: raise the target toward the recent actual CPA and walk it down gradually — don't conclude "smart bidding doesn't work here".
- **tROAS set too high → constriction.** Same mechanism on the value side: an aspirational ROAS target prices the campaign out of most auctions; spend dries up and the few remaining conversions look great, hiding the lost volume. Anchor the opening target at the campaign's recent actual ROAS (query 2: `conversions_value ÷ cost`), at or above break-even ROAS, and tighten in steps.
- **Reverting during the turbulence window.** The worst response to day-3 volatility is switching strategies back — that restarts learning *again*. Hold for a conversion cycle or two (sourced above), then judge against the pre-change baseline with `references/change-impact.md`.
- **Strategy-hopping instead of target-tuning.** Because target changes don't reset learning but strategy swaps do (sourced above), tuning the target on the current strategy is almost always cheaper than swapping. Swap only when the *goal type* is wrong (counts vs value), not when the number disappoints.
- **One portfolio over mixed economics.** Linking a 70%-margin service campaign and a 20%-margin product campaign to one tROAS averages their truth: one side overbids, the other starves. Portfolio = shared goal *and* shared economics, or don't share.
- **Unlink mask gotcha (live v21 finding).** The campaign strategy oneof's message fields have subfields, so an update that masks a bare `manual_cpc` (or `target_spend`) is rejected with `FIELD_HAS_SUBFIELDS`. The working pattern — validated live and baked into `link_campaign_to_bidding_strategy` — masks a **leaf** instead: set `manualCpc: { enhancedCpcEnabled: false }` with mask `manual_cpc.enhanced_cpc_enabled` (or `target_spend.cpc_bid_ceiling_micros` for MAXIMIZE_CLICKS). Setting the leaf both installs the standard strategy and clears the portfolio link in one mutate. Relevant if you ever drive this via raw `run_gaql`-adjacent mutates or debug an unlink failure.
- **Removing a still-linked portfolio.** `remove_bidding_strategy` requires every campaign detached first — unlink each via `link_campaign_to_bidding_strategy unlink:true`, then remove. And removal is permanent (no `undo_change`); prefer leaving an unused strategy in place if the user is unsure.
- **Forgetting the tracking prerequisite on PMax/Display.** `create_performance_max_campaign` and value-based Display setups inherit the same gate: conversion-based bidding on an untracked account fails server-side. Check the gate *before* composing the campaign, not after the error.
- **Treating "learning" or "limited" as "failing".** Before declaring a strategy bad, pull `campaign.bidding_strategy_system_status` (query 2). A `LEARNING_*` status means the verdict isn't in yet; a `LIMITED_*` status means something specific (data, budget, a bid ceiling) is the real problem. Changing the strategy in either state treats a symptom and restarts the clock.
- **Judging on thin windows.** A week of data on a campaign with 9 conversions is noise. Apply the sample-size discipline from `references/ppc-math.md`: under ~30 conversions in the window (Google's own evaluation guidance, sourced above), widen the window or wait.

## Sources

- Google Ads Help — "About Target ROAS bidding": https://support.google.com/google-ads/answer/6268637 (accessed 2026-06). Eligibility floors per campaign type (Search/Shopping ≥15 conversions/30 days), percent formula, 4-weeks-or-3-conversion-cycles value guidance, target-at-or-below-history guidance.
- Google Ads Help — "About Target CPA bidding": https://support.google.com/google-ads/answer/6268632 (accessed 2026-06). Recommended target = 30-day average CPA adjusted for conversion delay; ≥30-conversion evaluation windows; too-low-target warning.
- Google Ads Help — "How our bidding algorithms learn": https://support.google.com/google-ads/answer/10970825 (accessed 2026-06). Conversion-cycle definition, volatility expectations after changes, target changes not resetting learning.
- Google Ads API docs — "Bidding Strategy Status": https://developers.google.com/google-ads/api/docs/campaigns/bidding/strategy-status (accessed 2026-06). `campaign.bidding_strategy_system_status` field, `ENABLED` semantics, learning/misconfigured surfacing, example query.
- Search Engine Land — "Google Ads to deprecate Enhanced CPC for Search and Display ads": https://searchengineland.com/google-ads-deprecate-enhanced-cpc-search-display-446350 (2024; retirement completed March 2025; accessed 2026-06).
- Live v21 probe findings (this project, validated on a test account): `FIELD_HAS_SUBFIELDS` on bare oneof masks and the leaf-mask unlink pattern; conversion-tracking prerequisite for conversion-based portfolio strategies; `containsEuPoliticalAdvertising` required on v21 campaign creates.
- Cross-references: `references/ppc-math.md` (break-even CPA/ROAS, headroom, sample size), `references/conversion-tracking-first.md` (the gate), `references/conversion-design.md` (signal quality), `references/change-impact.md` (before/after measurement).
