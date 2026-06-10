# Reference — Repeatable Optimization Loops

The daily loop (`references/daily-operator.md`) catches fires. These are the slower, scheduled loops that actually improve an account over time. Each is a named, repeatable routine with a clear trigger, the tools it uses, and a stop condition so it doesn't run forever or thrash.

All read steps are `run_gaql`; all writes are confirmed and reversible (`undo_change` covers creates and updates; removals are permanent).

## Weekly loops

### Search-term hygiene
- **Trigger:** weekly for active spenders.
- **Do:** pull `search_term_view` (last 7–14 days), run the term-level + n-gram passes (`references/search-term-mining.md`), and propose negatives. Curate recurring junk into a shared list (`create_negative_keyword_list` → `add_keywords_to_negative_list` → `link_negative_list_to_campaign`) so it covers future campaigns too.
- **Stop when:** no new high-spend zero-conversion terms/patterns this week.

### Budget reallocation
- **Trigger:** weekly.
- **Do:** read impression-share + CPA/ROAS (`references/ppc-math.md`). Move budget toward campaigns that are budget-capped *and* converting under break-even (`update_campaign_budget`, small steps), away from those over break-even CPA with no headroom.
- **Stop when:** profitable campaigns are no longer budget-capped, or you've hit the account's total budget.

### Bid tuning
- **Trigger:** weekly, on accounts using manual/enhanced CPC.
- **Do:** find keywords whose CPA is well above or below target with enough volume; adjust with `update_keyword` (one) or `bulk_update_bids` (many, ≤100). The maxCpcBid guardrail backstops over-aggressive raises.
- **Stop when:** outliers are within range, or volume is too thin to judge (sample-size discipline).

## Monthly / less-frequent loops

### Quality Score remediation
- **Trigger:** monthly.
- **Do:** pull QS components (`references/quality-score.md`), take the high-impression below-median keywords, and fix the specific weak component (new RSA for ad relevance, ad-group split for expected CTR, flag landing page).
- **Stop when:** the weak components have moved toward AVERAGE/ABOVE_AVERAGE on re-pull.

### Structure review
- **Trigger:** monthly, or when an ad group has grown unfocused.
- **Do:** check ad-group theming and naming (`references/campaign-structure.md`); propose splits/consolidation. Note: restructuring is manual tool-by-tool here (no bulk-restructure tool), so scope it small.

### Change-impact review
- **Trigger:** ~1–2 weeks after any significant change.
- **Do:** run the before/after protocol (`references/change-impact.md`) on the change's `auditId`; keep, revert (`undo_change`), or mark inconclusive.

## Running loops well
- **One loop at a time per review** so you can attribute effects; overlapping changes break change-impact analysis.
- **Each loop has a stop condition** — "nothing left to fix this week" is success, not a reason to invent changes.
- **Calibrate against the account's own history** (`references/benchmark-calibration.md`), not external targets.
- **Cadence scales with spend:** a high-spend account earns weekly attention; a small one may only need monthly. Match effort to what's at stake.
