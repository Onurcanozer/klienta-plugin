# Reference — Audience Strategy (Advisory / Honest Scope)

Audiences are *who* sees an ad, layered on top of *what* a keyword or placement targets. This is one of the areas where ad tools overpromise hardest — so the first job of this reference is to be straight about a hard boundary: **our toolset has no write surface for audiences.** There is no `create_user_list`, no Customer Match upload, no tool to attach an audience or set targeting/observation mode or apply an audience bid adjustment (verified against this repo's tool surface). What we *can* do is read audience performance with `run_gaql` and advise the strategy the user then applies in the Google Ads UI. This reference draws that line clearly and shows the nearest supported path that *is* in our hands.

This is the same discipline as `references/benchmark-calibration.md`: claim only what we can defend. Honesty about scope is a competitive advantage, not a weakness — see also the targeting honesty in `references/display-rda.md`.

## When to use

- The user asks who to target, whether to add an audience, or how a specific audience is performing.
- The user wants **remarketing / RLSA** (re-engaging past visitors) — a frequent request that lands almost entirely outside our write surface; say so up front and pivot to what we can contribute.
- You're auditing a Display, PMax, or Search campaign and want to read whether attached audiences are pulling their weight (read-only, fully supported).
- **Not** for promising to build lists or flip audience settings — we cannot, and implying otherwise is the exact overreach this skill avoids.

## Decision framework

### The audience types (so advice names them correctly)

Google groups audience segments into these families (per Google Ads Help, "About audience segments," accessed 2026-06):

| Family | Reaches | Signal source |
|---|---|---|
| **Affinity** | People by long-term interests / habits | Google's interest model |
| **In-market** | People with *recent purchase intent* in a category | Recent behavior |
| **Custom segments** | Audiences you define by keywords, URLs, apps | Advertiser input |
| **Your data** (formerly "remarketing") | People who already engaged with your business (site/app visitors, etc.) | Your tags / data |
| **Customer Match** | Your existing customers, matched from CRM data you upload | Your first-party data |
| **Detailed demographics / life events** | Long-term life facts / milestones | Google's model |

**None of these can be created, uploaded, or attached with our tools.** "Your data" and Customer Match additionally depend on first-party data collection and CRM uploads that are off-platform regardless of tooling.

### Targeting vs observation — advise correctly even though we can't set it

This is the single most important audience *decision*, and it's pure advisory for us (we can read the result, not set the mode):

- **Observation** does **not** narrow reach — it layers an audience on so you can *monitor* its performance and optionally apply a bid adjustment, without restricting who sees the ad (per Google Ads Help, "About 'Targeting' and 'Observation' settings," accessed 2026-06).
- **Targeting** *restricts* the ad group/campaign to only the selected audience — it narrows reach (same source).

**Decision rule to advise:** on Search, default to **observation** first — it gathers audience performance data with zero downside to reach; only move to **targeting** once the data shows an audience worth restricting to. Recommending "targeting" blind throttles volume before you know the audience converts. We can frame this and, once the user has set observation in the UI, *read the result* to tell them whether to escalate to targeting or apply a bid adjustment — but the user makes both edits in the UI.

### The honest scope line — and the nearest supported path

| Capability | In our tools? | Where it happens |
|---|---|---|
| Read audience performance (which audience, how it did) | ✅ `run_gaql` | Here |
| Strategy/decision advice (which family, observation vs targeting) | ✅ advisory | Here (we advise) |
| Build a "your data"/remarketing list | ❌ no tool | Google Ads UI + site tags |
| Customer Match upload (CRM data) | ❌ no tool | Google Ads UI / API outside this surface |
| Attach an audience to a campaign/ad group | ❌ no tool | Google Ads UI |
| Set observation vs targeting mode | ❌ no tool | Google Ads UI |
| Audience bid adjustment | ❌ no tool | Google Ads UI |

**Nearest supported path — what we *can* do that approximates audience strategy:**

- **Audience-informed *structural* decisions in Search.** We can't attach an audience, but we *can* build structure that reflects audience value: separate campaigns/budgets for high-value segments (e.g. a dedicated campaign the user then layers a remarketing audience onto in the UI), using `create_search_campaign` / `update_campaign_budget` (`references/campaign-structure.md`). The structure is ours; the audience layer is the user's UI step.
- **Negatives and structure to shape *who* effectively gets reached.** Brand vs non-brand separation and negative keywords (`references/shared-negatives.md`, `references/search-term-mining.md`) steer spend toward the right intent — an intent proxy for audience when the audience tools aren't ours.
- **Measurement support around a remarketing campaign once it exists.** If the user builds the list and campaign in the UI, we supply the GAQL read-back and analysis (below) and the creative/structure around it.

## GAQL / tool examples

**1. How are attached audiences performing?** (read-only — fully supported). `campaign_audience_view` reports audiences attached at campaign level; `ad_group_audience_view` at ad-group level (per Google Ads API reference, `campaign_audience_view` / `ad_group_audience_view`, v21, accessed 2026-06):

```
SELECT campaign.name, user_list.name, user_list.type,
       metrics.impressions, metrics.clicks, metrics.cost_micros, metrics.conversions
FROM campaign_audience_view
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.cost_micros DESC
```

**Decision it supports:** identify which "your data"/remarketing lists (the `USER_LIST` criterion type) earn their spend. An audience spending with no conversions is a candidate to drop or down-bid — but the bid/exclusion change is the user's UI step (scope table above). We name the offender precisely; we don't pull the lever.

**2. Is a remarketing list even attached, and in which mode?** (read-only):

```
SELECT campaign.name, ad_group.name, ad_group_criterion.type,
       ad_group_criterion.user_list.user_list, metrics.conversions
FROM ad_group_audience_view
WHERE segments.date DURING LAST_30_DAYS
```

**Decision it supports:** confirm whether audiences are present before advising. If a user *thinks* they're remarketing but no audience criteria appear, that's the finding — and the fix (attach it, choose observation) is a UI walkthrough we provide, not a write we execute.

**3. Audience-informed structure (the nearest supported write).** Placeholders, never real IDs; confirm with the user:

```
create_search_campaign  { customerId: "<customer-id>", name: "Search | Remarketing-layer | ...", dailyBudget }   → PAUSED
# user then attaches their "your data" audience to this campaign in the Google Ads UI (observation first)
```

We build and budget the campaign; the audience attachment is the user's step. We then read it back with query 1.

## Pitfalls

- **Promising audience builds or uploads.** We have no `create_user_list`, no Customer Match upload, no attach/mode/bid-adjustment tool. Never imply we can — advise the UI path and own only the read + structure.
- **Recommending "targeting" before data.** Targeting narrows reach (sourced above); default-advise observation first, escalate only on evidence.
- **Confusing "your data" with Customer Match.** "Your data" = engagement-based (site/app); Customer Match = CRM upload. Both off-platform for us, but they're different mechanisms — name them correctly.
- **Reading audience views as if we can act on them.** The GAQL reads above produce *recommendations*; the apply step (exclude, bid-adjust, switch mode) is the user's UI action. Label it as such, like the Display targeting scope note (`references/display-rda.md`).
- **Treating intent proxies as audiences.** Negatives/structure approximate *who* gets reached but are not the same as a real audience layer — be honest that it's the nearest path, not equivalence.
- **First-party data assumptions.** Customer Match and "your data" depend on consent-compliant data collection the user owns; never assume it exists or advise uploading data without flagging the off-platform/consent prerequisites.

## Sources

- Google Ads Help, "About audience segments" (affinity, in-market, custom, your data / formerly remarketing, Customer Match, detailed demographics, life events) — accessed 2026-06: https://support.google.com/google-ads/answer/2497941
- Google Ads Help, "About 'Targeting' and 'Observation' settings" (observation does not narrow reach; targeting restricts reach; observation allows monitoring + optional bid adjustments) — accessed 2026-06: https://support.google.com/google-ads/answer/7365594
- Google Ads API reference, `campaign_audience_view` and `ad_group_audience_view` (audience performance read; segmented by `user_list`; criterion types `USER_LIST` / `USER_INTEREST` / `CUSTOM_AUDIENCE`), v21 — accessed 2026-06: https://developers.google.com/google-ads/api/fields/v21/ad_group_audience_view
- Tool surface (this repo): verified absence of any audience/user-list/Customer Match write tool (`create_user_list` not present) — the basis for the honest-scope boundary throughout.
- Cross-references: `references/campaign-structure.md` (structure/budget levers), `references/shared-negatives.md` + `references/search-term-mining.md` (intent shaping), `references/display-rda.md` (matching read-vs-write honesty), `references/benchmark-calibration.md` (claim-only-what-we-can-defend discipline).
