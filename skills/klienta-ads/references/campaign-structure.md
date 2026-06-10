# Reference — Campaign Structure & Naming

Structure is leverage: a well-organized account is easier to read, bid, and write ads for, and it earns better Quality Score because ads can stay relevant to tight keyword themes. This reference is how to organize Search campaigns and ad groups within our toolset — and an honest note on what restructuring we can and can't automate.

## The core principle: tight themes

The single most useful structural rule: **each ad group holds keywords that share one intent**, so its RSA can speak directly to them (`references/quality-score.md`, `references/rsa-writing.md`). A grab-bag ad group forces vague ads and drags expected-CTR and ad-relevance ratings down.

- **Campaign** = a budget + targeting + bidding boundary. Split into separate campaigns when you need *separate budgets or separate targeting* (e.g. different geos, or brand vs non-brand so brand traffic doesn't eat prospecting budget).
- **Ad group** = a tight keyword theme with its own ad(s). If you can't write one focused RSA that fits every keyword in the group, the group is too broad — split it.

How tight to go (a few specific keywords per group vs broader themed groups) is a trade-off, not a law: tighter groups give sharper relevance and control but more to manage; broader groups are simpler but vaguer. Lean tighter where relevance and CPC matter most, broader where volume is thin.

## Naming

Consistent names make an account legible at a glance and make `run_gaql` filtering easy. Pick a convention with the user and apply it uniformly — for example encoding the meaningful dimensions in order, like `Geo | Theme | MatchType` for campaigns and the keyword theme for ad groups. The exact scheme matters less than using it consistently. `update_campaign` can rename a campaign (name field); there's no ad-group rename tool, so set ad-group names correctly at creation (`create_ad_group`).

## Budget shape

Bias budget toward proven performers, but keep a slice for testing new themes and a buffer for scaling a winner — don't starve experiments or leave a profitable, budget-capped campaign capped (`references/ppc-math.md`). Treat any split as a hypothesis to measure (`references/change-impact.md`), not a one-way door.

## Honest limits — what we can and can't restructure

Our surface builds and edits **piece by piece**: `create_search_campaign`, `create_ad_group`, `add_keywords` / `bulk_add_keywords`, `create_responsive_search_ad`, plus the move/remove tools. There is **no bulk-restructure or move-keywords-between-ad-groups tool**, and no ad-group rename. So restructuring an existing account means rebuilding the target structure (new ad groups + keywords + ads) and pausing/removing the old — a deliberate, confirmed, multi-step job, not a one-click reshuffle.

Because of that, **scope restructures small and sequence them**: do one ad group at a time, verify each step (`run_gaql` read-back), and prefer pausing the old over removing it until the new structure proves out (removals aren't undoable). For a large reorganization, lay out the plan for the user and get approval before starting, rather than implying the account can be re-shaped in a single move.
