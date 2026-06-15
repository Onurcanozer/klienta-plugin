# Reference — Policy Compliance (write to pass review, not to fix rejections)

Most disapprovals are preventable. The cheapest policy fix is the one you make *before* the ad is created — a rejected ad serves nothing, and a disapproval on a sensitive vertical can escalate to account-level suspension. This reference is the compliance-first half of policy work: the policy taxonomy you write against, the high-risk vertical rules to internalize, and the pre-flight check (`precheck_ad_policy`) that catches violations before they cost you. For *detecting and recovering* already-disapproved ads, see `references/policy-disapproval.md` — this file is its upstream complement.

> **Fact-check status (verified 2026-06-15):** the policy claims below were checked against the live Google Ads policy pages on 2026-06-15 (sources at the foot of this file). Where the prior draft was wrong or outdated it has been corrected in place. Google revises these pages often, so re-verify before quoting any specific rule as authoritative; the cited URL + access date marks the last check. The *tool behavior* (`precheck_ad_policy`, `get_disapproved_ads`, `diagnose_campaign`) is from our own surface and is authoritative without web verification.

---

## When to use

- **Before** creating any RSA, especially in a sensitive vertical — run `precheck_ad_policy` first.
- When writing ad copy in government-services, third-party-services, financial, or health spaces (the high-risk verticals below).
- When an account has a history of disapprovals and you want to stop them at the source rather than firefight them.
- Pair with `references/policy-disapproval.md` once an ad is *already* disapproved.

---

## 1. The policy taxonomy (what gets ads rejected)

