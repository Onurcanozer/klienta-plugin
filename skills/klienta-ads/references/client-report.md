# Reference — Client-Facing Report Template

Audit findings and operator notes are full of GAQL, resource names, and micros. A client doesn't want that — they want to know how their account is doing, what you changed, and what you're asking them to approve. This reference turns the internal audit/optimization output into a short, plain-language update the account owner can actually read and act on.

Read-only: this is a communication format, not a tool. It consumes what the `klienta-audit` scans and `get_changes` already produced.

## Principles

- **Plain language, no jargon.** "We're losing about a third of possible clicks because the budget runs out by midday" — not "search_budget_lost_impression_share = 0.32." Translate every metric into a business consequence.
- **Money in currency, never micros.** Always divide `*_micros` by 1,000,000 and show the account's currency.
- **Lead with state, then changes, then asks.** The reader wants: How are we doing? What did you do? What do you need from me?
- **Separate done-and-verified from proposed-and-waiting.** Never imply a change is live unless you confirmed it with a read-back. Anything that needs the user's go-ahead goes in its own clearly-labeled list — nothing was executed there.
- **Every claim traces to data.** Behind each plain sentence is a `run_gaql` result; keep the numbers honest and the window explicit ("last 30 days").

## Template

```
# Account Update — <account name>
Period: <window, e.g. last 30 days> · Currency: <CCC>

## Where things stand
<2–4 plain sentences: overall health, total spend, and whether it's
producing results. If conversion tracking is off, say so first — results
are spend-only until it's set up.>

## What I changed (done & verified)
- <plain description> — <why> — <verified result, e.g. "confirmed the
  campaign is now paused">
  (only list changes you actually executed AND read back)

## Waiting for your approval
1. <proposed action in plain terms> — <expected effect> — <why it's worth it>
2. ...
(Nothing in this list has been done. Each needs your OK before I proceed,
and each is reversible except permanent removals, which I'll always confirm
with you first.)

## Watch items / next review
<anything to monitor, and when you'll look again>
```

## Mapping audit output → report sections

- **Audit score + tracking status** → "Where things stand." A `NOT_CONVERSION_TRACKED` finding is the first sentence, because it caveats everything else.
- **Confirmed changes from `get_changes`** (successful `update_*`/`create_*` you ran and verified) → "What I changed."
- **Audit findings' recommended actions** → "Waiting for your approval." These are proposals — the audit skill never executes — so they belong in the approval list, framed as the smallest reversible step.
- **Watch items** (e.g. a budget-constrained campaign you're monitoring, a change under change-impact review) → "Watch items."

## Tone discipline

- Don't oversell. If something is uncertain or data is thin, say so — "too early to tell" reads as competence, not weakness.
- Don't bury the ask. The approval list is the point of the report; make it the easiest part to act on.
- One report = one account, one period. Don't blend accounts or stretch windows to flatter the numbers.
