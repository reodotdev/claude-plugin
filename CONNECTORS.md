# Connectors and Context Files

`reo-mcp-tools` integrates with one required MCP connector and reads two optional user-context files. This document explains what each one provides and how to configure it.

## Required: `reo-gateway` MCP

The `reo-gateway` MCP is bundled with this plugin via `.mcp.json`. It is a remote HTTP MCP server hosted at `https://mcp.reo.dev/mcp`. First-time use will prompt for OAuth authentication in your browser.

Tools exposed:

| Tool | Purpose |
|---|---|
| `reo_get_account_research` | Composite endpoint — returns firmographic, engagement, activities, developers, prospects, and hiring data in a single call. Used by `account-research-v2`. |
| `reo_get_account_firmographic` | Company firmographic data (industry, HQ, founded year, tech stack, tech headcount, org_id). |
| `reo_get_tech_function_job_counts` | Active job counts broken down by tech function (Platform Eng, DevOps/SRE, etc.). |
| `reo_list_prospects` | LinkedIn prospects at a company, filterable by seniority, tech function, designation, and location. |
| `reo_get_account_activities` | Full activity feed for an account — deanonymized page visits, form captures, CLI commands, demo logins. |
| `reo_get_account_developers` | Developers who have engaged with your product at a given account. |
| `reo_list_accounts` | Your accessible accounts with customer fit score, country, and date filters. |
| `reo_list_segments` | Segments defined in your Reo tenant. |
| `reo_list_audiences` | Audiences defined in your Reo tenant. |
| `reo_get_tenant_settings` | Your tenant's ICP definition, territories, and high-intent page mappings. |
| `execute_open_data_query` | Sandboxed read-only SQL over open labor-market datasets (jobs, profiles, tech headcount). |

See `https://docs.reo.ai/mcp` for full tool schemas.

## Optional: `visualize` / `show_widget` tool

The `account-research-v2` skill renders its output via `show_widget` when that tool is available. Configure a `visualize` MCP server separately if you want inline HTML dashboards. Without it, the skill degrades gracefully to a markdown dashboard.

## Optional: calendar MCP

If a calendar MCP is available, `daily-account-prioritizer` can enrich its output with "accounts you have meetings with this week" as an additional signal. Any standard calendar MCP exposing a `list_events` tool works.

## Context files

Both context files live at the top level of the plugin under `references/`. They are loaded on demand by the skills and can be edited at any time.

### `references/me-context.md`

Your personal context: role, territory, ICP definition, objection handling notes, typical deal size, preferred outreach style. Used by `account-research` to personalize outreach notes and by `daily-account-prioritizer` to apply your territory filter.

### `references/my-company-context.md`

Your company's context: elevator pitch, primary value propositions, three–five customer proof points, main competitors and differentiation, pricing notes. Used by `account-research` to generate competitive context and suggested next steps.

Both files ship with placeholder stubs. Fill them in before production use.
