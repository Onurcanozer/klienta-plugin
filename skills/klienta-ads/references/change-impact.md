# Reference — Change-Impact Review Protocol

Making a change is half the job; knowing whether it *worked* is the other half. This protocol turns the audit log and `run_gaql` into a disciplined before/after measurement so you can say, with evidence, whether an intervention helped, hurt, or did nothing — and reverse it if it hurt.

Read-only throughout, except the optional `undo_change` at the end. Validated against v21.

## What you have to work with

- **`get_changes`** — this server's own audit log: every mutating tool call, when it ran, whether it succeeded, the resources it touched, and the `auditId` needed to undo it. Plus Google's `change_event` for context. This is your record of *what changed and when*.
- **`run_gaql` with date ranges** — pull the same metrics for the window *before* the change and the window *after* it.

## The protocol

1. **Identify the intervention and its timestamp.** From `get_changes`, find the change you're evaluating (e.g. a budget raise on campaign X on a given date) and note the `auditId` and the date it landed.
2. **Define matched windows.** Compare equal-length periods immediately before and after the change. Use a **7-day** window for fast-moving accounts and **14-day** for low-volume ones (more data, less noise). Equal length and adjacency matter — comparing a 7-day after-window to a 30-day before-window is meaningless.
   ```
   SELECT campaign.name, metrics.cost_micros, metrics.conversions,
          metrics.conversions_value, metrics.ctr, metrics.cost_per_conversion
   FROM campaign
   WHERE campaign.id = <CAMPAIGN_ID>
     AND segments.date BETWEEN '<before_start>' AND '<before_end>'
   ```
   Then the same query for `'<after_start>' AND '<after_end>'`. (Date-range `BETWEEN` is validated; pass explicit ISO dates derived from the change date.)
3. **Compare the metrics that the change was supposed to move.** A budget raise should move impressions/clicks/cost and ideally conversions; a negative-keyword addition should reduce wasted spend without dropping conversions; a bid change should move CPC and position. Look at the metric the change *targeted*, plus a guardrail metric (CPA/ROAS) to catch a "more volume but worse economics" outcome.

## Confounders — the part that separates evidence from coincidence

Before declaring a verdict, rule out the obvious alternative explanations. An after-window can differ from a before-window for reasons that have nothing to do with your change:

- **Seasonality / day-of-week / paydays** — a week-over-week jump may just be a busier week. Prefer equal weekday coverage; widen to 14 days if a holiday or spike sits inside a window.
- **Other changes in the same window** — check `get_changes` for *anything else* that landed in the after-window. If two changes overlap, you cannot attribute the effect to one of them.
- **Volume too low to conclude** — a few conversions either way is noise. If the numbers are thin, the honest verdict is "inconclusive — need more data," not a guess.
- **External factors you can't see** — promotions, news, competitor moves. Flag uncertainty rather than overclaiming.

## Verdict classification

Conclude with one of four labels, stated plainly with the before/after numbers:

- **Improved** — the targeted metric moved the right way *and* the guardrail metric held or improved, with enough volume and no confounder. Keep the change.
- **Worsened** — the targeted metric moved the wrong way, or volume rose while CPA/ROAS broke past break-even (`references/ppc-math.md`). Recommend reverting with `undo_change(auditId)` (or pausing/adjusting if it's not undoable).
- **Neutral** — no meaningful movement beyond noise. The change neither helped nor hurt; decide whether it's worth keeping for other reasons.
- **Inconclusive** — too little data or a confounder you can't isolate. Don't force a call; extend the window or wait.

## Reversal

If the verdict is "Worsened" and the change came from a `create_*`/`update_*` tool, `undo_change(auditId)` reverses it (creates → removed, updates → restored from the pre-write snapshot). `remove_*` changes cannot be undone — which is exactly why removals get extra confirmation up front. Always read the entity back with `run_gaql` after an undo to confirm the reversal landed.

> This protocol is the measurement discipline behind the operating contract's "verify after every write" rule, extended over days instead of seconds: the immediate read-back proves the change *applied*; the change-impact review proves it *worked*.
