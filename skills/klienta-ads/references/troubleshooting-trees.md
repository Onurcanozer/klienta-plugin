# Reference — Troubleshooting Trees (GAQL Diagnostic Paths)

When a user reports a symptom — "my ads aren't showing," "costs jumped," "conversions died" — the worst move is to guess and write. Each symptom has a small number of distinct causes, and the right one is decided by data, not intuition. This reference is six step-by-step diagnostic trees: each starts from a complaint, walks `run_gaql` checks in cause-likelihood order, and ends with the specific tool to intervene — plus an honest mark on the branches that live off-platform (landing pages, billing, account suspension), which our tools cannot touch.

This is the sibling of `references/performance-triage.md` (waste vs budget vs rank); use that for the *which-of-three-ways-is-it-bad* classification, and these trees for *named symptoms*. All queries are written for Google Ads API **v21** via `run_gaql`; run them read-only first and let the numbers pick the branch.

**Standing write discipline (applies to every "intervene" step below):** reads are free; every write needs the user's confirmation. Most writes are reversible with `undo_change` (create/add → removed/unlinked; update → snapshot restore), but `remove_*` and `unlink_*` are **permanent** — prefer pausing over removing, and confirm removals explicitly. Budget/CPC guardrails are enforced server-side; a blocked write returns the limit it hit.

## When to use

- A user reports a concrete symptom and you need to find the cause before recommending anything.
- You want to hand the model (or yourself) a repeatable order-of-checks so cheap/common causes are ruled out before expensive/rare ones.
- **Not** a substitute for the economics check: once you find the mechanical cause, "is the fixed state actually profitable?" still routes through `references/benchmark-calibration.md` and `references/ppc-math.md`.

## Decision framework — the six trees

### Tree 1 — "Ads not showing / 0 impressions"

Walk in this order; the top causes are the cheapest to check and the most common.

**1a. Is the campaign even running?**
```
SELECT campaign.name, campaign.status, campaign.advertising_channel_type
FROM campaign
WHERE campaign.name LIKE '%<name>%'
```
PAUSED/REMOVED campaign → nothing serves. *Intervene:* `update_campaign { status: "ENABLED" }` (reversible). Also check the ad group and ad aren't paused (`ad_group_ad.status`).

**1b. Are the ads approved?** Disapproved ads receive no impressions (per Google Ads Help, "Find your ad status," accessed 2026-06).
```
SELECT ad_group_ad.ad.id, ad_group_ad.status,
       ad_group_ad.policy_summary.approval_status,
       ad_group_ad.policy_summary.review_status,
       ad_group_ad.policy_summary.policy_topic_entries
FROM ad_group_ad
WHERE segments.date DURING LAST_30_DAYS
```
`DISAPPROVED` → fix the violating content and resubmit a new RSA (`create_responsive_search_ad`); the appeal/exemption flow itself is **off-platform** (no tool). `APPROVED_LIMITED` / "Eligible (limited)" → it *can* serve but is restricted by policy (location/age/device/restricted-category — per Google Ads Help, "Eligible (limited)," accessed 2026-06), so low/zero impressions can be legitimate. `UNDER_REVIEW` → most ads are reviewed within one business day (per Google Ads Help, accessed 2026-06); wait before "fixing."

**1c. Is budget or rank starving it?** (the impression-share split — see `references/performance-triage.md`)
```
SELECT campaign.name, metrics.search_impression_share,
       metrics.search_budget_lost_impression_share,
       metrics.search_rank_lost_impression_share
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
```
Search lost IS (budget) = share of time ads didn't show due to insufficient budget; lost IS (rank) = due to poor Ad Rank (per Google Ads Help, "Get impression share data," accessed 2026-06; campaign-level only). High budget-lost → `update_campaign_budget` (small step, only if converting). High rank-lost → bid/Quality Score, not budget (Tree 2 / `references/quality-score.md`).

**1d. Is there even demand, and can the auction find you?** A brand-new or very tight exact-keyword set with low volume can legitimately show near-zero impressions; check `metrics.impressions` at keyword level (`keyword_view`) and confirm geo/language targeting is what the user intended (set at creation, not editable later — `references/campaign-structure.md`). **Off-platform branch:** billing failure or account suspension also produces zero serving and is invisible to our read tools — if everything above looks healthy, tell the user to check billing/account status in the Google Ads UI.

### Tree 2 — "Impressions but no clicks" (low CTR)

