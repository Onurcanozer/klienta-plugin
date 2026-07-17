# Reference — Meta Ads Audit Playbook (Read-Only, Faz 1)

How to audit a Meta (Facebook/Instagram) ad account with Klienta's 7 read-only `meta_*` tools. Five write tools exist (`meta_update_status`, `meta_update_budget`, `meta_update_bid`, `meta_rename`, `meta_update_schedule` — see the SKILL.md Meta Ads WRITE section, incl. the minor-units and no-dry-run rules), and the Faz 3 create/targeting tools add targeting fixes (`meta_update_targeting` — geo/age/gender/placements plus interests/behaviors with ids from `meta_search_targeting`, search first) and new PAUSED structure builds (`meta_create_campaign`/`meta_create_adset`/`meta_create_ad` — see the SKILL.md Meta Ads CREATE section). Findings in those areas can end in a confirmed Klienta write; what remains (edits to existing creatives, deletes, custom audiences, flexible_spec AND/OR combos, appeals) still ends in "monitor" or "do this in Ads Manager." Never imply Klienta can apply a Meta fix outside these levers.

All calls below were validated against the shipped Faz-1 tool surface: `meta_list_ad_accounts`, `meta_list_campaigns`, `meta_list_adsets`, `meta_list_ads`, `meta_get_insights`, `meta_get_ad_creatives`, `meta_diagnose_delivery`. `act_<test-account-id>` is a placeholder — always use the id returned by `meta_list_ad_accounts`.

---

## 1. The descent order — account → campaign → ad set → ad → creative

Audit top-down. Each level answers one question; don't skip levels, because a "broken ad" is often a paused campaign two levels up.

| Step | Tool | Question it answers | Fields to read |
|---|---|---|---|
| 1 | `meta_list_ad_accounts` | Which accounts exist, are they healthy? | `accountStatus` (1=ACTIVE, 2=DISABLED, 3=UNSETTLED, 7=PENDING_RISK_REVIEW, 9=IN_GRACE_PERIOD, 101=CLOSED), `currency`, `timezone` |
| 2 | `meta_list_campaigns` | What's the structure and where does budget live? | `objective`, `status` vs `effective_status`, `daily_budget`/`lifetime_budget` (CBO signal), `buying_type`, `bid_strategy` |
| 3 | `meta_list_adsets` | Can each ad set actually deliver? | `effective_status`, `learning_stage_info`, `issues_info`, `optimization_goal`, `billing_event`, budgets, `targeting` |
| 4 | `meta_list_ads` | Are ads approved and running? | `effective_status`, `issues_info` (disapprovals surface here), `creative{id}` |
| 5 | `meta_get_ad_creatives` | What is the copy/creative actually saying? | `title`, `body`, `link_url`, `call_to_action_type`, `object_story_spec` / `asset_feed_spec` |
| 6 | `meta_get_insights` | Is any of it performing? | metrics per level — see §3 discipline first |

Account-status gate: if `accountStatus` ≠ 1, stop the performance audit — a DISABLED (2) or UNSETTLED (3, unpaid balance) account delivers nothing regardless of campaign settings. Report that first.

`status` vs `effective_status`: `status` is what the advertiser set; `effective_status` is what Meta actually does after inheriting parent pauses and review outcomes. An ad with `status=ACTIVE` inside a paused campaign shows `effective_status=CAMPAIGN_PAUSED`. **Audit on `effective_status`, always.**

---

## 2. Delivery diagnosis — `meta_diagnose_delivery` first

For any "why isn't this spending / delivering?" question, call `meta_diagnose_delivery` before manual digging — one call batches campaign status + budget, ad-set learning phase + issues, ad-level disapprovals, and last-7d spend into a prioritized findings list:

```json
{ "accountId": "act_<test-account-id>", "campaignId": "<campaign-id>" }
```

Read the result in this order:

1. **`findings` with severity `high`** — hard blockers: `CAMPAIGN_PAUSED`, `CAMPAIGN_ENDED` / `CAMPAIGN_PENDING` (schedule), `NO_ADSETS` / `ADSETS_NOT_ACTIVE`, `NO_ADS` / `ADS_NOT_ACTIVE`, `ADS_DISAPPROVED_OR_ISSUES`, plus any `issues_info` Meta reports directly. Any one of these fully explains zero delivery — fix these before discussing performance.
2. **Learning-phase states** (from `learning_stage_info.status` on each ad set):
   - `LEARNING` — normal after launch or a significant edit. Delivery and CPA are unstable; do **not** judge performance yet and do not recommend edits (each significant edit resets learning). Exits after ~50 optimization events in a 7-day window.
   - `LEARNING_LIMITED` (or `FAIL`) — the ad set cannot gather enough optimization events at its current setup. This is a *structural* problem: audience too narrow, budget too small for the event's cost, or the optimization event too rare. Remedies (Ads Manager): broaden targeting, raise budget, consolidate overlapping ad sets, or optimize for a higher-volume event.
   - No learning info + stable delivery — learning exited; performance numbers are judgeable.
