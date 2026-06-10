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

## Testing with experiments

For a *controlled* comparison — not just sequential observation — use the experiment tools to A/B test ad copy at the campaign level. A Search experiment splits real traffic between a control (your current campaign) and a treatment (a variant), so the difference you measure is caused by the change, not by timing.

Flow with our tools:
1. **Set up:** `create_experiment` (SEARCH_CUSTOM, a future start/end date — created in SETUP, nothing serves yet).
2. **Define the split:** `add_experiment_arms` — the control arm carries the base campaign; the treatment arm is the variant (scheduling builds its trial campaign). Traffic splits must sum to 100 (e.g. 50/50). Then edit the treatment's trial campaign's ads to hold the new RSA you want to test.
3. **Run:** `schedule_experiment` — starts it (long-running; the tool waits) and the experiment goes ENABLED with a live trial campaign.
4. **Read the result:** with `run_gaql`, compare control vs treatment over the experiment window (and respect sample size — see `references/ppc-math.md`). An experiment is a cleaner read than the sequential ship-and-watch approach above.
5. **Decide:**
   - Winner is the treatment → `promote_experiment` applies it to the base campaign (**this changes the base campaign — confirm with the user first; it is not undoable via undo_change**).
   - No clear winner / keep control → `remove_experiment` tears the whole thing down (cascade-deletes the trial campaign). `remove_experiment` is the reliable teardown in any state; `end_experiment` only works on a genuinely running experiment and can return an opaque error on a fresh one.

So: use the diverse asset pool for the everyday ship-and-observe loop, and reach for an experiment when a copy (or any campaign-level) change is important enough to warrant a controlled, attributable test.

## Relevance and structure
- **One theme per ad group, one ad pool per theme.** An RSA can only be relevant if the ad group's keywords share intent (`references/campaign-structure.md`). A broad ad group produces vague headlines.
- **Match the landing page.** Headlines should promise what the destination delivers; a mismatch hurts landing-page experience and wastes clicks.

## Iterating: observe, or test
Two ways to iterate, lightest first:
- **Ship and observe (everyday):** ship the RSA, let it gather data, read `ad_group_ad.ad_strength` and performance with `run_gaql`, and ship a revised `create_responsive_search_ad` (pause/remove the old one) when assets underperform. Be honest that this is sequential observation, not a controlled split test — and don't over-read small samples (sample-size discipline applies here too).
- **Experiment (when the change matters):** for an important, attributable comparison, run a Search experiment (see "Testing with experiments" above) so traffic is split between control and treatment and the difference is genuinely caused by the change.
