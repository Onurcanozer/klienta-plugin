---
name: klienta-audit
description: Read-only Google Ads account audit playbook for the Klienta MCP server. Use when the user asks to audit, review, health-check, or "look over" an account without making changes. Scans campaigns, keywords, budgets, search terms, quality, impression share, and conversion tracking with run_gaql, then produces a structured health report (score + findings + recommended actions). Never writes.
---

# Klienta — Read-Only Account Audit

A repeatable, **strictly read-only** account review. This skill never calls a write tool — its output is a health report a human (or the `klienta-ads` skill) can act on. If the user approves changes, hand off to **klienta-ads**, which owns the confirmation-and-write discipline.

> **Maintenance note:** This skill assumes the current 50-tool inventory and v21 GAQL field names. When the tool inventory or API version changes, update this playbook accordingly. (The mutating tools — keyword research, assets, bulk ops, shared negative-keyword lists, portfolio bidding, experiments, Performance Max, Display — belong to the **klienta-ads** workflow, not this read-only audit. `get_usage` reports the user's plan/usage and is read-only.)

## Tools used (read-only only)

- `list_accounts` — pick the customer ID; note currency, manager, test flags.
- `run_gaql` — every scan below (single query, or up to 20 in parallel).
- `search_geo_targets` — only if you need to interpret geo settings; not required for the core audit.
- `get_changes` — recent change history (this server's audit log + Google `change_event`); read-only.
- `get_guardrails` — read the user's current safety limits; read-only.

**Never** call any mutating tool from this skill — no `create_*`, `update_*`, `add_*`, `remove_*`, `upload_*`, `set_guardrails`, or `undo_change`. Auditing observes; it does not touch the account. If a finding warrants a change, hand off to **klienta-ads**.

## How to run the audit

1. `list_accounts` → choose the account. If it's under a manager, pass `loginCustomerId`.
2. Run the six scans below with `run_gaql` (read-only). Each maps to a section of the report.
3. Fill in the **report template** and present it. Convert every `*_micros` value to account currency units (÷ 1,000,000).

All scans validated against Google Ads API **v21**.

### Scan 1 — Conversion tracking (run first; it weights everything)
```
SELECT customer.conversion_tracking_setting.conversion_tracking_status
FROM customer LIMIT 1
```
`NOT_CONVERSION_TRACKED` is a critical finding: every conversion-based metric in the rest of the audit is untrustworthy. Flag it at the top of the report and cap the score (see scoring).

### Scan 2 — Campaign performance
```
SELECT campaign.name, campaign.status, campaign.advertising_channel_type,
       metrics.cost_micros, metrics.clicks, metrics.conversions,
       metrics.ctr, metrics.average_cpc
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.cost_micros DESC
```
Finding source: spend distribution, obvious zero-conversion spenders, CTR outliers.

### Scan 3 — Impression share (budget vs rank)
```
SELECT campaign.name,
       metrics.search_impression_share,
       metrics.search_budget_lost_impression_share,
       metrics.search_rank_lost_impression_share
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
```
Finding source: whether growth is capped by budget or by quality/bid (see `klienta-ads/references/performance-triage.md` for the split).

### Scan 4 — Keyword quality
```
SELECT ad_group_criterion.keyword.text,
       ad_group_criterion.quality_info.quality_score,
       metrics.impressions, metrics.ctr
FROM keyword_view
WHERE segments.date DURING LAST_30_DAYS
  AND metrics.impressions > 0
ORDER BY ad_group_criterion.quality_info.quality_score ASC
```
Finding source: low Quality Score keywords dragging cost and rank.

### Scan 5 — Wasted spend on search terms
```
SELECT search_term_view.search_term, metrics.cost_micros,
       metrics.clicks, metrics.conversions, campaign.name
FROM search_term_view
WHERE segments.date DURING LAST_30_DAYS
  AND metrics.conversions = 0
  AND metrics.clicks > 3
ORDER BY metrics.cost_micros DESC
```
Finding source: off-intent queries draining budget → negative-keyword opportunity (only meaningful if Scan 1 says tracking is active).

### Scan 6 — Budgets & ad coverage
```
SELECT campaign_budget.name, campaign_budget.amount_micros, campaign_budget.status
FROM campaign_budget
```
```
SELECT ad_group_ad.ad.id, ad_group_ad.ad_strength, ad_group_ad.status
FROM ad_group_ad
WHERE segments.date DURING LAST_30_DAYS
```
Finding source: budget sizing, and ad strength / ad-group coverage gaps (ad groups with weak or no active ads).

### Scan 7 — Recent changes (optional context)
Call `get_changes` (with the account's `customerId`) to see what changed recently — both this server's own audit log (every mutating tool call, with who/what/when and whether it succeeded) and Google's `change_event`. This is read-only and useful for explaining a sudden performance shift ("budget was raised 3 days ago", "an ad was paused"). It is context, not a scored finding; mention it in the report when a change plausibly explains what the scans show. Do not act on it — surfacing the relevant `auditId` lets the user ask **klienta-ads** to `undo_change` if a change was a mistake.

### Scan 8 — Network segmentation (Google vs Search Partners)
A Search campaign can serve on Google search itself and on the Search Partners network. They often perform very differently, and the campaign-level totals hide it. Segment by network:
```
SELECT campaign.name, segments.ad_network_type,
       metrics.cost_micros, metrics.clicks, metrics.conversions, metrics.ctr
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
```
Finding source: compare `SEARCH` (Google) vs `SEARCH_PARTNERS` rows for the same campaign. If Search Partners is taking meaningful spend at materially worse CTR/conversions than Google search — judged against this campaign's own Google numbers, not an external benchmark — that's a finding: the partner network may be diluting efficiency. The remediation (turning off Search Partners) is a campaign network setting; note that the current toolset's `update_campaign` covers status/name only, so flag it as a recommendation for the user to adjust in the Google Ads UI rather than implying the tool can change it.

## Health report template

Present exactly this structure:

```
# Account Health Report — <account name> (<CID>)
Window: Last 30 days · Currency: <CCC>

## Overall score: <0–100> — <Healthy | Needs attention | At risk>
Tracking: <Active | NOT TRACKED — results below are spend-only>

## Findings
| # | Area              | Severity | Evidence (query + numbers)                  | Recommended action (handoff: klienta-ads) |
|---|-------------------|----------|---------------------------------------------|--------------------------------------------|
| 1 | Conversion track. | High/Med/Low | <status value from Scan 1>              | <e.g. set up create_conversion_action>     |
| 2 | Wasted spend      | ...      | <term, spend, 0 conv — Scan 5>              | <review for add_negative_keywords>         |
| 3 | Budget vs rank    | ...      | <campaign, budget-lost X% / rank-lost Y%>   | <small update_campaign_budget OR fix QS>   |
| 4 | Keyword quality   | ...      | <keyword, QS=n — Scan 4>                     | <improve relevance / update_keyword>       |
| 5 | Ad coverage       | ...      | <ad group, ad_strength / status — Scan 6>   | <new create_responsive_search_ad>          |

## Top 3 actions, smallest reversible first
1. ...
2. ...
3. ...
```

Rules for the report:
- **Every finding cites the query and the actual numbers.** No claim without evidence.
- **Recommended actions name a real tool** from the 14-tool inventory and are framed as the smallest reversible step. The audit does **not** perform them — it hands off.
- If tracking is `NOT_CONVERSION_TRACKED`, the #1 finding is always "set up conversion tracking," and conversion-based findings are labeled spend-only.

## Scoring guidance (transparent, not magic)

Start at 100 and subtract for what the scans reveal; show the deductions so the score is auditable:
- **−40 cap:** tracking `NOT_CONVERSION_TRACKED` (you literally can't judge effectiveness) → score cannot exceed 60.
- **−5 to −15:** material wasted spend (Scan 5) scaled by how large a share of total spend it is.
- **−5 to −15:** campaigns budget-constrained while converting well (missed opportunity) or rank-constrained with low QS (Scans 3–4).
- **−5 to −10:** ad groups with weak/no active ads or poor ad strength (Scan 6).
- **−5:** large share of spend on REMOVED/PAUSED-then-rebuilt clutter or unclear structure.

Map the final number: **80–100 Healthy · 50–79 Needs attention · <50 At risk.** State the deductions next to the score so the user sees how it was derived.

## Presenting to the account owner

The health-report template above is the internal/operator format (scores, scans, GAQL-backed findings). When the audience is the **client/account owner**, translate it into plain language using `references/client-report.md` — state of the account, what was changed-and-verified, and a clearly-separated list of actions waiting for their approval. Keep money in currency, drop the jargon, and never imply a proposed action was already executed.

## Handoff

When the user wants to act on a finding, switch to **klienta-ads**, which enforces confirm-before-write and read-back-after-write. This audit skill stops at the recommendation.