3. **`status.spend7d` / `impressions7d`** — the ground truth. `effective_status=ACTIVE` with 0 impressions over 7 days and no high finding usually means auction starvation (tiny budget, ultra-narrow audience, uncompetitive bid cap) — check ad-set budget and `bid_strategy` next.
4. **`queryErrors`** — if present, part of the diagnosis is missing (rate limit, permission). Say so; don't present a partial diagnosis as complete.

Distinguish the three non-delivering classes explicitly in your report: **paused** (someone turned it off — intentional?), **limited** (running but throttled: learning-limited, low budget) and **rejected** (policy — `effective_status` `DISAPPROVED`/`WITH_ISSUES` on ads; fix is editing the ad in Ads Manager and resubmitting for review).

---

## 3. Insights discipline — the attribution window is part of the number

`meta_get_insights` requests explicit attribution windows (`1d_click`, `7d_click`, `1d_view`) and echoes them as `attributionWindows` in the output. Since **January 2026 Meta removed the 7d-view and 28d-view windows (28d-click was already retired in 2021)** — historical reports quoted with those windows are not reproducible today.

Rules — no exceptions:

- **Never report ROAS, CPA, purchases or any action-derived number without naming its window.** "ROAS 3.2 (7d_click)" is a fact; "ROAS 3.2" is not. `actions`, `action_values` and `purchase_roas` come back as arrays with per-window values — read the window key, don't sum blindly.
- **Keep click-through and view-through separate.** `7d_click` is someone who clicked and converted within 7 days; `1d_view` is someone who merely saw the ad and converted within a day. Adding them into one "conversions" figure inflates results — report them side by side.
- **Meta and Google Ads conversion numbers are not comparable** — different attribution models, different windows, different dedup. Never put them in the same column of a table without a loud caveat.
- Typical audit call:

```json
{
  "accountId": "act_<test-account-id>",
  "level": "campaign",
  "datePreset": "last_30d",
  "fields": ["campaign_name", "impressions", "clicks", "spend", "ctr", "cpc",
             "reach", "frequency", "actions", "action_values", "purchase_roas"]
}
```

- `spend` is in account-currency units (not cents) in insights output; entity budgets (§4) ARE minor units. Don't mix them without converting.
- Narrow to specific campaigns with `filtering`: `[{"field":"campaign.id","operator":"IN","value":["<campaign-id>"]}]`.

---

## 4. Reading the budget structure — CBO vs ABO, and the cents trap

Budget lives at exactly one level per campaign:

- **CBO** (Advantage campaign budget): `meta_list_campaigns` shows `daily_budget` or `lifetime_budget` on the campaign. Meta distributes across ad sets automatically; per-ad-set budget fields are then absent/ignored. Audit question: is one ad set eating the whole budget while others starve? Check with `meta_get_insights` at `level: "adset"` filtered to that campaign.
- **ABO**: campaign budget fields are null; each ad set in `meta_list_adsets` carries its own `daily_budget`/`lifetime_budget`. Audit question: are budgets spread too thin — many small ad sets each too underfunded to exit learning (§2)?

`meta_diagnose_delivery` flags ABO explicitly with the `ABO_BUDGETS` info finding.

**Minor-units warning:** every `daily_budget` / `lifetime_budget` / `bid_amount` on campaign/ad-set objects is an **integer in minor units of the account currency** (cents for USD/EUR/TRY). `5000` = $50.00, not $5,000. Divide by 100 before quoting (a few currencies have no minor unit — check `currency` from `meta_list_ad_accounts` if unusual). Misreading this by 100× is the single most embarrassing audit error available; the tools repeat this in their `budgetNote` for a reason.

---

## 5. Creative fatigue — frequency up, CTR down

Fatigue is a *trend*, not a snapshot. Detect it with daily rows:

```json
{
  "accountId": "act_<test-account-id>",
  "level": "ad",
  "timeRange": { "since": "<30d-ago>", "until": "<today>" },
  "timeIncrement": 1,
  "fields": ["ad_name", "impressions", "ctr", "cpc", "frequency", "spend"],
  "filtering": [{ "field": "adset.id", "operator": "IN", "value": ["<adset-id>"] }]
}
```

Read the two curves together across the daily rows:

- **`frequency` climbing** (average times each person has seen the ad, scoped to the queried range) **while `ctr` declines week over week** — the same audience is seeing the same creative too often and has stopped responding. Rising `cpc` at flat bids corroborates.
- Either signal alone is weak: high frequency with stable CTR = the audience still responds; falling CTR at low frequency = a creative or audience problem, not fatigue.
- Split by placement when the account runs auto placements: add `"breakdowns": ["publisher_platform"]` — fatigue often hits one platform (e.g. Instagram) while Facebook still performs.
- Compare against the ad's **own** first-two-weeks CTR, not any external number (§7).

Fatigue remediation is Ads Manager work for now: refresh creative, broaden the audience, or lower spend on the fatigued ad set. Use `meta_get_ad_creatives` to review what's currently running (`title`, `body`, `call_to_action_type`) and to check how many distinct creatives an ad set actually rotates.

---

## 6. Common findings checklist

Run through these on every audit; each maps to specific tool evidence.

