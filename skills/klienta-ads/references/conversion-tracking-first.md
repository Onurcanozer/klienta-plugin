# Reference — Conversion-Tracking-First Gate

Optimizing a Google Ads account without conversion tracking is steering with the windshield painted over. Every "this converts, that doesn't" judgment, every smart-bidding strategy, every ROAS number depends on tracking being real. So tracking is a **gate**, not a step: if it's missing, you do not hand out conversion-based optimization advice — you fix tracking first.

Validated against Google Ads API **v21** via `run_gaql`.

---

## Step 1 — Detect tracking status (run this before optimizing)

```
SELECT customer.conversion_tracking_setting.conversion_tracking_status
FROM customer
LIMIT 1
```

**Decision it supports — the gate itself:**

- `CONVERSION_TRACKING_MANAGED_BY_SELF` or `..._BY_THIS_MANAGER` → tracking is set up. **Gate open.** You may proceed with conversion-based analysis and recommendations.
- `NOT_CONVERSION_TRACKED` → **gate closed.** Do not call anything "waste" or "winning" based on conversions; those numbers are meaningless. Route to setup.
- `UNKNOWN` / `..._BY_ANOTHER_MANAGER` → tracking may exist but you can't trust or manage it from here. Tell the user explicitly and don't optimize on conversion data you can't verify.

Cross-check what conversion actions actually exist and whether any are usable:

```
SELECT conversion_action.name, conversion_action.status,
       conversion_action.type, conversion_action.category
FROM conversion_action
```

**Decision it supports:** even with a "tracked" status, confirm there is at least one `ENABLED` conversion action. A paused or empty set means tracking is nominal, not functional.

---

## Step 2 — When the gate is closed: the create-first flow

If the user wants conversion-based optimization (smart bidding, "cut wasted spend", "improve ROAS") and tracking is absent, **stop optimizing and set up tracking** with the tools we have:

1. **Explain the gate** in one sentence: without a conversion action, the account can't tell which clicks become customers, so any optimization is guesswork.
2. **Create the conversion action** with `create_conversion_action`:
   - Defaults to `type: UPLOAD_CLICKS` — the right choice for **offline conversions** (e.g. a CRM-confirmed sale, a phone lead) imported back into Ads.
   - Pick `category` to match the goal (`PURCHASE`, `LEAD`, `SIGNUP`, or `DEFAULT`).
   - Choose `countingType`: `ONE_PER_CLICK` for leads (one qualified lead per click), `MANY_PER_CLICK` for purchases (a click can yield multiple sales).
   - Optionally set `defaultValue` + `defaultCurrencyCode` if conversions carry a known value.
   - This is a **write** → show the user the exact definition and confirm first.
3. **Verify** the action exists and is enabled (re-run the `conversion_action` query above). Note: Google may take a few hours to fully process a brand-new conversion action before uploads succeed.
4. **Feed conversions** with `upload_click_conversions` once you have offline conversion data:
   - Each row needs a click ID (`gclid`, `gbraid`, or `wbraid`), a `conversionDateTime` **with timezone offset** in the account's timezone (e.g. `2026-06-10 14:30:00+03:00`), and the `conversionActionResourceName` returned at creation.
   - Provide `orderId` so retries are safe (deduplicated).
   - Uploads run with partial-failure enabled: bad rows are reported per-row without dropping the whole batch — report which rows failed and why.

> **Scope honesty:** Klienta supports **offline click upload** tracking (`UPLOAD_CLICKS`). It does not install website tag-based tracking or Google Analytics linking. If the user needs on-site tag tracking, say so plainly — that's set up outside these tools — and offer the offline-upload path as what we can do here.

---

## Step 3 — Only now, optimize

With the gate open and at least one enabled, functioning conversion action, the conversion columns in `performance-triage.md` (`metrics.conversions`, zero-conversion waste, etc.) become trustworthy. Until then, restrict yourself to tracking-independent observations (impression share, CTR, spend distribution) and be explicit that conversion-based conclusions are unavailable.

---

## The gate in one line

> Check `conversion_tracking_status` first. If `NOT_CONVERSION_TRACKED`, don't optimize on conversions — run the `create_conversion_action` flow, verify it, then proceed. No tracking, no conversion-based advice.
