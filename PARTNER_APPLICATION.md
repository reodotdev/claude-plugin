# Reo — Application for Anthropic's `knowledge-work-plugins` Marketplace

## One-liner

Reo is a buyer intelligence platform for B2B revenue teams — we turn ~90M enriched LinkedIn profiles, firmographic data, hiring signals, and first-party engagement activity into a single ranked view of which accounts and people to work today.

## Why Claude + Reo

Sales reps live in a swivel-chair workflow. A typical pre-call prep today: LinkedIn for the prospect, the company site for positioning, BuiltWith for tech stack, LinkedIn Jobs for hiring signals, Crunchbase for funding, internal CRM for engagement, and a Google Doc to stitch it all together. An SDR spends 25–40 minutes per account. An AE running 8 discovery calls a day simply doesn't do it, and the call suffers.

With Reo inside Claude:

- **Account research** — `/account-research vercel` replaces that 30-minute tab tour with one command that produces a structured dashboard in ~60 seconds.
- **Morning triage** — `/daily-account-prioritizer` gives an SDR a ranked list of accounts to hit first, scored by ICP fit plus recent activity plus hiring momentum — not a static list from last quarter.
- **Call prep** — mid-conversation, Claude can pull a named prospect's tech function, recent job postings, and engagement history without the rep leaving the chat.
- **Pipeline review** — an AE asks Claude "which of my open opps have cooled in the last two weeks" and gets an answer grounded in real activity data, not vibes.

The qualitative shift: Claude becomes a competent sales analyst, not a writing assistant.

## What we ship

The `reo-mcp-tools` plugin bundles:

- **1 remote MCP server** (`reo-gateway`) exposing ~12 read tools: `reo_get_account_firmographic`, `reo_get_tech_function_job_counts`, `reo_list_prospects`, `reo_get_account_activities`, `reo_get_account_developers`, `reo_list_accounts`, `reo_list_segments`, `reo_list_audiences`, plus a sandboxed open-data SQL interface over our ClickHouse jobs / profiles / tech-headcount dataset.
- **2 skills + 2 slash commands**: `/account-research` and `/daily-account-prioritizer`.

A concrete example — `/account-research vercel` returns a dark-mode HTML dashboard containing:

- Firmographics (size, geography, funding, current headcount by function)
- Tech stack and hiring signals (open roles by team, 90-day delta)
- Deanonymized developers engaged with the customer's product, with LinkedIn links
- Ranked LinkedIn prospects filtered by ICP persona, with a recommended opener per contact
- Engagement score, ICP fit score, recent news, named competitors, and three suggested next steps

Output is a self-contained artifact the rep can skim in 90 seconds or forward to their AE.

## Why this fits `knowledge-work-plugins`

Reo sits in the same category as Common Room — buyer / customer intelligence for revenue teams — but with a developer-flavored dataset (deanonymized engaged developers, tech-function hiring, stack signals) that Common Room doesn't cover. We're complementary, not competing with the current roster:

- Asana and Linear serve the delivery side of the org.
- Common Room covers community signal.
- Reo covers outbound prospecting and account intelligence for dev-tools and infra companies — a category we believe Anthropic's plugin catalog is currently missing. Listing Reo gives partner coverage for "outbound / dev-centric GTM" without overlap.

## Production readiness

- **MCP hosting**: migrating the `reo-gateway` MCP from local header-based auth (`tenant_id` / `username`) to a public HTTPS endpoint behind OAuth 2.1 with PKCE, token rotation, and per-tenant scoping. Target: live before any public marketplace listing.
- **Transport**: TLS 1.3, rate-limited per tenant, structured audit logs.
- **Plugin**: semver-versioned, changelog-maintained, published from a public GitHub repo under Apache 2.0.
- **Docs**: public API reference, quickstart, and per-tool schema docs.
- **Support**: dedicated email + Slack channel for integration partners; on-call rotation for P1.

Honest status: OAuth 2.1 rollout is in progress, not done. Timeline and scope are committed — happy to share the implementation plan during technical review.

## Customer validation and traction

- [N] paying customers, [$X] ARR
- Notable logos: [to fill]
- Usage: [accounts enriched / week, prospects surfaced / week]
- Case study: [customer] reduced SDR pre-call prep from [X] min to [Y] min

## What we're asking Anthropic for

1. Listing in the `knowledge-work-plugins` marketplace (vendored under `partner-built/reo`).
2. Technical review of the MCP server and plugin before listing.
3. Co-marketing where it fits — launch post, category landing page mention, joint customer story with a mutual user. Not a blocker.

## Contact and links

- Founder: [Name], [email]
- Plugin repo: [github.com/.../reo-mcp-tools]
- MCP server: [https://mcp.reo.dev]
- Product: https://reo.ai
- Docs: [docs.reo.ai]
