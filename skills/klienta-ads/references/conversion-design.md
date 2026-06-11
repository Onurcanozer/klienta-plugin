# Reference — Conversion Action Design: Category, Counting, Value

`references/conversion-tracking-first.md` decides *whether* tracking exists; this reference decides *what to build* when it doesn't — because a badly designed conversion action poisons everything downstream. Category, counting type, and value settings are the levers; smart bidding (see `references/bidding-spectrum.md`) will optimize exactly what these settings tell it to, right or wrong.

Write surface: `create_conversion_action` and `upload_click_conversions`. Verification is read-only `run_gaql` against Google Ads API **v21**. The category enum below reflects a live v21 finding, not just documentation.

## When to use

- The tracking gate is closed (`NOT_CONVERSION_TRACKED`) and the user wants conversion-based anything — build the action first.
- Tracking exists but the *design* is suspect: every conversion worth the same default value, purchases counted once per click, or duplicate offline uploads inflating counts.
- The user has offline outcomes (CRM-confirmed sales, qualified phone leads) that never make it back into Google Ads — the `UPLOAD_CLICKS` path is exactly for this.

## Decision framework

**Category — what kind of outcome is this?** Valid v21 values for `create_conversion_action`: `DEFAULT`, `PURCHASE`, `SIGNUP`, `SUBMIT_LEAD_FORM`, `CONTACT`. ⚠️ **`LEAD` is not a valid v21 ConversionActionCategory** — passing it fails; this is a live finding from our own probes, and a common trap because "lead" is the natural word for lead-gen. Map a generic lead to `SUBMIT_LEAD_FORM` (form fill) or `CONTACT` (call/message), and use `DEFAULT` only when nothing fits. Category drives reporting segmentation and goal grouping, so pick the closest real meaning rather than defaulting everything.

**Counting type — one per click or every conversion?** Google's guidance: count **every** conversion for purchases (each sale adds value — someone buying 3 rooms and 2 cars is 5 conversions) and **one** per click for leads (a person who submits the same form three times is still one lead) (per Google Ads Help, "About conversion counting options", accessed 2026-06). Our parameter: `countingType: MANY_PER_CLICK` for purchases, `ONE_PER_CLICK` for leads. Getting this backwards either inflates lead counts with duplicates or undercounts multi-purchase customers — and smart bidding inherits the distortion.

**Value — does this conversion carry money?** If conversions have real, differing values (e-commerce), pass per-conversion values in the upload and skip `alwaysUseDefaultValue`. If every conversion is genuinely worth about the same (fixed-price lead), `defaultValue` + `defaultCurrencyCode` is honest. The trap is stamping one default on outcomes that actually differ: value-based bidding (tROAS) then optimizes toward the cheapest conversions, not the best ones — the signal-hygiene failure described in `references/bidding-spectrum.md`. No real value data → say so and keep the account on count-based bidding.

**Lookback window.** `clickThroughLookbackWindowDays` accepts 1–90. Match it to the real sales cycle (measure with the `conversion_lag_bucket` query in `references/bidding-spectrum.md`): a window shorter than the cycle silently drops late conversions; needlessly long windows credit stale clicks.

## GAQL/tool examples

**1. Create the action** (write — show the user the exact definition and confirm first):

```
create_conversion_action {
  customerId, name: "CRM qualified lead",
  category: "SUBMIT_LEAD_FORM",        # NOT "LEAD" — invalid in v21
  type: "UPLOAD_CLICKS",               # offline import path (our default)
  countingType: "ONE_PER_CLICK",
  defaultValue: 250, defaultCurrencyCode: "TRY",   # only if leads truly share a value
  clickThroughLookbackWindowDays: 30
} → { resourceName, conversionActionId }
```

**2. Verify it exists and is usable:**

```
SELECT conversion_action.name, conversion_action.status, conversion_action.type,
       conversion_action.category, conversion_action.counting_type
FROM conversion_action
WHERE conversion_action.status = 'ENABLED'
```

**Decision it supports:** confirms the action is `ENABLED` with the intended category/counting before anything is uploaded or any bidding decision leans on it.

**3. Feed it offline conversions:**

```
upload_click_conversions {
  customerId, conversionActionResourceName,        # from step 1
  conversions: [{
    gclid: "Cj0K...",                              # or gbraid / wbraid (iOS)
    conversionDateTime: "2026-06-10 14:30:00+03:00",  # timezone offset REQUIRED
    conversionValue: 412.50, currencyCode: "TRY",
    orderId: "ORD-10293"                           # dedup key — makes retries safe
  }]
} → { uploaded, failed, errors[] }
```

Batches run with **partial failure**: bad rows are reported per-row (`errors[]` with index and message) without dropping the rest — report which rows failed and why, fix, re-send only those. Max 2,000 conversions per call (our tool limit); chunk larger sets.

## Pitfalls

- **`category: "LEAD"` fails on v21** (live finding — see above). Use `SUBMIT_LEAD_FORM` or `CONTACT`.
- **Uploading immediately after creation.** Google may take a few hours to fully process a brand-new conversion action before uploads succeed (noted in the `create_conversion_action` tool itself). Create, verify with query 2, schedule the first upload later — don't read early failures as a broken setup.
- **`conversionDateTime` without a timezone offset** is rejected. Format is `yyyy-mm-dd hh:mm:ss±hh:mm` in the account's timezone; the conversion time must postdate the click.
- **No `orderId` → duplicate conversions on retry.** Any retry or overlapping export double-counts, inflating both reports and smart-bidding signal. Always pass a stable transaction ID.
- **`alwaysUseDefaultValue` on variable-value outcomes** flattens the value signal tROAS depends on. Use it only when values genuinely don't vary.
- **Scope honesty:** we support offline click upload (`UPLOAD_CLICKS`). Website tag/GA4-based tracking is configured outside these tools — say so plainly and offer the offline path as what we can do here (same boundary as `references/conversion-tracking-first.md`).

## Sources

- Google Ads Help — "About conversion counting options": https://support.google.com/google-ads/answer/3438531 (accessed 2026-06). Every-conversion for purchases vs one-conversion for leads, with worked examples.
- Live v21 probe finding (this project): valid `ConversionActionCategory` values `DEFAULT`/`PURCHASE`/`SIGNUP`/`SUBMIT_LEAD_FORM`/`CONTACT`; `LEAD` rejected.
- Tool contracts (this repo): `create_conversion_action` and `upload_click_conversions` schemas — lookback 1–90 days, 2,000-row batch cap, partial-failure reporting, timezone-offset datetime format.
- Cross-references: `references/conversion-tracking-first.md` (the gate and create-first flow), `references/bidding-spectrum.md` (signal hygiene, lag measurement), `references/ppc-math.md` (what values should represent).
