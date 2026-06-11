# Reference — Performance Max Playbook

Performance Max (PMax) is one campaign that serves across every Google surface — Search, Display, YouTube, Discover, Gmail, Maps — driven entirely by conversion-based bidding and the asset group you hand it. You give it goals, a budget, a landing page, and a pile of text + image assets; Google decides placement, audience, and creative mix. That trades control for reach, and it changes how you build, diagnose, and tear down. This reference is how to run PMax within our toolset, grounded in a live v21 build we ran end-to-end.

Our PMax surface is four tools: `create_image_asset`, `create_performance_max_campaign`, `create_asset_group`, `update_asset_group`. Everything below binds to those plus `run_gaql` for reads. All asset minimums and error behaviour here were verified against the **Google Ads API v21** in a live probe against a dedicated test account (2026) — where a claim rests on that probe rather than a published spec, it says so.

---

## When to use

Reach for PMax when the goal is conversions or conversion value and the advertiser can't (or won't) hand-curate keywords and placements across surfaces:

- **Conversion tracking exists and is trusted.** PMax has no manual bids — it *only* optimizes toward a conversion signal. With no conversions, or a thin/wrong signal, it spends blind. Gate on `references/conversion-tracking-first.md` before building, exactly as you would for any smart-bidding strategy (`references/bidding-spectrum.md`).
- **You want incremental reach beyond Search**, or e-commerce with a product feed, or a lead-gen advertiser who wants Google to find conversions wherever they are.
- **You can supply real, diverse creative** — multiple genuinely different images and several distinct headlines/descriptions. PMax starves on thin creative.

Do **not** reach for PMax when: there's no conversion tracking yet (build a Search campaign with `create_search_campaign` first); the advertiser needs query-level control or transparency (PMax reporting is a black box — see Pitfalls); or the only goal is protecting brand terms (PMax can cannibalize Search brand traffic — see the cannibalization section).

---

## Decision framework

**1. Bidding goal first.** `create_performance_max_campaign` takes `biddingStrategy` = `MAXIMIZE_CONVERSIONS` (default) or `MAXIMIZE_CONVERSION_VALUE`. Pick value-based only when conversions carry meaningful, accurately-passed values (revenue). Lead-gen with flat conversions → `MAXIMIZE_CONVERSIONS`. Add target CPA/ROAS later via `update_campaign` once there's volume — starting with a hard target on a cold campaign throttles learning (`references/bidding-spectrum.md`).

**2. Budget.** The campaign is created **PAUSED** with a `dailyBudget`. Size it so the campaign can clear several conversions while learning; a budget too small to gather signal keeps PMax permanently in exploration. Confirm the number with the user before enabling — this is the spend lever.

**3. Asset group = your only creative + targeting lever.** PMax has no keywords and no manual placements. The asset group's text, images, and final URL are the entire input you control. One asset group per distinct theme/audience/landing-page; use `create_asset_group` to add more to the same campaign when you have genuinely separate themes (e.g. two product lines), not to pad asset count.

