# Reference — Agency Multi-Account Operation (MCC)

An agency or in-house operator runs several Google Ads accounts — across one or more **manager accounts (MCC)** and, in our surface, across one or more **linked Google accounts (connections)**. This reference is how to navigate those accounts, decide where attention should go, and run a repeatable monthly/quarterly review — within our toolset, and honest about where the surface ends.

## When this applies

Use this whenever the user can reach more than one account. Two things can multiply accounts:

- **Multiple linked Google accounts.** A user can connect several Google accounts (dashboard → *Add Google account*). `list_accounts` returns the **union** of everything reachable across all of them, each row tagged with the connection/email it came through.
- **Manager (MCC) nesting.** Accounts under a manager show up too, with a `manager` flag and currency; the manager's own children are included automatically.

## Navigating accounts

```
1. list_accounts                       → every reachable account across all linked
                                         Google connections + MCC children, each
                                         tagged with its connection/email + currency
2. run_gaql(customerId=<account-id>)   → read any of them — just the customerId
```

Routing is **automatic**: pass only `customerId`. The server selects the Google connection that actually serves that account (token isolation across linked accounts) and, when the account is nested under a manager, sets the manager login internally. There is **no `loginCustomerId` parameter** — manager resolution happens server-side via `customer_client`. Guardrails, budgets, and currencies remain **per account**: there is no cross-account bulk write in our surface, so a change to ten accounts is ten confirmed operations, each read back on its own account (`references/campaign-structure.md` honest-limits discipline applies per account).

**Scale limits (for context, not something we administer):** a manager account can be linked to up to 85,000 non-manager accounts, with the *active* limit tiered by trailing-12-month spend — 50 accounts under $10,000 USD/mo, 2,500 up to $500,000, and 85,000 above (per Google Ads Help, "About account limits for manager accounts," accessed June 2026). An individual account can be directly managed by at most 5 manager accounts (per Google Ads Help, accessed June 2026). Linking and access changes are off-platform — done in the Google Ads UI, not through our tools.

## Account prioritization — where attention goes

You can't review every account equally every day. Rank by **leverage**, not by size alone. Pull a one-row-per-account snapshot to triage:

```
SELECT customer.descriptive_name, metrics.cost_micros, metrics.conversions,
       metrics.cost_per_conversion, metrics.search_budget_lost_impression_share
FROM customer
WHERE segments.date DURING LAST_7_DAYS
```

(Run per account by its `customerId`; correlate the results yourself — there's no single call that returns sibling accounts together.)

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

- **Wrong account id** — routing keys entirely off `customerId`; pass the wrong one and you read/write the wrong account. Confirm the `customerId` (and, from `list_accounts`, which connection/email serves it) before any write.
- **Mixed currencies** — accounts under one MCC, or across different linked connections, can differ; never sum `cost_micros` across accounts without normalizing, and always divide micros by 1,000,000 in client output.
- **Treating the MCC as a bulk tool** — it isn't, in our surface. Per-account confirmation and read-back still apply to each change.
- **One report for many accounts** — flatters or hides per-account truth; one report = one account, one period.

## Sources

- Google Ads Help, "Manager Accounts (MCC): About account limits for manager accounts" (85,000 linked cap; active tiers 50 / 2,500 / 85,000 by trailing-12-month spend) — accessed June 2026: https://support.google.com/google-ads/answer/7526520
- Google Ads Help, manager-account linking limits (an account directly managed by at most 5 manager accounts) — accessed June 2026: https://support.google.com/google-ads/answer/7456530