Impressions prove eligibility; no clicks is a relevance/appeal problem, almost never a bid problem.
```
SELECT ad_group_criterion.keyword.text, metrics.impressions,
       metrics.clicks, metrics.ctr
FROM keyword_view
WHERE segments.date DURING LAST_30_DAYS
  AND metrics.impressions > 100
ORDER BY metrics.ctr ASC
```
Low-CTR-with-volume keywords → the keyword, ad copy, and intent are misaligned. Check ad relevance and creative quality:
```
SELECT ad_group_ad.ad.id, ad_group_ad.ad_strength
FROM ad_group_ad
WHERE segments.date DURING LAST_30_DAYS
```
```
SELECT ad_group_criterion.keyword.text,
       ad_group_criterion.quality_info.quality_score,
       ad_group_criterion.quality_info.creative_quality_score,
       ad_group_criterion.quality_info.search_predicted_ctr
FROM keyword_view
WHERE segments.date DURING LAST_30_DAYS AND metrics.impressions > 0
```
*Intervene:* rewrite the RSA for tighter keyword→ad match (`create_responsive_search_ad`, `references/rsa-writing.md`); tighten match type (`update_keyword`); split a grab-bag ad group (`references/campaign-structure.md`). **Raising the bid does not fix low CTR** — it buys more of the same ignored impressions. Note: `ad_strength` rates *asset diversity*, not Quality Score (`references/quality-score.md`); treat it as a creative checklist, not a performance verdict.

### Tree 3 — "CPA spiked" (cost per conversion jumped)

Confirm the spike is real and find which surface caused it before touching bids.
```
SELECT campaign.name, metrics.cost_micros, metrics.conversions,
       metrics.cost_per_conversion, metrics.average_cpc
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.cost_per_conversion DESC
```
Compare to the prior period (dated `run_gaql`, `references/change-impact.md`) — a one-window number isn't a spike. Then localize the cause:

- **New wasteful search terms** (broad/phrase pulling junk):
  ```
  SELECT search_term_view.search_term, campaign.name,
         metrics.cost_micros, metrics.conversions
  FROM search_term_view
  WHERE segments.date DURING LAST_30_DAYS
  ORDER BY metrics.cost_micros DESC
  ```
  *Intervene:* `add_negative_keywords` or a shared list (`references/shared-negatives.md`); tighten match type with `update_keyword`.
- **Bid strategy out of band** — average CPC climbing under a manual or target strategy; check the system status:
  ```
  SELECT campaign.name, campaign.bidding_strategy_type,
         campaign.bidding_strategy_system_status, campaign.target_cpa
  FROM campaign
  WHERE segments.date DURING LAST_30_DAYS
  ```
  *Intervene:* adjust the target/caps via `update_bidding_strategy` (or `update_campaign`), in small steps (`references/bidding-spectrum.md`).