**4. Required assets — know the two different "minimums".** There is the **API hard floor to enable** (what the v21 mutate actually accepts) and the **published spec minimum** (what Google's asset-group builder/ad-strength expects for full eligibility). They differ, and being honest about which is which matters:

| Field | API floor to ENABLE (v21 probe, 2026) | Google published min / max | Char / ratio |
|---|---|---|---|
| HEADLINE | 3 | 3 / 15 | ≤30 chars (≥1 should be ≤15) |
| DESCRIPTION | 2 | 2 / 5 | ≤90 chars |
| LONG_HEADLINE | 1 | 1 / 5 | ≤90 chars |
| BUSINESS_NAME | 1 | 1 | ≤25 chars |
| MARKETING_IMAGE | 1 | **4** / 20 | 1.91:1, ≥600×314 |
| SQUARE_MARKETING_IMAGE | 1 | **4** / 20 | 1:1, ≥300×300 |
| LOGO | 1 | 1 / 5 | 1:1, ≥128×128 |

The API let us **enable** an asset group with a single marketing image, single square image, and single logo (probe, 2026). But Google's published spec lists **4 marketing + 4 square images** as the asset-group minimum for full serving/ad-strength. So "it enabled" ≠ "it serves everywhere well." `create_performance_max_campaign` accepts up to 15 headlines, 5 descriptions, and a single `imageAssets` trio — to push toward the published minimum and better ad strength, upload more images first with `create_image_asset` and add a richer asset group, or expect a "Poor"/limited-inventory rating on a bare-minimum group. Optional video: if you supply none, Google auto-generates one from your assets.

**5. Brand Guidelines.** Our tool sets `brandGuidelinesEnabled: false` deliberately — the classic model, where business name and logo live in the asset group. With Brand Guidelines *enabled* (the v21 default if you build raw), business name + square logo become **campaign-level** assets required at create time, and a plain create fails (see Pitfalls). The tool removes that footgun for you; just know why it's off.

---

## GAQL / tool examples

**Build — one atomic call.** `create_performance_max_campaign` issues a single `googleAds:mutate` that creates budget + campaign + asset group + every required asset together (verified live: 13 operations, one response, probe 2026). This is not optional sequencing — an **empty asset group cannot be created**, so the assets must land in the same atomic mutate (see Pitfalls). Upload images first:

```
create_image_asset { data: <base64 1.91:1 PNG>, name: "hero-wide" }   → resourceName
create_image_asset { data: <base64 1:1 PNG>,    name: "hero-square" } → resourceName
create_image_asset { data: <base64 1:1 logo>,   name: "logo" }        → resourceName

create_performance_max_campaign {
  name, dailyBudget, finalUrl,
  headlines: [≥3], descriptions: [≥2], longHeadline, businessName,
  imageAssets: { marketing, square, logo },   // the resourceNames above
  biddingStrategy: "MAXIMIZE_CONVERSIONS"
}                                              // created PAUSED
```

**Enable** after confirming budget: `update_asset_group { assetGroupId, status: "ENABLED" }` (only succeeds once minimums are met — probe-verified), then `update_campaign` to enable the campaign.

**Read performance** (PMax appears in the `campaign` resource like any campaign):

```
SELECT campaign.name, campaign.advertising_channel_type,
       metrics.cost_micros, metrics.conversions, metrics.conversions_value
FROM campaign
WHERE campaign.advertising_channel_type = 'PERFORMANCE_MAX'
  AND segments.date DURING LAST_30_DAYS
ORDER BY metrics.cost_micros DESC
```

**Asset-group level** (which group is carrying the campaign):

```
SELECT asset_group.name, asset_group.status,
       metrics.cost_micros, metrics.conversions
FROM asset_group
WHERE segments.date DURING LAST_30_DAYS
```

### Asset-group signals — be honest about what we can and can't set

Google lets you give PMax two steering signals: an **audience signal** (a hint of who converts — your segments, custom audiences) and **search themes** (phrases you expect to convert). These speed learning; they are *hints*, not targeting.

**Our tools do not set either.** `create_performance_max_campaign` / `create_asset_group` take creative + final URL only — no `audience_signal`, no `search_themes` parameter. So with our surface you ship the asset group without those signals and let PMax learn from the conversion data alone. Don't claim to configure an audience signal we can't write; if the advertiser needs one, that's a Google Ads UI step today, and worth saying plainly.

### Search cannibalization — diagnose, then structure around it

PMax can serve on queries your Search campaigns already cover, including **brand** terms, and Google's auction logic generally lets PMax take eligible traffic. The risk: PMax quietly absorbs cheap brand conversions, inflating its apparent performance while starving (and taking credit from) your Search brand campaign.

Diagnose by comparing surfaces and watching brand Search volume over time:

```
SELECT campaign.name, campaign.advertising_channel_type,
       metrics.cost_micros, metrics.conversions, metrics.impressions
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
ORDER BY campaign.advertising_channel_type
```

PMax does expose **aggregated** search categories via `campaign_search_term_insight` (grouped `category_label` themes, not raw query rows) — useful to see roughly what PMax matched, but it is not the keyword-level `search_term_view` you get on Search. Treat it as directional.

What we *can* do about cannibalization within our toolset: keep brand on a dedicated Search campaign with tight structure (`references/campaign-structure.md`), and watch the brand campaign's impression share and conversions for a drop after PMax launches (`references/change-impact.md`). What we *cannot* do: set PMax brand exclusions / negative keywords at the campaign level — there's no tool for it, and account-level brand exclusions for PMax are a Google-rep / UI path. Say so rather than implying a fix.

### Teardown — order is not optional

Remove the **asset group first, then the campaign**:

```
update_asset_group { assetGroupId, status: "REMOVED" }   // first
remove_campaign    { campaignId }                          // second
```

`update_asset_group` with `status: "REMOVED"` issues a real remove operation. If you remove the campaign first, the asset group is locked forever (see Pitfalls). Removals are permanent and not undoable — confirm explicitly and prefer pausing while testing.

---

## Pitfalls

- **Empty asset group is rejected.** A bare `assetGroups:mutate` with no assets fails with every `NOT_ENOUGH_*_ASSET` at once (probe, 2026). That's why the build is one atomic `googleAds:mutate` — don't try to create the group then add assets in separate calls.
- **Brand Guidelines default.** Build PMax raw in v21 and `brandGuidelinesEnabled` defaults true, so a create without a campaign-level business-name + square-logo asset fails (`REQUIRED_BUSINESS_NAME_ASSET_NOT_LINKED`, `REQUIRED_LOGO_ASSET_NOT_LINKED`). Our tool sets it `false`; surface this error if it ever appears.
- **Teardown order.** Removing the campaign first locks the asset group: `CANNOT_MUTATE_ASSET_GROUP_FOR_REMOVED_CAMPAIGN` — it can no longer be removed and is left inert (probe left exactly one such orphan, 2026). Asset group first, every time.
- **Image dedup by content.** Google dedupes assets by content hash. Two byte-identical images uploaded under different names collide: `DUPLICATE_ASSETS_WITH_DIFFERENT_FIELD_VALUE`. Use genuinely different images; to reuse one image in two roles, link the same asset twice rather than uploading it twice.
- **Library assets can't be deleted — only unlinked.** Google has no delete for `Asset` resources; unlinking leaves them orphaned-but-inert in the library. This is structural, not a bug — don't promise asset cleanup.
- **Reporting is a black box.** No keyword-level search terms, no placement-level transparency, limited control. Set expectations: PMax gives you aggregated category insight and surface/asset-level signals, not the granular `search_term_view` accountability of Search. If the advertiser needs that accountability, Search is the right channel.
- **Thin creative / bare minimum.** Enabling with one image each clears the API floor but earns a "Poor"/limited rating against Google's published 4-image spec — it won't serve well across inventory. More genuinely-different assets is the single biggest in-toolset quality lever.

---

## Sources

- **Live API probe, Google Ads API v21**, dedicated test account (2026) — empirical asset-group enable floor (1 marketing / 1 square / 1 logo), atomic-create requirement, Brand Guidelines default, teardown-order lock, content dedup, library-asset un-deletability. *(internal probe; sprint15)*
- Google Ads Help — *About text assets for Performance Max campaigns*, accessed 2026-06-11. https://support.google.com/google-ads/answer/14528373 (headlines 3–15, long headlines 1–5, descriptions 2–5, business name 1; character limits).
- Google Ads Help — *About image assets for Performance Max campaigns*, accessed 2026-06-11. https://support.google.com/google-ads/answer/14530211 (marketing image 1.91:1 min 4 / ≥600×314; square 1:1 min 4 / ≥300×300; logo 1:1 min 1 / ≥128×128; 5 MB max).
- Google Ads Help — *Best practices for asset groups in Performance Max campaigns*, accessed 2026-06-11. https://support.google.com/google-ads/answer/14528220 (asset-group completeness / "Poor" rating on minimum-only groups).
- Google Ads API — *campaign_search_term_insight* (v21), accessed 2026-06-11. https://developers.google.com/google-ads/api/fields/v21/campaign_search_term_insight_query_builder (aggregated search categories, not raw query rows).
- Tool surface: `src/tools/pmax.ts` — `create_image_asset`, `create_performance_max_campaign`, `create_asset_group`, `update_asset_group` (parameters, brandGuidelinesEnabled:false, error dictionary).
