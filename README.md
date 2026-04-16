# Reo MCP Tools

Turn Reo into your GTM copilot inside Claude — research accounts, prioritize your daily list, and surface the buyers and engaged developers that matter, all grounded in Reo's buyer intelligence data.

## Overview

`reo-mcp-tools` is a Claude Code plugin that combines the Reo gateway MCP server with opinionated sales skills and slash commands for SDR and AE workflows. It replaces the swivel-chair workflow of pulling account research, prospect data, engagement signals, hiring trends, and CRM context from half a dozen tools with single commands that return structured, actionable briefs in seconds.

The plugin bundles:

- The `reo-gateway` remote MCP server — read-only access to firmographics, tech stack, hiring signals, LinkedIn prospects, deanonymized engaged developers, and account activity across ~90M enriched profiles.
- Two skills that orchestrate the MCP tools into production-ready sales workflows.
- Two slash commands that invoke those skills directly.

## Requirements

- **Claude Code** 2.1 or later (or Claude Desktop with plugin support).
- **A Reo account** with API access — contact [hello@reo.ai](mailto:hello@reo.dev) to provision your tenant.
- **Network access** to `https://mcp.reo.dev/mcp` from the machine running Claude.
- **Optional:** a `visualize` / `show_widget` tool configured separately if you want the `account-research` skill to render HTML dashboards inline. Without it, the dashboard falls back to markdown.

## Skills

| Skill | Triggers |
|---|---|
| `account-research-v2` | "research [company]", "pull up account intel on [domain]", "prep me for my call with [company]", "what do we know about [company]" |
| `daily-account-prioritizer` | "what accounts should I focus on today", "give me my hot list", "prioritize my accounts", "rank my accounts by fit", "build my SDR list" |

## Commands

| Command | Usage |
|---|---|
| `/account-research <company or domain>` | Generate a full account intelligence dashboard for a single company. |
| `/daily-account-prioritizer [context]` | Produce a ranked, scored daily account list for SDR or AE triage. |

## What each skill produces

**`account-research-v2`** returns a dark-mode HTML dashboard covering: firmographics (industry, tech headcount, HQ, founded year, tech stack), ICP fit score with a per-criterion breakdown, engagement score, deanonymized developers who have engaged with your product (with page-level activity history), ranked LinkedIn prospects filtered by ICP persona, active hiring signals by tech function, recent news and trigger events, competitive context, buyer KPIs by role, and 3–5 prioritized next-step actions. Uses the `reo_get_account_research` composite endpoint for faster data fetching (single API call instead of 5–7).

**`daily-account-prioritizer`** returns a ranked account list (typically 10–25 accounts) scored on four dimensions: ICP fit, developer funnel stage, activity score, and recency momentum. Territory and CRM stage act as hard filters. Each entry includes a one-line "why this account today" rationale and recommended next action.

## Setup

### Option 1 — Quick dev session (no install)

```bash
claude --plugin-dir /path/to/reo-mcp-tools
```

### Option 2 — Permanent install via local marketplace

```bash
claude plugin marketplace add /path/to/reo-mcp-tools
claude plugin install reo-mcp-tools@reo-mcp-tools-marketplace
```

### Option 3 — Install from a public marketplace

Once listed in the Anthropic `knowledge-work-plugins` marketplace:

```bash
claude plugin marketplace add anthropics/knowledge-work-plugins
claude plugin install reo@knowledge-work-plugins
```

### Verify

- `/mcp` in Claude Code should show `reo-gateway` as connected.
- First tool call will trigger OAuth in your browser — log in with your Reo credentials.
- `/` should show `/account-research` and `/daily-account-prioritizer` in the menu.

## User context

The `account-research` skill reads a user-context file (`references/me-context.md`) to personalize outreach notes and ICP scoring. Update that file with your ICP profile, your company's elevator pitch, and any persona-specific talking points. See `CONNECTORS.md` for the full list of context files and connectors the plugin can use.

## Customization

Both skills expose a `CONFIG` section at the top of their `SKILL.md` — edit those values (lookback windows, max contacts per section, high-intent pages, which dashboard sections are shown) to tune the output without touching the skill logic. Changes apply on the next invocation.

## Support

- **Docs:** https://docs.reo.dev
- **Email:** support@reo.dev
- **Issues:** use the GitHub issues tab on this repo.

## License

Apache License 2.0. See [LICENSE](LICENSE).
