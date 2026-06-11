# Reference — Shared Negative-Keyword Lists

A shared negative-keyword list (a *shared set* in the API) is one curated block list you maintain once and attach to many campaigns. It's the account-wide companion to the per-campaign negatives in `references/search-term-mining.md`: that file is about *discovering* what to block; this one is about *where* the block lives so you don't re-apply the same junk by hand to every campaign.

Five tools cover the full lifecycle: `create_negative_keyword_list`, `add_keywords_to_negative_list`, `remove_keyword_from_negative_list`, `link_negative_list_to_campaign`, `unlink_negative_list_from_campaign`. Reads use `run_gaql`. All against Google Ads API **v21**.

---

## When to use

Use a shared list when the **same** exclusion should apply across **several** campaigns and you want to manage it in one place:

- **Account-wide junk** that's junk everywhere — recurring off-intent tokens you'd otherwise add campaign-by-campaign.
- **A reusable, business-calibrated block library** — the negatives this account has *confirmed* over time (via `get_changes` audit history), promoted from one-off campaign negatives into a durable list that every new campaign inherits on launch.

Stay with **campaign-level** `add_negative_keywords` (per `references/search-term-mining.md`) when the exclusion is specific to one campaign's theme — e.g. a term that's junk for the prospecting campaign but is the *point* of another. Don't push campaign-specific negatives into a shared list; you'll silently suppress good traffic elsewhere.

The line: **shared = universal across campaigns; campaign-level = local to one theme.** A keyword can be negative in both places at once; they're additive.

---

## Decision framework

1. **Is this junk everywhere, or only here?** Everywhere → shared list. Only here → campaign-level negative. This single question decides placement.
2. **Does a fitting list already exist?** Reuse it (`run_gaql` on `shared_set`) before creating another — lists are capped (see Pitfalls) and a sprawl of overlapping lists is harder to reason about than a few well-named ones.
3. **Match type, narrowest that works.** Same discipline as any negative (`references/search-term-mining.md`): negative EXACT blocks one phrasing; negative PHRASE blocks any query containing the token. A shared PHRASE negative is high-leverage *and* high-blast-radius — it suppresses that token across every linked campaign at once. Confirm the token is off-intent across all of them before adding it shared.
4. **Confirm before write, prefer reversible.** Adding/linking is reversible (unlink, remove the criterion); explicit removals are permanent. Show the user the proposed list + the campaigns it'll touch before linking.

### Universal exclusion themes (conceptual, calibrate to the business)

Common account-wide negative *themes* — useful as starting categories, not a fixed list to paste blindly:

- **Job-seekers** — "jobs", "careers", "hiring", "salary" (unless the advertiser *is* recruiting).
- **Free / cheap intent** — "free", "cheap", "torrent", "crack" — clicks that rarely convert for paid offerings.
- **DIY / how-to** — "how to", "tutorial", "diy" — research intent, not buying intent, for a done-for-you service.
- **Competitor brands** — only when the advertiser doesn't want to bid on rivals (sometimes they do — confirm).

These are directional. Whether each fits depends on the actual business — a DIY-supplies retailer *wants* "diy". Build the real list from the account's own confirmed negatives (`references/search-term-mining.md` n-gram pass), not an assumed block list. There's no published "standard" count of negatives to apply — don't invent one.

---

## GAQL / tool examples

**Create, fill, link** (one list across many campaigns):

```
create_negative_keyword_list { name: "Account-wide exclusions" }   → sharedSetId
add_keywords_to_negative_list {
  sharedSetId,
  keywords: [ {text:"free", matchType:"PHRASE"}, {text:"jobs", matchType:"PHRASE"} ]   // max 100/call
}
link_negative_list_to_campaign { campaignId: <A>, sharedSetId }
link_negative_list_to_campaign { campaignId: <B>, sharedSetId }
```

**Read back — what's on the list:**

```
SELECT shared_criterion.keyword.text,
       shared_criterion.keyword.match_type,
       shared_criterion.criterion_id
FROM shared_criterion
WHERE shared_set.id = <SHARED_SET_ID>
```

**Read back — which campaigns a list is linked to:**

```
SELECT campaign.name, campaign_shared_set.shared_set, campaign_shared_set.status
FROM campaign_shared_set
WHERE shared_set.id = <SHARED_SET_ID>
```

**Maintenance.** `remove_keyword_from_negative_list { sharedSetId, criterionId }` (get `criterionId` from the `shared_criterion` read above) drops one keyword from the list — affecting every linked campaign at once. `unlink_negative_list_from_campaign { campaignId, sharedSetId }` detaches the list from one campaign without deleting the list. Both removals/unlinks are permanent (not undoable) — confirm explicitly.

---

## Pitfalls

- **Over-block by leverage.** A shared PHRASE negative fires across *every* linked campaign. The same token that's pure junk for one campaign may sit inside a converting query for another. Before adding shared, verify it's off-intent across all linked campaigns (check converting terms containing it, as in `references/search-term-mining.md`); if mixed, keep it as a campaign-level negative instead.
- **Account limits.** Google caps a negative keyword list at **5,000 keywords**, and an account at **20 negative keyword lists** (Google Ads Help, accessed 2026-06-11). Don't architect around hundreds of tiny lists — consolidate into a few well-named ones.
- **`add_keywords_to_negative_list` is 100 keywords per call**, with partial-failure reporting (per-item errors come back; the call doesn't all-or-nothing). For larger sets, batch.
- **Unlink ≠ delete.** `unlink_negative_list_from_campaign` only detaches the list from one campaign; the list and its keywords persist for other campaigns. Removing a *keyword* needs `remove_keyword_from_negative_list`.
- **Don't duplicate effort.** A keyword in both a linked shared list and a campaign-level negative is harmless but redundant — when promoting confirmed campaign negatives into a shared list, you can leave the originals or clean them; just don't assume the shared list *replaces* campaign-level negatives automatically.

---

## Sources

- Google Ads Help — *About negative keyword lists*, accessed 2026-06-11. https://support.google.com/google-ads/answer/2453983 (up to 5,000 negative keywords per list; up to 20 lists per account; apply one list to multiple campaigns).
- Google Ads Help — *About your Google Ads account limits*, accessed 2026-06-11. https://support.google.com/google-ads/answer/6372658 (5,000 keywords per negative keyword list; 20 lists per manager/child account).
- Tool surface: `src/tools/shared.ts` — `create_negative_keyword_list`, `add_keywords_to_negative_list` (max 100/call, partial failure), `remove_keyword_from_negative_list`, `link_negative_list_to_campaign`, `unlink_negative_list_from_campaign`; GAQL resources `shared_set`, `shared_criterion`, `campaign_shared_set`.
