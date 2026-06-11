# Reference — Attribution, Consent Mode, Enhanced Conversions (Advisory — Mostly Off-Platform)

These three topics decide how much of the truth your conversion data tells: **attribution** decides which click gets credit, **Consent Mode** decides whether EEA traffic is measurable at all, and **enhanced conversions** decide how much of the measurement loss gets recovered. They matter — and almost none of them is operable from our tool surface. This reference exists to give correct advice and to be explicit about the boundary, not to pretend we can push buttons we don't have.

**What we CAN do here:** read attribution and tracking state via `run_gaql`; create offline-import conversion actions and feed them (`create_conversion_action`, `upload_click_conversions` — see `references/conversion-design.md` and the gate in `references/conversion-tracking-first.md`).
**What we CANNOT do here:** set or change a conversion action's attribution model, install or modify Consent Mode tags/CMP banners, or set up enhanced conversions. Those are Google Ads UI, site-tag, or CMP work — every recommendation below labels its apply-step accordingly.

## When to use

- Conversion counts dropped or diverged from the CRM/analytics and the account has **EEA traffic** — consent is the first suspect.
- The user asks "which attribution model should I use?" or wonders why conversion columns changed without anyone touching campaigns.
- Smart bidding underperforms on an account whose measurement is visibly leaky (few conversions despite healthy click volume) — see signal hygiene in `references/bidding-spectrum.md`; this reference covers the *measurement* causes.
- **Not** for designing the conversion actions themselves (`references/conversion-design.md`) or for the basic does-tracking-exist gate (`references/conversion-tracking-first.md`).

## Decision framework

### Attribution models — what's left, and what to recommend

The menu collapsed: Google Ads now offers **two** models — **last click** and **data-driven** — with data-driven the **default for most conversion actions**; first click, linear, time decay, and position-based were discontinued and existing conversion actions on them were automatically upgraded to data-driven (per Google Ads Help, "About attribution models", accessed 2026-06; the deprecated models were removed in 2023). The model applies to website and Google Analytics conversion actions and changes how the "Conversions" column counts — which means it also changes what Target CPA/ROAS optimize toward (same source).

Practical advice: **data-driven is the default and the right answer for almost everyone**; recommend last click only when the user needs deliberately conservative, single-touch bookkeeping (e.g. matching a finance report) and accepts that smart bidding then optimizes on that narrower view. Fractional conversion numbers in reports are a *consequence* of data-driven credit-splitting, not a bug — warn users before they see them.

**Scope, plainly:** there is **no attribution-model parameter** on our `create_conversion_action` and no tool to change the model on an existing action. Setting it is a Google Ads UI step (conversion action settings). What we *can* do is read the current state:

### Consent Mode v2 — the EEA measurement gate

For end users in the EEA, advertisers must collect consent for the use of personal data and pass the signals to Google to keep ad personalization and measurement; Consent Mode v2 adds two parameters — `ad_user_data` (consent to send user data for advertising) and `ad_personalization` (consent for personalized advertising) — on top of the original storage parameters (per Google Ads Help, "Updates to consent mode for traffic in EEA", accessed 2026-06). Enforcement is real: since March 2024, Customer Match lists in the EEA require both consent fields set to GRANTED, and unconsented EEA user data is not processed for personalization (per Google Ads Help, "How to provide consent for Customer Match", accessed 2026-06).

Symptoms that point here: EEA-heavy account, conversions visibly below CRM reality, remarketing lists not filling. The diagnosis conversation belongs to us; the fix does not:

- **Off-platform, entirely:** implementing Consent Mode v2 lives in the site's tag stack (gtag/Tag Manager) and consent banner/CMP. We cannot read consent-mode status via GAQL, cannot install tags, and should say exactly that.
- **What we can contribute:** the offline-upload path (`upload_click_conversions`) is click-ID-based and gives the account a conversion signal while the tag-side work is being done — it complements, not replaces, a compliant consent setup. One honest caveat: Google's API supports attaching consent signals to uploaded click conversions, but our `upload_click_conversions` schema does not currently expose a consent field — for EEA-sensitive uploads, flag this limitation rather than glossing it.

### Enhanced conversions — recovering match-loss, tag-side

Enhanced conversions supplement existing conversion tags by sending **hashed first-party data** (e.g. email addresses, normalized then SHA-256 hashed) that Google matches against signed-in accounts, improving measurement accuracy where cookies/click IDs fall short; a "for leads" variant upgrades offline conversion import by matching on user-provided data instead of click IDs (per Google Ads Help, "About enhanced conversions", accessed 2026-06).

