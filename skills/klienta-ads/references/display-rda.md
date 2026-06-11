# Reference — Display Campaigns & Responsive Display Ads

Display is push, not pull: ads appear while people read, watch, and browse — they didn't ask for you. That changes everything downstream: expectations (clicks are cheap, intent is low), creative (images carry the message), and measurement (CTR comparisons with Search are meaningless). This reference is the build flow with our tools, the live-validated contract quirks, and an honest map of which Display levers we have and which we don't.

Write surface: `create_image_asset` → `create_display_campaign` → `create_ad_group` (type `DISPLAY_STANDARD`) → `create_responsive_display_ad`. Diagnostics via `run_gaql`, Google Ads API **v21**. The full flow below was validated end-to-end on a live test account; the asset rules and the logo finding are empirical, not just documentation.

## When to use

- The user wants **awareness or reach** beyond what Search captures — there is no more search demand to buy (impression share is already high on converting campaigns) and the next growth lever is being seen earlier.
- **Cheap top-of-funnel traffic** is acceptable as a goal in itself, with the understanding that conversions will lag Search.
- **Not** as a substitute for Search when the account has unspent search demand — check `metrics.search_impression_share` first (`references/ppc-math.md`); buying low-intent display clicks while high-intent search clicks go unbought is backwards.
- **Set expectations before building:** display CTRs run far below Search — WordStream's all-industry benchmarks have historically put display around 0.46% CTR versus 3.17% for search (per WordStream all-industry Google Ads benchmarks, earlier-period data, accessed 2026-06), and the gap persists in their 2024 data where search averaged 6.42% (per WordStream/LocaliQ 2024 Google Ads benchmarks, accessed 2026-06). Judge Display on cost-efficient reach and assisted outcomes, never on Search-style CTR.

## Decision framework

### Build order (each step its own call — no atomic mutate needed)

Unlike PMax, Display assembles with plain sequential calls — no atomic temp-id composition, no Brand Guidelines gate, no asset-group quirks (validated live):

