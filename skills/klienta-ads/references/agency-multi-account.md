# Reference — Agency Multi-Account Operation (MCC)

An agency or in-house operator running several Google Ads accounts works through a **manager account (MCC)**. This reference is how to navigate accounts under one manager, decide where attention should go, and run a repeatable monthly/quarterly review — within our toolset, and honest about where the manager surface ends.

## When this applies

Use this whenever the connected Google user can see more than one account, or any account is nested under a manager. The tell is `list_accounts`: it returns each account with a `manager` flag and currency. If you see a manager account, the child accounts under it are reached by passing `loginCustomerId` (the manager's 10-digit ID, e.g. `<mcc-id>`) on every call to that child.

## Navigating accounts

```
1. list_accounts                      → discover IDs, currencies, manager flags
2. list_accounts(loginCustomerId=<mcc-id>)  → accounts under that manager
3. run_gaql(customerId=<child>, loginCustomerId=<mcc-id>)  → read a child
```

Every read or write against a manager-nested account needs both `customerId` (the child, `customers/<cid>/...`) **and** `loginCustomerId` (the manager). Omitting `loginCustomerId` on a nested account is the most common multi-account error — the call fails or hits the wrong account. Guardrails, budgets, and currencies are **per child account**: there is no cross-account bulk write in our surface, so a change to ten accounts is ten confirmed operations, each read back on its own account (`references/campaign-structure.md` honest-limits discipline applies per account).

**Scale limits (for context, not something we administer):** a manager account can be linked to up to 85,000 non-manager accounts, with the *active* limit tiered by trailing-12-month spend — 50 accounts under $10,000 USD/mo, 2,500 up to $500,000, and 85,000 above (per Google Ads Help, "About account limits for manager accounts," accessed June 2026). An individual account can be directly managed by at most 5 manager accounts (per Google Ads Help, accessed June 2026). Linking and access changes are off-platform — done in the Google Ads UI, not through our tools.

## Account prioritization — where attention goes

You can't review every account equally every day. Rank by **leverage**, not by size alone. Pull a one-row-per-account snapshot to triage:

```
SELECT customer.descriptive_name, metrics.cost_micros, metrics.conversions,
       metrics.cost_per_conversion, metrics.search_budget_lost_impression_share
FROM customer
WHERE segments.date DURING LAST_7_DAYS
```

(Run per child with its `loginCustomerId`; correlate the results yourself — there's no account it returns for siblings in one call.)

Prioritize, in order:

1. **Spending + off track** — high `cost_micros`, CPA above the client's break-even (`references/ppc-math.md`, `references/benchmark-calibration.md`). Money burning now.
2. **Constrained winners** — profitable but high `search_budget_lost_impression_share`; a budget move captures proven demand (`references/optimization-loops.md`).
3. **No conversion tracking** — any account where tracking is off is flying blind; fix the gate before optimizing anything (`references/conversion-tracking-first.md`).
4. **Quiet + healthy** — low spend, stable CPA. Light-touch; review on the monthly cycle, not daily.

The daily loop (`references/daily-operator.md`) runs against the top of this list; the long tail rolls up into the periodic review below.

## QBR / monthly review structure

A monthly or quarterly business review (QBR) is the client-facing rollup, not a new analysis — it consumes the same reads, summarized over a longer window. Build it per account on the client-report shape (`references/client-report.md`); never blend accounts into one report.

Pull the period and its prior period for trend (the only honest "good/bad" basis we have — `references/benchmark-calibration.md`):

```
SELECT campaign.name, metrics.cost_micros, metrics.conversions,
       metrics.conversions_value, metrics.cost_per_conversion, metrics.ctr
FROM campaign
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.cost_micros DESC
```

Then re-run with the prior 30 days (explicit `BETWEEN` dates) and pair the change with `get_changes` so every metric move is attributable to a known action (`references/change-impact.md`). The review structure:

- **Where it stands** — spend, results, and trend vs prior period (lead with tracking status if it's off).
- **What changed and what it did** — `get_changes` cross-referenced to before/after metrics; separate done-and-verified from proposed.
- **What's next** — the prioritized asks for the coming period, framed as the smallest reversible step (`references/client-report.md` approval-list discipline).

## Pitfalls

- **Wrong `loginCustomerId`** — reads/writes silently hit the wrong account or fail; always pair child `customerId` with its manager ID.
- **Mixed currencies** — accounts under one MCC can differ; never sum `cost_micros` across accounts without normalizing, and always divide micros by 1,000,000 in client output.
- **Treating the MCC as a bulk tool** — it isn't, in our surface. Per-account confirmation and read-back still apply to each change.
- **One report for many accounts** — flatters or hides per-account truth; one report = one account, one period.

## Sources

- Google Ads Help, "Manager Accounts (MCC): About account limits for manager accounts" (85,000 linked cap; active tiers 50 / 2,500 / 85,000 by trailing-12-month spend) — accessed June 2026: https://support.google.com/google-ads/answer/7526520
- Google Ads Help, manager-account linking limits (an account directly managed by at most 5 manager accounts) — accessed June 2026: https://support.google.com/google-ads/answer/7456530
