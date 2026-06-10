# Reference — Search-Term Mining → Negative-Keyword Discipline

Broad and phrase match keywords show ads on queries you never literally chose. Search-term mining is the recurring hygiene job of reading what people *actually searched*, keeping the good, and blocking the bad with negatives. Done with discipline, it is the single highest-ROI maintenance habit in a Search account.

All queries validated against Google Ads API **v21** via `run_gaql`. This is a read → decide → confirm → write loop using `add_negative_keywords`.

---

## Step 1 — Pull the search terms, sorted by spend

```
SELECT search_term_view.search_term,
       metrics.cost_micros, metrics.conversions, metrics.clicks,
       campaign.name
FROM search_term_view
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.cost_micros DESC
```

**Decision it supports:** which actual queries are consuming budget. Read top-down by spend — that is where the money leaks. Convert `cost_micros` to currency units (÷ 1,000,000).

---

## Step 2 — Classify each term (three buckets)

For each high-spend term, decide which bucket it belongs to:

1. **Converting / relevant** → leave it. If it converts well and isn't already an exact keyword, consider promoting it to its own exact-match keyword later (`add_keywords`, `EXACT`) for tighter control. Not a negative.
2. **Irrelevant** → spend, no conversions, and clearly off-intent (wrong product, "free", "jobs", "diy", competitor-but-not-wanted, wrong geography). **This is a negative candidate.**
3. **Ambiguous** → spend, no conversions yet, but plausibly on-intent. Don't negate yet; it may need more time or a landing-page fix. Flag for the next review, don't act on thin data.

The disciplined rule: **a term becomes a negative only when it has enough spend to matter AND is clearly off-intent.** Negating on one click, or negating something merely unprofitable-but-relevant, throws away reach. Spend with zero conversions is necessary but not sufficient — relevance is the deciding test.

A focused query for the strongest negative candidates:

```
SELECT search_term_view.search_term, metrics.cost_micros,
       metrics.clicks, metrics.conversions, campaign.name
FROM search_term_view
WHERE segments.date DURING LAST_30_DAYS
  AND metrics.conversions = 0
  AND metrics.clicks > 3
ORDER BY metrics.cost_micros DESC
```

**Decision it supports:** terms that have taken real clicks/spend and returned nothing — the shortlist you actually review one by one for relevance.

---

## Step 3 — Choose match type for the negative

- **Negative EXACT** — block exactly this query, nothing else. Safest; use when only this specific phrasing is bad.
- **Negative PHRASE** — block any query containing this sequence. Use for a recurring junk token (e.g. `free`) where you're confident every variant is unwanted.
- Prefer the **narrowest** negative that solves the problem. An over-broad negative phrase can silently suppress good traffic — that's the mirror image of the waste you're trying to stop.

---

## Step 4 — Confirm, then write, then verify

1. **Show the user the proposed negatives** as a list: term, match type, campaign, and the spend/clicks that justify each. Get approval — this is a write.
2. **Write** with `add_negative_keywords` (campaign-level), passing `{ text, matchType }` per term.
3. **Verify** with a read-back:

```
SELECT campaign_criterion.keyword.text,
       campaign_criterion.keyword.match_type,
       campaign_criterion.negative
FROM campaign_criterion
WHERE campaign_criterion.negative = true
  AND campaign.id = <CAMPAIGN_ID>
```

**Decision it supports:** confirms the negatives actually landed on the campaign with the intended match types. Report the before/after to the user.

---

## Cadence

Search-term mining is not one-and-done. New queries appear constantly. Recommend a recurring review (e.g. weekly for active spenders) and keep each pass small and reversible — a short, well-justified negative list every week beats a giant speculative block once a quarter.
