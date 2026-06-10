# Reference — Account Archetype Playbooks

Different business models want different things from Search, so the same metric can be good for one and bad for another. These short postures tune the operating contract to two common archetypes. They are biases, not rules — read the account's own data first.

## Lead generation (local services, B2C inquiries, form/call leads)

**What success looks like:** qualified leads at or under a target cost; lead *quality* matters as much as count.

- **Conversion tracking is the whole game.** A "lead" is usually offline-qualified (a call that became a job, a form that became a customer). Use `create_conversion_action` (UPLOAD_CLICKS) + `upload_click_conversions` to feed back *qualified* leads, not raw form fills — otherwise you optimize toward cheap junk. See `references/conversion-tracking-first.md`.
- **Geography and intent are the main levers.** Tight geo targeting (`search_geo_targets`), and aggressive negative-keyword hygiene to cut research/DIY/"free"/job-seeker queries (`references/search-term-mining.md`, shared lists).
- **Bidding:** if qualified-lead conversion data is solid, a TARGET_CPA portfolio strategy fits; until then, manual/maximize-clicks with tight negatives.
- **Anti-patterns:** optimizing to form-fill volume (rewards junk); ignoring call quality; broad match without negatives bleeding into unrelated searches.

## SaaS / B2B

**What success looks like:** pipeline and revenue, not clicks or even raw signups — the buying cycle is long and multi-touch.

- **Value, not just count.** A free-trial signup is not a sale. Where possible, feed back *down-funnel* value (qualified opportunity, closed-won) via offline conversions so bidding chases revenue, not top-of-funnel noise. Consider conversion-value strategies (TARGET_ROAS / MAXIMIZE_CONVERSION_VALUE) only when tracked value is real.
- **Long lag.** Conversions land weeks after the click. Be patient with change-impact windows (`references/change-impact.md` — lean to 14-day or longer) and don't tune on a few days of a long cycle.
- **Intent segmentation.** Separate high-intent bottom-funnel terms (product/category/competitor-comparison) from broad research terms; structure and budget them differently (`references/campaign-structure.md`).
- **Anti-patterns:** judging by cost-per-signup when signups don't predict revenue; cutting a campaign for "no conversions" before the buying cycle has closed; smart-bidding on value you don't actually track.

## Using these
- **Confirm the archetype from data and the user**, don't assume from the URL alone.
- **Both archetypes still run the same loops** (daily/weekly/monthly) — the postures just change what counts as "good" and which levers to pull first.
- **No benchmark numbers here on purpose** — calibrate "good CPA/ROAS" against this account's own history and the user's margin (`references/benchmark-calibration.md`, `references/ppc-math.md`).
