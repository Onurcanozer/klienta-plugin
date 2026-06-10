# Reference — Benchmark Calibration (No Invented Numbers)

People ask "is a $40 CPA good?" The honest answer is: it depends on the margin and the account, and there is no universal benchmark we can quote. This reference is the method for judging "good" without inventing industry numbers — because we have no sourced benchmark dataset, and quoting one would be making up data.

## Why no benchmark table here

Industry CPC/CTR/CPA/CVR figures vary enormously by vertical, geography, season, device, and time, and any number not tied to a citable source is a guess dressed up as fact. Presenting invented benchmarks would make recommendations *feel* grounded while being baseless. So this skill calibrates against two things we can actually defend: **the account's own history** and **the user's real economics**.

## Calibration source 1 — the account's own history

The most reliable yardstick is the account compared to itself.

- **Self-baseline.** Pull the metric over a meaningful trailing window and look at its distribution, not a single number. A keyword's QS, a campaign's CPA, an ad group's CTR — judge each against the *account's own median* and its own prior periods.
  ```
  SELECT campaign.name, metrics.cost_per_conversion, metrics.conversions, metrics.ctr
  FROM campaign
  WHERE segments.date DURING LAST_30_DAYS
  ```
  Then compare to the prior 30 days (dated `run_gaql`, `references/change-impact.md`). "Worse than this account usually does" is a defensible finding; "worse than the industry" is not.
- **Within-account peers.** Compare like to like inside the account — campaigns of the same type, ad groups in the same theme. An outlier relative to its peers is a real signal.
- **Trend over level.** Direction over time (improving/worsening) is more trustworthy than a single absolute number, and it's immune to the "what's normal?" problem.

## Calibration source 2 — the user's economics

Absolute "good" comes from the business, not a table:

- **Break-even CPA / ROAS** from the user's margin (`references/ppc-math.md`) is the real ceiling. A $40 CPA is good if break-even CPA is $80 and bad if it's $25 — same number, opposite verdict.
- **Get the inputs from the user** (margin, AOV, LTV). If they're unavailable, say the profitability verdict is unavailable rather than substituting an assumed benchmark.

## How to answer "is this good?"

1. Compare to the account's own history and within-account peers (trend + distribution).
2. Compare to break-even from the user's margin.
3. State the verdict *relative to those*, and name the basis: "above this account's 90-day median CPA, and above your break-even CPA of X — so it's losing money," not "above the industry average."
4. If you genuinely lack both an account history and the user's economics (e.g. a brand-new account), say so plainly and gather more data before judging — don't fill the gap with a made-up number.

> The discipline in one line: **calibrate against the account's own data and the user's margin; never quote an industry benchmark we can't source.**