1. **Images first** — `create_image_asset` for at least one landscape **1.91:1** marketing image and one square **1:1**. Images are content-hash deduped by Google: two byte-identical uploads collide with `DUPLICATE_ASSETS_WITH_DIFFERENT_FIELD_VALUE` (live finding — our tool translates this error). Genuinely different images, or reuse one asset's resourceName twice instead of re-creating it.
2. **Campaign** — `create_display_campaign` (created PAUSED for safety). Bidding default is `MAXIMIZE_CLICKS` (`targetSpend`), which **requires no conversion tracking** — the right opener for an account whose tracking isn't proven yet. `MAXIMIZE_CONVERSIONS`/`MAXIMIZE_CONVERSION_VALUE` are available but conversion-based: gate them behind `references/conversion-tracking-first.md`, same as Search (see `references/bidding-spectrum.md`).
3. **Ad group** — `create_ad_group` with `type: DISPLAY_STANDARD` (a Search-type ad group under a Display campaign won't take an RDA).
4. **The ad** — `create_responsive_display_ad`, then enable the campaign with `update_campaign` once everything is in place.

### The RDA contract — inline text, referenced images, optional logo

Three properties distinguish the RDA payload from RSA and PMax, all validated live:

- **Text is inline.** Headlines, the long headline, descriptions, and business name are plain strings in the ad itself — *not* text-asset resource names. (PMax asset groups require text assets; RSAs take inline text too but have no image slots. Don't transplant the PMax pattern here.)
- **Logo is OPTIONAL** (live probe finding). An RDA created with no `logoImages` at all is valid and serves. This is a real difference from PMax, where a logo is a hard minimum on the asset group — don't block a Display launch waiting for a logo file.
- **Field shape:**

| Field | Min–Max | Form | Constraint |
|---|---|---|---|
| `headlines` | 1–5 | inline text | ≤30 chars each |
| `longHeadline` | exactly 1 | inline text | ≤90 chars, required |
| `descriptions` | 1–5 | inline text | ≤90 chars each |
| `businessName` | 1 | inline string | ≤25 chars, required |
| `marketingImages` | 1–15 | IMAGE asset resourceName | **1.91:1**, required |
| `squareMarketingImages` | 1–15 | IMAGE asset resourceName | **1:1**, required |
| `logoImages` | 0–5 | IMAGE asset resourceName | optional |

(Limits as enforced by our `create_responsive_display_ad` schema; minimums confirmed live — an RDA with 1 of each required field served.) Variety guidance from `references/rsa-writing.md` applies to the text pool: distinct angles, not rewordings.

### Targeting — what we can read, what we can't write

Honest scope, because Display targeting is where tools overpromise:

- **Readable via `run_gaql`:** where ads actually served and how each slice performed — placements (`detail_placement_view`, `group_placement_view`), topics (`topic_view`), display keywords (`display_keyword_view`). This supports real recommendations: find placements burning spend with zero conversions, find topics that work.
- **Not writable with our tools:** placement/topic/audience targeting criteria, placement exclusions, and content exclusions have no write surface here. A new Display campaign launches with Google's default (broad) targeting. Say this plainly: we can *diagnose* targeting with data and *advise* exact changes, but the user applies targeting edits in the Google Ads UI.
- **Remarketing / RLSA:** we have **no user-list or audience tools** — building remarketing lists, Customer Match uploads, or attaching audiences is entirely outside this surface. If the user's actual goal is remarketing, advise it honestly as a UI/off-platform task; what we *can* contribute is the creative (this reference) and the measurement (GAQL) around a remarketing campaign once it exists.

## GAQL/tool examples

**1. The launch sequence** (writes — confirm with the user; placeholders, never real IDs):

```
create_image_asset            { customerId: "<customer-id>", ... }   → marketing (1.91:1) + square (1:1) asset resourceNames
create_display_campaign       { customerId: "<customer-id>", name, dailyBudget, biddingStrategy: "MAXIMIZE_CLICKS" }   → PAUSED
create_ad_group               { customerId: "<customer-id>", campaignId: "<campaign-id>", type: "DISPLAY_STANDARD", cpcBid }
create_responsive_display_ad  { customerId: "<customer-id>", adGroupId: "<ad-group-id>", finalUrl, headlines, longHeadline,
                                descriptions, businessName, marketingImages, squareMarketingImages }   # no logo needed
update_campaign               { customerId: "<customer-id>", campaignId: "<campaign-id>", status: "ENABLED" }
```

**2. Where did Display ads actually serve?** (placement audit)

```
SELECT detail_placement_view.display_name,
       detail_placement_view.placement_type,
       metrics.impressions, metrics.clicks, metrics.cost_micros, metrics.conversions
FROM detail_placement_view
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.cost_micros DESC
```

**Decision it supports:** spot placements consuming spend with no conversions — the Display analog of search-term mining. We can name the offenders precisely; the exclusion itself is a UI step (scope note above).

**3. Display campaign health, judged on Display terms:**

```
SELECT campaign.name, metrics.impressions, metrics.clicks, metrics.ctr,
       metrics.average_cpc, metrics.cost_micros, metrics.conversions,
       metrics.view_through_conversions
FROM campaign
WHERE campaign.advertising_channel_type = 'DISPLAY'
  AND segments.date DURING LAST_30_DAYS
```

**Decision it supports:** compare CPC and cost-per-outcome against the account's own Search numbers, expect CTR an order of magnitude lower (sourced above), and read `view_through_conversions` separately — they indicate post-impression influence but are weaker evidence than click conversions; never sum them silently into the conversion column.

## Pitfalls

- **Judging Display by Search CTR.** A Display CTR that would be a disaster on Search can be normal here (benchmark direction sourced above). The relevant questions are cost per reach and cost per (assisted) outcome.
- **Byte-identical images** → `DUPLICATE_ASSETS_WITH_DIFFERENT_FIELD_VALUE` (live finding). Different creatives, or link the same asset twice — never create it twice.
- **Wrong ad group type.** The RDA must land in a `DISPLAY_STANDARD` ad group under a `DISPLAY` campaign; our `create_ad_group` takes the type explicitly.
- **Conversion bidding without tracking.** `MAXIMIZE_CONVERSIONS`/`MAXIMIZE_CONVERSION_VALUE` on an untracked account fails or starves; open with `MAXIMIZE_CLICKS` and graduate per `references/bidding-spectrum.md`.
- **Blocking launch on a logo.** Logo is optional (live finding) — required images are exactly one 1.91:1 and one 1:1.
- **Teardown order is child → parent** (live finding): ad → ad group → campaign. Library image assets cannot be deleted at all — only unlinked (Google constraint); they remain inert.
- **Implying targeting control we don't have.** Placement exclusions, audience attachment, and remarketing lists are not on our write surface — recommend precisely, but label the apply step as the user's.
- **EU political declaration:** v21 campaign creates require `containsEuPoliticalAdvertising` (live finding); our `create_display_campaign` sends it automatically — relevant only if debugging raw mutates.

## Sources

- WordStream/LocaliQ — "Google Ads Benchmarks 2024": https://www.wordstream.com/blog/2024-google-ads-benchmarks (accessed 2026-06). 2024 search average CTR 6.42%.
- WordStream — all-industry Google Ads benchmarks (earlier-period historical data): https://www.wordstream.com/ppc-benchmarks (accessed 2026-06). Directional search ~3.17% vs display ~0.46% CTR gap.
- Live v21 Display probe (this project, validated on a test account): full create flow first-try success; **logo optional on RDA**; inline text (not text assets); sequential (non-atomic) build; content-hash asset dedup error; child→parent teardown; `containsEuPoliticalAdvertising` requirement.
- Tool contracts (this repo): `create_display_campaign`, `create_responsive_display_ad`, `create_ad_group`, `create_image_asset` schemas — field counts, character limits, aspect ratios, bidding defaults.
- Cross-references: `references/bidding-spectrum.md` (bidding graduation), `references/conversion-tracking-first.md` (gate), `references/rsa-writing.md` (text variety), `references/ppc-math.md` (impression share, economics).
