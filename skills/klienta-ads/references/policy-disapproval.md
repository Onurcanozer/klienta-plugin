# Reference — Policy & Disapproval Resolution

A disapproved ad serves nothing — it's silent wasted potential, and on a new campaign it can look like "no traffic" when the real cause is a policy block. This reference is how to detect policy problems, fix the ones we can fix with our tools, and be honest about the parts that are off-platform. Detection is `run_gaql`; the fix path is "write a compliant replacement," not "edit in place" — because our surface has no ad-text edit and no appeal/exemption tool. All queries validated against Google Ads API **v21**.

---

## When to use

- A new ad or campaign isn't serving and you need to rule policy in or out before chasing budget/bid/QS.
- An RSA or asset shows a non-serving or restricted status and you need to know *why* and *what to do*.
- Periodic hygiene: catch disapprovals before they quietly drag a campaign.

---

## Decision framework

**1. Detect first — read the policy status, don't guess.** Every ad carries a `policy_summary`. Pull `approval_status` and the `policy_topic_entries` (the specific policy topics flagged). The approval statuses you'll see (PolicyApprovalStatus enum, Google Ads API):

- **DISAPPROVED** — won't serve at all. Highest priority; it's pure dead weight.
- **APPROVED_LIMITED** — serves, but restricted (the UI's *Eligible (limited)*). Limits can be by location, device, age, or restricted-category eligibility (Google Ads Help).
- **APPROVED** — serves with no restriction.
- **AREA_OF_INTEREST_ONLY** — limited serving tied to area-of-interest only.

**2. Classify the violation, then choose the fix.** Read the `policy_topic_entries` topic and type. Broadly:
- **Editorial / content** (capitalization, punctuation, claims, prohibited phrasing) → fixable by us: write a compliant replacement ad.
- **Trademark / brand** in ad text → rewrite to remove the infringing term.
- **Landing-page / destination** problems (broken, mismatched, disallowed content) → **off-platform**: the page must change; no ad rewrite fixes a destination policy. Flag it for the user (mirrors the landing-page limit in `references/quality-score.md`).
- **Restricted categories** (healthcare, gambling, financial, etc.) → often need certification/eligibility the advertiser must obtain off-platform; we can't grant it.

**3. Fix the way our toolset allows — replace, don't edit.** We have no "edit ad text" tool. So the compliant fix for a disapproved RSA is: **create a new, compliant `create_responsive_search_ad`** in the same ad group, verify it's approved, then **pause the offending one** with `update_ad_status` (prefer pausing over removing — removals aren't undoable). Same pattern for a disapproved asset: create a compliant replacement, unlink/pause the bad one. A newly created ad enters review automatically; there's no separate "resubmit" call in our surface.

**4. Appeals and exemptions are off-platform — say so.** We have **no appeal tool and no policy-exemption tool.** If the disapproval is wrong (a false positive) or needs an exemption, that's a Google Ads UI / Policy Manager action the user takes. Be honest about the limits Google places on it: an ad is limited to **3 appeals**, and you should **wait at least 24 hours between appeals** for the same ads/campaigns or they may be marked duplicates (Google Ads Policy Help). Don't imply we can appeal for them — point them to the UI and, for genuine errors, to Google support.

---

## GAQL / tool examples

**Find everything not fully approved:**

```
SELECT ad_group_ad.ad.id,
       ad_group.name,
       ad_group_ad.policy_summary.approval_status,
       ad_group_ad.policy_summary.review_status,
       ad_group_ad.policy_summary.policy_topic_entries
FROM ad_group_ad
WHERE ad_group_ad.policy_summary.approval_status != 'APPROVED'
```

**Decision it supports:** the full list of ads blocked or restricted, with the exact policy topics. Triage DISAPPROVED first (serving nothing), then APPROVED_LIMITED. The `policy_topic_entries` tell you whether the fix is a rewrite (us) or a landing-page/eligibility change (the user).

**Narrow to outright disapprovals on a campaign:**

```
SELECT ad_group_ad.ad.id, ad_group_ad.policy_summary.policy_topic_entries
FROM ad_group_ad
WHERE campaign.id = <CAMPAIGN_ID>
  AND ad_group_ad.policy_summary.approval_status = 'DISAPPROVED'
```

**Fix path (our tools):**

```
create_responsive_search_ad { adGroupId: "<AD_GROUP_ID>", headlines: [...compliant...], descriptions: [...], finalUrl: "..." }
# verify the new ad is APPROVED (re-run the query above), then:
update_ad_status { adId: "<OLD_AD_ID>", status: "PAUSED" }
```

**Verify the fix landed:** re-run the first query and confirm the new ad shows `APPROVED` and the old one is paused. Report the before/after to the user.

---

## Pitfalls

- **Reading "no traffic" as a budget/bid problem when it's a disapproval.** Always check `policy_summary` early in a "why isn't my ad showing" investigation — it's a one-query rule-out.
- **Trying to edit ad text in place.** No such tool. Fix = new compliant ad + pause the old. Don't promise an in-place edit.
- **Treating a landing-page or restricted-category disapproval as ad-copy.** No rewrite fixes a destination or eligibility policy — that's off-platform. Misdiagnosing it wastes a rewrite and the ad stays blocked.
- **Implying we can appeal or request exemptions.** We can't. Appeals/exemptions are UI/support actions, capped at 3 appeals per ad with ≥24h between (Google policy). State the limit; hand it back to the user.
- **Removing instead of pausing the bad ad.** Removals aren't undoable; pause while the replacement proves out.

---

## Sources

- Google Ads Help — *Fix a disapproved ad or appeal a policy decision*, accessed 2026-06-11. https://support.google.com/google-ads/answer/9338593 (fix-and-auto-review flow; appeals limited to 3 per ad; wait ≥24h between appeals on the same ads/campaigns).
- Google Ads Help — *Eligible (limited): Definition*, accessed 2026-06-11. https://support.google.com/google-ads/answer/2684542 (limited serving by location, device, age, or restricted-category eligibility).
- Google Ads API — *PolicyApprovalStatus* enum (v21), accessed 2026-06-11. https://developers.google.com/google-ads/api/reference/rpc/v21/PolicyApprovalStatusEnum.PolicyApprovalStatus (DISAPPROVED, APPROVED_LIMITED, APPROVED, AREA_OF_INTEREST_ONLY).
- Google Ads API — *Get all disapproved ads* sample, accessed 2026-06-11. https://developers.google.com/google-ads/api/samples/get-all-disapproved-ads (`ad_group_ad.policy_summary.approval_status` + `policy_topic_entries` query pattern).
- Tool surface: `create_responsive_search_ad`, `update_ad_status` (replace-and-pause fix path; no ad-text edit, no appeal/exemption tool). Related: `references/quality-score.md` (landing-page limits are off-platform).
