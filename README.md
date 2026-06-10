# Klienta — Google Ads plugin for Claude Code

Manage **your own** Google Ads account through AI chat. Klienta connects Claude Code to your Google Ads account via the hosted **Klienta MCP server** (`https://klienta.co/mcp`) and ships two skills that govern *how* Claude analyzes and changes the account so it stays safe.

- **`klienta-ads`** — operating contract for managing Search campaigns, keywords, budgets, ads, and conversion tracking. Enforces measure-before-acting, evidence-backed recommendations, smallest reversible action, **confirmation before every write**, and read-back verification after writes.
- **`klienta-audit`** — strictly **read-only** account audit playbook. Scans campaigns, keywords, budgets, search terms, quality, and impression share, then produces a scored health report. Never writes.

The MCP server is a **streamable HTTP** endpoint with OAuth — on first use Claude Code opens a browser to authorize access to your Google Ads account. Your Google refresh token is stored encrypted server-side; nothing sensitive lives in this repo.

---

## Install (recommended — plugin marketplace)

In Claude Code:

```
/plugin marketplace add Onurcanozer/klienta-plugin
/plugin install klienta@klienta
```

Then restart Claude Code (or reload plugins). On first Google Ads request, Claude Code will prompt you to authenticate the `Klienta-GoogleAds` MCP server in your browser.

Verify:

```
/plugin            # klienta should be listed/enabled
/mcp               # Klienta-GoogleAds should appear (authenticate if prompted)
```

## Install (manual — MCP server only, no skills)

If you only want the MCP tools without the plugin skills:

```
claude mcp add --transport http Klienta-GoogleAds https://klienta.co/mcp
```

Then run `/mcp` and authenticate when prompted. (This adds just the tool surface; the `klienta-ads` / `klienta-audit` skills come only with the plugin install above.)

## Usage

```
audit my Google Ads account          # → klienta-audit (read-only health report)
optimize my search campaigns         # → klienta-ads (proposes changes, confirms before writing)
find wasted spend in search terms
```

## What's in this repo

```
.
├── .claude-plugin/
│   ├── marketplace.json     # marketplace manifest
│   └── plugin.json          # plugin manifest (name, version, author)
├── .mcp.json                # Klienta-GoogleAds MCP server (streamable HTTP)
├── skills/
│   ├── klienta-ads/         # SKILL.md + references/
│   └── klienta-audit/       # SKILL.md
└── README.md
```

## Safety

- Every account-changing action requires explicit confirmation; new campaigns start **paused** by default.
- Server-side guardrails (`forbiddenCustomerIds`, `maxDailyBudget`, `maxBudgetIncreasePct`, `maxCpcBid`, `requirePausedOnCreate`) constrain writes regardless of what the model proposes.
- This repository contains **no credentials, tokens, or account IDs** — authentication is handled entirely by the hosted MCP server's OAuth flow.

## License

MIT © Onurcanozer — https://klienta.co