- [ ] **Ad set stuck in learning / learning-limited** — `meta_list_adsets` → `learning_stage_info.status` = `LEARNING` for >1–2 weeks, or `LEARNING_LIMITED`. Structural underfeeding (§2.2).
- [ ] **ACTIVE but zero delivery** — `effective_status=ACTIVE` yet `meta_get_insights` shows 0 impressions for 7d. Auction starvation or unresolved issue; run `meta_diagnose_delivery` on the parent campaign.
- [ ] **Single-creative ad set** — `meta_list_ads` shows one ad under an ad set. No creative rotation → faster fatigue, no learning signal about what works. Flag as concentration risk.
- [ ] **Platform imbalance** — insights with `"breakdowns": ["publisher_platform"]`: one platform absorbing most spend at clearly worse CPA (window-labeled) than the others. Consider manual placements (Ads Manager).
- [ ] **Disapproved / with-issues ads silently dead** — `meta_list_ads` → `effective_status` `DISAPPROVED` or `WITH_ISSUES`, `issues_info` has the policy detail. Often unnoticed for weeks.
- [ ] **Campaign schedule surprises** — `stop_time` in the past or `start_time` in the future on `meta_list_campaigns`. Diagnose tool flags these as `CAMPAIGN_ENDED` / `CAMPAIGN_PENDING`.
- [ ] **Budget misallocation** — CBO campaign where adset-level insights show one ad set with all spend and poor results; or ABO with many sub-scale budgets (§4).
- [ ] **Frequency creep** — account- or campaign-level `frequency` trending up month over month without creative refresh (§5).
- [ ] **Account-level red flags** — `accountStatus` 3 (unsettled — payment issue) or 9 (grace period) on `meta_list_ad_accounts`: everything else is secondary until billing is fixed.

---

## 7. Honesty rules

These are hard constraints on every Meta audit output:

1. **The audit itself stays read-only.** The audit pass never mutates anything. When a finding lands on one of the five write levers (status, budget, bid, name, schedule), the fix MAY be offered as a Klienta write — with explicit confirmation, the minor-units quote, and a read-back, per the SKILL.md Meta Ads WRITE section (note its LIVE-REHEARSAL-PENDING caution). Every other fix routes to Ads Manager ("Edit the ad and resubmit for review"); never say Klienta changed something it didn't, and never imply a write lever exists beyond those five.
2. **No invented benchmarks.** Never quote industry CPM/CTR/CPA/ROAS figures ("good CTR is ~1%") — you don't have that data and the tools don't provide it. The only valid comparison baseline is **the account's own history**: this ad vs its own first weeks, this month vs last month, this ad set vs the sibling ad set — all via `meta_get_insights` with explicit ranges.
3. **Say "insufficient data" when it's insufficient.** An ad set with 300 impressions has no judgeable CTR; a campaign 3 days into learning has no judgeable CPA. State the threshold problem instead of forcing a verdict.
4. **Degraded ≠ empty.** `connectionErrors` in `meta_list_ad_accounts` or `queryErrors` in `meta_diagnose_delivery` mean part of the picture is missing — report the gap explicitly. An error is never "the account has no campaigns."
5. **Windows always attached** (§3). And never merge Meta and Google numbers into one metric.

---

## 8. Worked call sequences

**A. Full account audit (top-down):**

```
meta_list_ad_accounts {}
meta_list_campaigns  { "accountId": "act_<test-account-id>" }
meta_list_adsets     { "accountId": "act_<test-account-id>" }
meta_get_insights    { "accountId": "act_<test-account-id>", "level": "campaign", "datePreset": "last_30d" }
meta_get_insights    { "accountId": "act_<test-account-id>", "level": "campaign", "datePreset": "last_30d",
                       "breakdowns": ["publisher_platform"] }
```

Then descend (`meta_list_ads`, `meta_get_ad_creatives`) only into campaigns the insights flagged.

**B. "Campaign X isn't spending":**

```
meta_diagnose_delivery { "accountId": "act_<test-account-id>", "campaignId": "<campaign-id>" }
```

Only if findings are empty and spend7d is still 0: `meta_list_adsets { "accountId": "act_<test-account-id>", "campaignId": "<campaign-id>" }` and inspect budgets/bid_strategy manually.

**C. Fatigue check on a hero ad set:**

```
meta_list_ads     { "accountId": "act_<test-account-id>", "adsetId": "<adset-id>" }
meta_get_insights { "accountId": "act_<test-account-id>", "level": "ad", "timeIncrement": 1,
                    "timeRange": { "since": "2026-06-16", "until": "2026-07-16" },
                    "fields": ["ad_name", "impressions", "ctr", "frequency", "spend"],
                    "filtering": [{ "field": "adset.id", "operator": "IN", "value": ["<adset-id>"] }] }
meta_get_ad_creatives { "accountId": "act_<test-account-id>", "adId": "<ad-id>" }
```

**D. Copy review across the account:**

```
meta_get_ad_creatives { "accountId": "act_<test-account-id>", "limit": 100 }
```

Cross-reference `creative.id` from `meta_list_ads` to know which creatives are actually live before critiquing library leftovers.