When to recommend it: lead-gen businesses losing gclid continuity (long forms, cross-device journeys), or any account where the consent/cookie environment starves click-ID matching. **Scope, plainly:** setup runs through the Google tag, Tag Manager, or a Google Ads API surface we have not implemented — none of it is operable from our tools. Our nearest supported alternative remains classic gclid/gbraid/wbraid offline upload via `upload_click_conversions`, which works today but requires the click ID to survive into the user's CRM (`references/conversion-tracking-first.md` step 2 covers that flow).

## GAQL/tool examples

**1. Read each conversion action's attribution model** (we can diagnose, even though we can't set it):

```
SELECT conversion_action.name,
       conversion_action.type,
       conversion_action.attribution_model_settings.attribution_model,
       conversion_action.attribution_model_settings.data_driven_model_status,
       conversion_action.status
FROM conversion_action
WHERE conversion_action.status = 'ENABLED'
```

**Decision it supports:** spot actions still on `GOOGLE_ADS_LAST_CLICK` (candidates for a UI switch to data-driven) and check `data_driven_model_status` where data-driven is in use. Recommend the change; label the apply-step as Google Ads UI.

**2. Quantify the suspected EEA measurement gap** (indirect — consent state itself has no GAQL field):

```
SELECT user_location_view.country_criterion_id,
       metrics.clicks, metrics.conversions, metrics.cost_micros
FROM user_location_view
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.clicks DESC
```

Be honest about method: there is no GAQL surface for consent-mode status, so the evidence is indirect — compare conversions-per-click between EEA and non-EEA countries (resolve `country_criterion_id` via `search_geo_targets`), and compare account conversions against the user's CRM counts for the same window. A large EEA-specific shortfall is *evidence toward* a consent/measurement gap, not proof — present it that way.

**3. What we can actually operate** — the offline pipeline, unchanged: `create_conversion_action` (type `UPLOAD_CLICKS`) → verify via the `conversion_action` query → `upload_click_conversions` with gclid/gbraid/wbraid + timezone-offset datetime + `orderId` dedup. Full contract in `references/conversion-design.md`.

## Pitfalls

- **Implying we can set the attribution model.** We can read it (query 1) and recommend; the setting itself is UI-only from our surface. Never present it as a tool action awaiting confirmation.
- **Comparing conversion counts across the model change.** An action upgraded to data-driven splits credit fractionally; before/after comparisons spanning the switch mis-read bookkeeping changes as performance changes. Note the model in any `references/change-impact.md` window that crosses it.
- **Blaming campaigns for a consent problem.** On EEA-heavy accounts, falling conversions with stable clicks is a measurement hypothesis before it is a performance hypothesis. Check the gap (example 2) before restructuring anything.
- **Selling enhanced conversions as our deliverable.** It's tag-side. Recommend it with the sourced rationale, hand the implementation to the user's web team, and offer the offline-upload path as what we run today.
- **Forgetting smart bidding inherits all of this.** Attribution model and measurement coverage shape the conversion stream that tCPA/tROAS learn from (`references/bidding-spectrum.md` signal hygiene). Fixing measurement often beats touching targets.

## Sources

- Google Ads Help — "About attribution models": https://support.google.com/google-ads/answer/6259715 (accessed 2026-06). Two remaining models (last click, data-driven), data-driven as default, 2023 discontinuation and auto-upgrade of first click/linear/time decay/position-based, effect on conversion columns and smart bidding.
- Google Ads Help — "Updates to consent mode for traffic in European Economic Area (EEA)": https://support.google.com/google-ads/answer/13695607 (accessed 2026-06). `ad_user_data` / `ad_personalization` parameters; consent signals required for personalization and measurement.
- Google Ads Help — "How to provide consent for Customer Match": https://support.google.com/google-ads/answer/14546648 (accessed 2026-06). March 2024 EEA enforcement; both consent fields GRANTED required; unconsented EEA data not processed.
- Google Ads Help — "About enhanced conversions": https://support.google.com/google-ads/answer/9888656 (accessed 2026-06). Hashed (SHA-256) first-party data matching, normalization, web and for-leads variants, tag/GTM/API implementation paths.
- Tool contracts (this repo): `create_conversion_action` (no attribution parameter), `upload_click_conversions` (no consent field) — the basis of every scope statement above.
- Cross-references: `references/conversion-tracking-first.md` (gate + offline flow), `references/conversion-design.md` (action design, upload contract), `references/bidding-spectrum.md` (signal hygiene), `references/change-impact.md` (measurement windows).