Google organizes advertising policy into **four top-level families** (verified 2026-06-15, [Google Ads policies overview](https://support.google.com/adspolicy/answer/6008942)). Know which family a violation falls in, because the *fixability* differs sharply by family. **Note the correction from earlier drafts:** "Misrepresentation" and "Trademarks" are **not** top-level families — they are policies *within* the families below.

**A. Prohibited content** — things that may never be advertised at all: counterfeit goods, dangerous products/services, content enabling dishonest behavior, and inappropriate/offensive content. No rewrite saves a prohibited-content ad — the *offer itself* is disallowed. (Source: overview, 2026-06-15.)

**B. Prohibited practices** — things you can't *do* as an advertiser. **This is where Misrepresentation lives** (deceiving users about who you are, what you offer, or the terms — impersonation, dishonest pricing/hidden fees, unreliable claims), alongside abuse of the ad network and data. **This is the family our high-risk vertical (section 2) lives in, and the one most likely to trigger account-level action rather than a single-ad rejection.** (Source: overview + [Misrepresentation](https://support.google.com/adspolicy/answer/6020955), 2026-06-15.)

**C. Restricted content and features** — legal to advertise but only under conditions: gating by certification, geography, or audience. Includes adult content, alcohol, gambling, healthcare/medicines, financial services, political content — **and Trademarks** (restricted on trademark-owner complaint). The fix is usually **off-platform eligibility/certification the advertiser must obtain**, not ad copy. (Source: overview, 2026-06-15.)

**D. Editorial and technical** — quality and mechanics: capitalization/punctuation/symbols not used for their intended purpose, gimmicky repetition, vague or unidentified ad text, and technical requirements on the destination (must load, must work on the targeted device, no mismatch between ad and landing page). **This family is the most fixable by us** — it's almost always a copy or final-URL rewrite. (Source: [Editorial](https://support.google.com/adspolicy/answer/6021546), 2026-06-15.)

**Triage rule of thumb:** Editorial (D), the everyday Misrepresentation copy fixes, and a trademark term in copy are often fixable by rewriting the ad with our tools. Prohibited content (A) is never fixable (drop the offer). Restricted content + features (C) — incl. the government-documents vertical (section 2) — usually needs the *advertiser* to obtain eligibility/**certification** off-platform. Misdiagnosing the family wastes a rewrite — classify before you fix.

---

## 2. High-risk vertical deep-dive: government documents, official services & third-party services

This is the vertical most likely to produce disapprovals **and** account suspensions, so it gets the most space. `precheck_ad_policy` has dedicated rules for exactly this vertical (see section 4) — but understand *why* so the copy is right the first time.

> **CRITICAL CORRECTION (verified 2026-06-15).** Earlier drafts framed this vertical as "write around it" — add a disclaimer, disclose fees, avoid 'official + free,' and you can run. **That is no longer how the gate works.** Per the current [Government documents and services](https://support.google.com/adspolicy/answer/13156083) policy (a dedicated policy under *Restricted content and features*, not generic Misrepresentation): **"Only certified governments and authorized providers may run ads that promote direct acquisition of specific government documents and services."** Advertisers **must be certified by Google** (apply for a certification option + complete Google's advertiser-verification program). An **authorized provider** must have a domain that is **"linked from an official government website and explicitly referenced as authorized"** to provide that document/service. For certified non-government advertisers, **Google automatically generates a "Not a government website" disclosure** on the Search ad — the advertiser does not hand-write it. **Bottom line:** a private assistance business that is *not* certified/authorized generally **cannot** run ads promoting direct acquisition of these documents at all — copy tweaks do not unlock it. The copy guidance below still matters for staying out of Misrepresentation generally, but certification is the real gate.

The broader Misrepresentation pattern Google targets ([Misrepresentation](https://support.google.com/adspolicy/answer/6020955), 2026-06-15): a business advertising help with **government documents or official services** (passports, visas, ESTA/eTA, licenses, permits, tax filing, business registration, benefit applications, certificate/record requests) in a way that makes users think they're dealing with the government or an official channel — "you can't make it seem like you're supported by another brand, organization or government entity when you're not" — or that obscures fees. Verified policy name: **"Government documents and services"** (dedicated page) + **Misrepresentation** (incl. **"Dishonest pricing practices"** for fee disclosure).

### The four failure modes (and how to write around each)

**(a) Official impersonation.** Copy/branding implies you *are* the government agency or an official partner. Google's rule: don't "make it seem like you're supported by another brand, organization or government entity when you're not," and don't obscure or omit "material information about your identity, affiliations, or qualifications" ([Misrepresentation](https://support.google.com/adspolicy/answer/6020955), 2026-06-15). **Avoid:** government seals/crests, ".gov"-style framing, phrasing like "official passport service," agency names used as if you are them. **Write instead:** position clearly as a private *assistance/agency* service.

**(b) "Official" + "free" deception.** Claiming the service is "official" and/or "free" when you are not official and/or charge a service fee. *Verification note:* Google does **not** publish a dedicated, separately-named "official + free" rule — this is an **example** of the general Misrepresentation prohibitions on false affiliation (a) and dishonest pricing (c), not a standalone policy. Treat the combined "Official & Free" claim as a strong heuristic flag, not a quotable named rule. **Avoid:** "Get your [document] — Official & Free." **Write instead:** state plainly that this is a paid third-party service.

**(c) Fee transparency.** Under Misrepresentation → **"Dishonest pricing practices"** you must "clearly and conspicuously disclose the payment model or full expense that a user will bear before and after purchase" ([Misrepresentation](https://support.google.com/adspolicy/answer/6020955), 2026-06-15). *Verification note:* the policy does **not** explicitly require itemizing your service fee *separately from* the government's own fee — that separation is our best-practice recommendation, not verbatim policy. Hidden or surprise fees are the violation. **Avoid:** burying the fee, implying your fee is the government's fee. **Write instead:** disclose the full cost the user will bear; where space allows, note an official lower-cost/free route may exist.

**(d) Non-affiliation disclosure.** *Corrected (verified 2026-06-15):* for the **certified** government-documents categories, **Google automatically generates a "Not a government website" disclosure** on the Search ad ([Government documents and services](https://support.google.com/adspolicy/answer/13156083)) — the advertiser does not author it, and it is not a free-form copy line you add. Outside that certified path, Google's general guidance is softer than a hard mandate: if you reference another brand/organization without official affiliation, **"consider a disclaimer on your website and in your ads"** ([Misrepresentation](https://support.google.com/adspolicy/answer/6020955)). **Write instead:** where you are referencing an official body without affiliation, add a clear "not affiliated with / not endorsed by [agency]" statement on the landing page (and in copy where space allows) — but understand the binding gate for direct-acquisition ads is certification, not a self-authored disclaimer.

### Third-party technical support & related "official-sounding" services

A parallel high-risk space: **third-party tech support, account-recovery, and similar services.** *Corrected (verified 2026-06-15):* this is **not** a "verify-and-you-can-run" vertical like government docs. Under [Other restricted businesses → third-party consumer technical support](https://support.google.com/adspolicy/answer/13527027), **"technical support for consumer technology products and online services offered by third-party providers is not allowed"** — including remote/online/offline troubleshooting, **account and password support, account recovery, password resets, software setup, data recovery, virus removal, installations and maintenance**. There is **no general advertiser-verification program that unlocks consumer third-party tech support** (unlike the government-documents certification). To comply, **remove any mention of repair/installation/technical-support services from the ad and landing page**, and make the "About Us"/business description state the services you actually offer. The anti-impersonation logic still applies, but the operative rule here is *prohibition*, not disclosure.

### Vertical checklist (apply before writing copy)

1. **Direct-acquisition of a government document/service?** If yes, the binding gate is **Google certification + authorized-provider status** (domain linked from an official government site). No certification → don't run; copy fixes won't unlock it.
2. Are we clearly a **private/independent** service (no implied official status)?
3. Did we avoid the deceptive **"official" + "free"** combination?
4. Is the **full cost the user bears clearly disclosed** (Dishonest-pricing rule)? Best practice: separate your service fee from any government fee.
5. For **tech-support-style offers**: third-party consumer tech support is **prohibited** — remove repair/installation/support claims entirely (no verification path opens it).

If any answer is "no," fix it before creating the ad — then confirm with `precheck_ad_policy`.

---

## 3. Disapproval-recovery workflow (our tools, by name)

When an ad is *already* disapproved, this is the loop. Each step maps to a real tool on our surface — no appeal/exemption tool exists here (appeals are a Google UI action; see `references/policy-disapproval.md`).

**Step 1 — Detect: `get_disapproved_ads`.** Returns the ads Google DISAPPROVED (they stop serving) with the policy topics behind each rejection. This is the entry point — it tells you *which* ads and *which* policy topic, so you can classify the family (section 1) and choose the fix path. (For a one-call "what do I need to touch today" sweep that includes disapprovals among other anomalies, `get_alerts` surfaces them too.)

**Step 2 — Diagnose context: `diagnose_campaign`.** When a disapproval is *why a campaign isn't serving*, `diagnose_campaign` stitches campaign primary status + reasons, ad/ad-group policy approval, budget, and bidding state into one plain-English diagnosis with a suggested next action. Use it to confirm the disapproval is the real blocker (not budget/bid) before spending a rewrite. This is the "is it really policy?" rule-out.

**Step 3 — Fix by rewrite (we have no edit-in-place for ad text):**
- Classify the policy family from the topic in step 1 (section 1: Prohibited content / Prohibited practices+Misrepresentation / Restricted content+features incl. Trademarks / Editorial).
- **Editorial, copy-level Misrepresentation, or a trademark term in copy** → write a compliant replacement. `update_responsive_search_ad` is the clean path: it SWAPs (creates a new ENABLED ad with the full corrected creative and pauses the old one — RSAs are immutable). Or `create_responsive_search_ad` for a fresh ad + `update_ad_status` to PAUSE the offending one. **Run `precheck_ad_policy` on the new copy first (section 4).**
- **Landing-page / destination** → fix the page (off-platform) or repoint with `update_ad_final_url`. No copy rewrite fixes a destination policy.
- **Restricted-content eligibility / certification** (e.g. government-documents, restricted verticals) → off-platform: the advertiser must obtain certification/authorization. We can't grant it; say so.

**Step 4 — Resubmit = automatic.** There is **no separate "resubmit" call** in our surface. A newly created/swapped ad enters review automatically. So the loop is: rewrite → the new ad auto-enters review → verify.

**Step 5 — Verify: `get_disapproved_ads` again (or `run_gaql` on `policy_summary`).** Confirm the new ad is approved and the old one is paused. Report before/after to the user. (GAQL verification query lives in `references/policy-disapproval.md`.)

**Never imply we can appeal.** False-positive disapprovals and exemption requests are Google-UI actions the user takes — hand those back. See `references/policy-disapproval.md` for the appeal limits.

---

## 4. `precheck_ad_policy` playbook (run BEFORE you create)

`precheck_ad_policy` is the pre-flight. **Run it on every RSA before `create_responsive_search_ad`, and on the new copy before any disapproval-recovery rewrite.** It costs one operation and can save a disapproval and a re-do.

**What it does (our surface — authoritative):**
- **Google's native policy validation** — it builds the *real* create operation and calls Google with `validateOnly`, so **nothing is created**; you get Google's own verdict.
- **Government-docs-vertical rules** — the section-2 checks: official-impersonation, deceptive "official + free," and missing non-affiliation disclaimer.
- **An SSRF-guarded landing-page check** — pulls the destination safely to catch obvious destination problems.

**What it returns:** a **pass/fail per check, with fixes** — so a failure tells you *which* rule tripped and *how* to correct it. Read-only.

**When to run it:**
1. Drafting a new RSA → precheck → fix any fail → then `create_responsive_search_ad`.
2. Recovering a disapproval (section 3, step 3) → precheck the *replacement* copy → then swap.
3. Any government-services / official-services / tech-support vertical copy → always precheck; the vertical rules are exactly what it's built for.

**How to act on results:**
- **All pass** → proceed to create (still under the normal confirm-before-write rule).
- **Native validation fail** → Google itself would reject it; rewrite per the returned fix, don't create.
- **Vertical-rule fail** (impersonation / "official+free" / missing disclaimer) → apply the section-2 fix: declare independence, drop "official+free," add the non-affiliation disclaimer.
- **Landing-page check fail** → the *destination* needs work (off-platform) or repointing; no copy change fixes it.

**Honest limit:** precheck reflects validation at check time and the vertical heuristics we implement — it raises confidence, it is not a guarantee Google never disapproves later (manual review and policy changes still apply). Don't promise "this will definitely be approved"; say "it passes pre-flight."

---

## 5. RSA disapproval-avoidance copy rules

Compliance constraints on the asset pool, layered on top of the craft guidance in `references/rsa-writing.md` (which covers variety, limits, and pinning). These are the lines to *not* write. The editorial rules below were verified against the live [Editorial policy](https://support.google.com/adspolicy/answer/6021546) on 2026-06-15.

**Editorial / technical (the everyday rejections):**
- **No gimmicky capitalization** — Google bars "capitalization that isn't used correctly or for its intended purpose" (don't SHOUT a whole headline or "CamelCaseEveryWord" for emphasis). (Editorial, 2026-06-15.)
- **No gimmicky punctuation/symbols** — "punctuation or symbols that aren't used correctly or for their intended purpose" and "invalid or unsupported characters" are prohibited (avoid "!!!", repeated symbols, or symbols standing in for words). (Editorial, 2026-06-15.)
- **No repetition gimmicks** — "non-standard, gimmicky, or unnecessary repetition of names, words, or phrases" is prohibited ("Cheap cheap cheap"). (Editorial, 2026-06-15.)
- **Functional, specific copy + identify the business** — ads that "don't name the product, service, or entity they are promoting," and vague/non-functional phrasing, get flagged. Make every line a real, true statement. (Editorial, 2026-06-15.)
- **No unsupported claims / invented numbers** — don't write "#1," "guaranteed," or specific stats you can't substantiate (this also echoes the "specifics you can stand behind" rule in `references/rsa-writing.md`).

**Misrepresentation (the dangerous rejections — see section 2):**
- **No implied official status** unless you genuinely are official.
- **Never combine "official" + "free"** deceptively.
- **No hidden-fee framing** — if there's a service fee, the copy/landing page must be honest about it.
- **Include the non-affiliation disclaimer** for government/official-services offers.

**Trademark:**
- **Don't put a competitor's/other party's trademark in headlines or descriptions** unless an exception applies. Verified ([Trademarks](https://support.google.com/adspolicy/answer/6118), 2026-06-15): trademark use in ad text is **restricted on the trademark owner's complaint**; Google will **not** restrict it where the landing page is a genuine **reseller** (sells/clearly facilitates sale of the goods, with pricing/purchase), is **informational** about the trademarked product, or uses the term **descriptively in its ordinary meaning**. Direct competitors and confusing/deceptive use **are** restricted. Note trademarks are **always allowed as keywords** and in the display-URL second-level domain — the restriction is on ad *text*. Fix = remove the infringing term from copy (or qualify under a valid exception).

**Landing-page alignment (technical):**
- **Promise only what the destination delivers** — ad↔landing-page mismatch is a destination policy issue, not just a relevance one. The page must load and work on targeted devices.

**Workflow:** draft the pool → self-check against this list → `precheck_ad_policy` → create. The list catches what you can see; precheck catches what Google sees.

---

## Pitfalls

- **Treating policy as a post-hoc cleanup.** The cheapest fix is pre-flight. Run `precheck_ad_policy` before creating, not after a disapproval.
- **Misclassifying the policy family.** A restricted-category or landing-page disapproval can't be rewritten away. Read the topic from `get_disapproved_ads`, classify (section 1), then choose the fix.
- **Skipping the vertical checks on "official" services.** Government-docs/official-services/tech-support copy is the highest-suspension-risk space — always run the section-2 checklist and precheck.
- **Promising approval.** Precheck raises confidence; it doesn't guarantee Google's manual review. Say "passes pre-flight," not "will be approved."
- **Implying we can appeal or resubmit on demand.** No appeal/exemption tool here; resubmit is automatic on a new ad. Appeals are a Google-UI action (see `references/policy-disapproval.md`).
- **Quoting these rules as stale gospel.** The claims here were verified 2026-06-15 against the cited pages, but Google revises policy often — re-check the cited URL before quoting a specific rule as current. In particular: the government-documents vertical is now a **certification/authorized-provider** regime, and third-party consumer tech support is **prohibited** — not the "add a disclaimer / get verified" framing earlier drafts used.

---

## Sources

- Tool surface (authoritative): `precheck_ad_policy` (validateOnly native check + government-docs vertical rules + SSRF-guarded landing-page check, pass/fail + fixes), `get_disapproved_ads` (disapprovals + policy topics), `diagnose_campaign` (serving/policy diagnosis), `get_alerts` (anomaly sweep incl. disapprovals), `create_responsive_search_ad` / `update_responsive_search_ad` / `update_ad_status` / `update_ad_final_url` (rewrite-and-swap fix path). See `SKILL.md` tool inventory.
- Related references: `references/policy-disapproval.md` (detection via GAQL + appeal limits + fix path), `references/rsa-writing.md` (asset-pool craft), `references/quality-score.md` (landing-page limits are off-platform).
- **Google Ads policy (verified against live pages 2026-06-15):**
  - [Google Ads policies — overview / the four families](https://support.google.com/adspolicy/answer/6008942) — Prohibited content, Prohibited practices, Restricted content and features, Editorial and technical.
  - [Misrepresentation](https://support.google.com/adspolicy/answer/6020955) (under Prohibited practices) — false affiliation/government-endorsement, Dishonest pricing practices (fee disclosure), unreliable claims.
  - [Government documents and services](https://support.google.com/adspolicy/answer/13156083) — certification + authorized-provider requirement; Google-generated "Not a government website" disclosure.
  - [Other restricted businesses → third-party consumer technical support](https://support.google.com/adspolicy/answer/13527027) — third-party consumer tech support not allowed.
  - [Editorial](https://support.google.com/adspolicy/answer/6021546) — capitalization/punctuation/symbols/repetition/business-identification rules.
  - [Trademarks](https://support.google.com/adspolicy/answer/6118) (under Restricted content and features) — complaint-driven restriction on ad text; reseller/informational/descriptive exceptions; allowed as keywords + display-URL.
  - All accessed 2026-06-15. Re-verify before quoting a specific rule as current.
