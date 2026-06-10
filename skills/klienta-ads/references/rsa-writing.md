# Reference — Responsive Search Ad (RSA) Copywriting

A Responsive Search Ad gives Google a pool of headlines and descriptions and lets it assemble combinations per auction. Good RSAs give the system *useful variety* to work with; weak ones give it near-duplicates or off-message lines. This reference is how to write the asset pool that `create_responsive_search_ad` ships.

Tied to our surface: `create_responsive_search_ad` takes 3–15 headlines (≤30 chars each), 2–4 descriptions (≤90 chars each), and optional path1/path2 (≤15 chars). There is no pinning parameter and no experiments tool, so the guidance below works within that.

## Hard limits (enforced by the tool)
- **Headlines:** 3–15, each ≤30 characters. More is generally better — give the system room to test.
- **Descriptions:** 2–4, each ≤90 characters.
- **Display paths:** path1/path2 optional, ≤15 characters each; use them to reinforce relevance (e.g. the category), not to repeat the domain.

Write within the character limits deliberately — a headline cut off mid-word reads as careless. Count characters as you draft.

## Variety is the job
The system needs genuinely different angles to test, not fifteen rewordings of one. Cover a spread:
- **The keyword/intent** — echo what the user searched (this also helps ad relevance, `references/quality-score.md`).
- **The offer / value** — what they get (the benefit, not just the feature).
- **Proof / differentiation** — what makes this the right choice.
- **A call to action** — what to do next.
- **Specifics** — concrete terms the business can stand behind (avoid invented numbers or claims you can't support).

If two headlines could swap without changing the meaning, one is wasted. Aim for distinct themes across the pool.

## Pinning balance (conceptual — not in this tool)
Pinning forces a headline/description into a fixed position. Our `create_responsive_search_ad` doesn't expose pinning, which is fine: heavy pinning is usually counterproductive because it shrinks the combinations the system can test. The takeaway for our surface: rely on a *diverse, individually-true* asset pool rather than trying to control ordering. If a legal/brand line absolutely must always appear, that's a real reason to want pinning — flag it as a limitation rather than implying the tool can pin.

## Relevance and structure
- **One theme per ad group, one ad pool per theme.** An RSA can only be relevant if the ad group's keywords share intent (`references/campaign-structure.md`). A broad ad group produces vague headlines.
- **Match the landing page.** Headlines should promise what the destination delivers; a mismatch hurts landing-page experience and wastes clicks.

## Iterating without an experiments tool
There's no A/B experiments tool here, so iterate by observation: ship the RSA, let it gather data, then read `ad_group_ad.ad_strength` and performance with `run_gaql`, and ship a revised `create_responsive_search_ad` (pause or remove the old one) when assets underperform. Be honest that this is sequential observation, not a controlled split test — and don't over-read small samples (sample-size discipline applies here too).