- **Competition / auction shift** — costs up with no internal change. This is partly **off-platform** (auction dynamics we can't read directly); the supported read is `metrics.search_rank_lost_impression_share` rising, which points to bid/quality pressure rather than your own mistake.

### Tree 4 — "Conversions dropped"

Order matters here: rule out *measurement* before *market*, because a tracking break looks exactly like a demand collapse.

**4a. Is tracking healthy?** (the gate — `references/conversion-tracking-first.md`)
```
SELECT conversion_action.name, conversion_action.status,
       conversion_action.type, conversion_action.category
FROM conversion_action
```
A conversion action that flipped to non-active, or a tag change, can zero out reported conversions while real ones still happen. **This is the most common false alarm.** Tracking/tag and landing-page changes are largely **off-platform** — we can read `conversion_action.status` but not inspect the website tag; say so plainly and have the user verify the tag/GTM in the UI.

**4b. Is it a trend or a blip?** Pull the conversion trend over weeks and against the prior period:
```
SELECT campaign.name, metrics.conversions, metrics.conversions_value,
       metrics.cost_per_conversion
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
```
Re-run for the prior window; a single low day is noise (`references/benchmark-calibration.md`).

**4c. Seasonality / market.** A real, broad decline that tracks the calendar is likely demand, not a defect — there is **no seasonality-adjustment tool** in our surface; this branch is advisory (`references/budget-pacing-seasonality.md` if present).
**4d. Bid/learning disruption.** A recent strategy change can suppress conversions while the strategy recalibrates — it can take up to around 50 conversion events or 3 conversion cycles to settle (per Google Ads Help, "Duration of the learning period," accessed 2026-06). Check `campaign.bidding_strategy_system_status` (Tree 6) before "fixing" a strategy that's merely learning.
**4e. Landing page.** A broken or slowed page kills conversions with clicks intact — **off-platform**; we can flag the suspicion (clicks steady, conversions gone) but cannot inspect or fix the page.

### Tree 5 — "Cost surged" (total spend jumped)

Distinct from Tree 3: here total *spend* rose, regardless of CPA. Find what opened the spigot.
```
SELECT campaign.name, metrics.cost_micros, metrics.clicks, metrics.impressions
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.cost_micros DESC
```
Then the usual suspects, in order:

- **Broad match / match-type drift** — a broad keyword catching a flood of new queries. Confirm via `search_term_view` (Tree 3 query): a surge of new, loosely-related terms is the tell. *Intervene:* negatives (`add_negative_keywords` / shared list) and tighter match types (`update_keyword`).
- **Budget raised** — someone increased `update_campaign_budget`; cross-check with `get_changes` (`references/change-impact.md`).
- **Bid strategy** — a switch to a volume-seeking strategy (e.g. Maximize Clicks/Conversions) or a raised tCPA/tROAS target spends more by design; read `campaign.bidding_strategy_type` / `bidding_strategy_system_status`. *Intervene:* `update_bidding_strategy` or `update_campaign_budget` to cap, small steps.
- **Network spread** — spend leaking to search partners / display segment. Segment by network to see it:
  ```
  SELECT campaign.name, segments.ad_network_type,
         metrics.cost_micros, metrics.conversions
  FROM campaign
  WHERE segments.date DURING LAST_30_DAYS
  ```
  If a non-Search network is spending poorly, the network setting is the lever — **off-platform** to change (no network-toggle tool here); advise the UI setting.

### Tree 6 — "Limited status: budget vs bid strategy vs learning"

The "Limited"/"Learning" labels confuse users because they look like errors but often aren't. Read the actual system status before reacting:
```
SELECT campaign.name, campaign.bidding_strategy_type,
       campaign.bidding_strategy_system_status,
       metrics.search_budget_lost_impression_share
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
```
Google's bid-strategy statuses (per Google Ads Help, "About bid strategy statuses," accessed 2026-06):

- **Learning** — the strategy is recalibrating after a change (new strategy, settings edit, campaigns/keywords added/removed). *Action:* usually **wait**, don't stack more changes; recalibration can take up to ~50 conversions or 3 conversion cycles (sourced above). Resetting it repeatedly is the real damage.
- **Limited → budget constrained** — budget is holding bids back. Confirm with `search_budget_lost_impression_share`; if it converts profitably, step `update_campaign_budget` up (only then). This is the same "budget" face as `references/performance-triage.md`.
- **Limited → bid limits / inventory / bidding strategy** — bid caps, thin search volume, or an underperforming target. *Action:* loosen caps or adjust the target via `update_bidding_strategy` (`references/bidding-spectrum.md`); inventory-limited has no bid fix.
- **Misconfigured (conversion setting)** — a conversion-based strategy (tCPA/tROAS/Maximize Conversions) lacking proper conversion tracking. *Action:* fix the conversion action first (`create_conversion_action`, `references/conversion-tracking-first.md`) — bidding can't optimize toward conversions it can't see.

*Decision in one line:* **Learning → wait; Limited-by-budget → budget (if profitable); Limited-by-bid/strategy → bidding; Misconfigured → tracking.** Don't raise budget on a rank/learning problem, and don't churn a strategy that's merely learning.

## Pitfalls

- **Guessing before reading.** Every tree starts with a read; the symptom alone rarely names the cause (low CTR ≠ low bid; "limited" ≠ broken).
- **Acting on a thin window.** A spike/drop over a few clicks or one day is noise — compare to the prior period (`references/change-impact.md`, `references/benchmark-calibration.md`) before writing.
- **Calling it "waste" with tracking off.** Zero conversions is meaningless until `conversion_action.status` and the tag are verified (Tree 4a) — the diagnosis is *invalid*, not "bad."
- **Resetting a learning strategy.** Stacking changes on a `Learning` status restarts the clock; wait it out (sourced above).
- **Raising budget on a rank/learning problem.** Budget only fixes budget-lost IS; on rank or learning it just costs more for the same losses.
- **Implying we can fix off-platform branches.** Billing, account suspension, landing pages, conversion tags, appeals, and network toggles are **not** on our write surface — diagnose and advise precisely, but label the apply step as the user's UI/account task.
- **Forgetting permanence.** `remove_*`/`unlink_*` don't undo; prefer pausing, and confirm any removal explicitly.

## Sources

- Google Ads Help, "Find your ad status" (Approved / Disapproved — no impressions / Eligible (limited) / Under review — most ads within one business day) — accessed 2026-06: https://support.google.com/google-ads/answer/1722129
- Google Ads Help, "Eligible (limited): Definition" (policy restrictions by location/age/device/restricted category) — accessed 2026-06: https://support.google.com/google-ads/answer/2684542
- Google Ads Help, "Get impression share data" (lost IS budget = time not shown due to insufficient budget; lost IS rank = due to Ad Rank; campaign-level) — accessed 2026-06: https://support.google.com/google-ads/answer/7103314
- Google Ads Help, "About bid strategy statuses" (Inactive / Active / Learning / Limited [inventory, bid limits, budget constrained, bidding strategy] / Misconfigured) — accessed 2026-06: https://support.google.com/google-ads/answer/6263057
- Google Ads Help, "Duration of the learning period for campaigns" (up to ~50 conversion events or 3 conversion cycles to calibrate) — accessed 2026-06: https://support.google.com/google-ads/answer/13020501
- Validated GAQL field names (Google Ads API v21) reused from this corpus's live-tested queries: `references/performance-triage.md`, `references/quality-score.md`, `references/conversion-design.md`.
- Cross-references: `references/performance-triage.md` (budget vs rank), `references/bidding-spectrum.md` (strategy/targets), `references/conversion-tracking-first.md` (tracking gate), `references/shared-negatives.md` + `references/search-term-mining.md` (negatives), `references/change-impact.md` (before/after windows), `references/campaign-structure.md` (structure/geo).
