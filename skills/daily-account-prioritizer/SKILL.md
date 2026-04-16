---
name: daily-account-prioritizer
description: >
  Generates a holistic, scored, SDR/AE-ready list of accounts to focus on today —
  combining ICP fit, dev funnel stage, activity score, and recency momentum signals
  from reo, with territory and CRM stage acting as hard filters. Use this skill
  whenever someone asks: "what accounts should I focus on today?", "give me my daily
  account list", "who should I call today?", "prioritize my accounts", "show me my
  hot list", "rank accounts by fit", "build my SDR list", "score my accounts", or
  any request for a ranked or scored account list. Always trigger this skill for
  daily pipeline triage workflows.
compatibility: "Requires: reo_list_accounts (account-list API), reo_get_account_activities, reo_get_account_developers, reo_list_prospects."
---

# Daily Account Prioritizer — Skill Instructions

## What this skill does

Produces a ranked, scored list of accounts for an SDR or AE to act on today. It
combines **account-level fit** (is this worth my time?) with **developer-level
activity** (is something actually happening right now?), then layers a **recency
momentum** score on top so accounts that are heating up this week surface first.

Territory and CRM stage are **filters** — they remove accounts from the pool
before scoring. They never contribute to the score itself.

---

## Scoring overview (100 points total)

| Component       | Weight | Source                                        |
|-----------------|--------|-----------------------------------------------|
| ICP fit         | 20     | `customer_fit` from account-list API          |
| Funnel fit      | 25     | `developerStage` from account-list API        |
| Activity score  | 25     | `developerActivityScore` from account-list API|
| Recency score   | 30     | `activity_trend` + `reo_get_account_developers`|

The exact thresholds are fixed and documented in Step 3. **Never** rescale or
reweight these — they were tuned on a spreadsheet the user maintains and are the
source of truth.

---

## Two-stage scoring logic

1. **Stage A — coarse scoring (ICP + Funnel + Activity, 70 pts max).**
   A single paginated call to `reo_list_accounts` returns all fields needed for
   Stage A. Apply hard filters. Score every remaining account and keep the top
   50–100.
2. **Stage B — recency scoring (30 pts).**
   Computed entirely from data already in the account-list response
   (`activity_trend` for 3D) plus `reo_get_account_developers` per shortlist
   account (for Mgr+ and deanon dev count). No additional bulk call needed.
   Take the top 20.
3. **Enrichment — signals detail.**
   For the top 20 only, call `reo_get_account_activities` (activity rows for
   the card) and `reo_list_prospects` (Step 5 prospects). Developer data is
   already in hand from Stage B.

---

## Step 0 — Confirm territory

Ask once: *"What's your territory? (e.g. US + Canada, EMEA, or 'global')"*
Wait for the answer unless it's obvious from context. Territory becomes the
`countryList` parameter passed to `reo_list_accounts` — it is a server-side
filter, so out-of-territory accounts never appear in the response at all.

If the user says "global", omit `countryList` from the request (do not pass an
empty array — omit the field entirely so the API returns all geographies).
If an account's country is unknown, include it.

---

## Step 1 — Pull the candidate pool via reo_list_accounts

Call the account-list API (`reo_list_accounts`) with the following fixed
parameters. Paginate until all results are retrieved.

```json
{
  "pageNo": 1,
  "pageSize": 50,
  "sortKey": "LATEST_ACTIVITY",
  "direction": "DESC",
  "fromDate": "1970-01-01T00:00:01.193Z",
  "toDate": "<today at 23:59:59 UTC>",
  "activitySurgeFilterApplied": true,
  "customerFitScore": ["STRONG", "MODERATE"],
  "developerScore": ["HIGH"],
  "countryList": ["<territory from Step 0>"]
}
```

**Why these parameters:**
- `activitySurgeFilterApplied: true` — pre-filters to accounts with a current
  activity surge. Combined with `sortKey: LATEST_ACTIVITY`, dormant accounts
  never appear regardless of pool size.
- `customerFitScore: ["STRONG","MODERATE"]` — excludes WEAK ICP accounts
  server-side. A WEAK account (5 pts ICP + capped activity) cannot reach the
  top 20 anyway; excluding them reduces pages to paginate.
- `developerScore: ["HIGH"]` — same logic; LOW activity accounts are scored out
  before they'd ever reach the shortlist.
- `countryList` — territory is applied server-side; no post-filter needed.

Paginate by incrementing `pageNo` until the response returns an empty list or
fewer results than `pageSize`. Collect all account objects. This is the
**raw candidate pool**.

**Fields used from each account object:**

| Response field              | Used for                              |
|-----------------------------|---------------------------------------|
| `org_id`                    | Key for all downstream API calls      |
| `domain`                    | Display + `reo_list_prospects` call   |
| `org_name`                  | Card display                          |
| `customer_fit`              | ICP score (3A)                        |
| `developerStage`            | Funnel score (3B)                     |
| `developerActivityScore`    | Activity score (3C)                   |
| `activity_trend[]`          | Recency delta score (3D)              |
| `deAnonymisedDevCountRange` | Deanon dev count — card display       |
| `crmAccountStage` / `stage` | CRM hard filter (Step 2)              |
| `latestActivity`            | Recent-activity hard filter (Step 2)  |
| `techFunctionsCount`        | Step 5 prospect ranking signal        |
| `estimated_employee_range`  | ICP fallback if `customer_fit` absent |
| `location` / `state`        | Display on card                       |
| `industry`                  | Display on card                       |

---

## Step 2 — Apply hard filters

Remove accounts where:

- **CRM stage filter** — `crmAccountStage` or `stage` ∈
  { Customer, Closed Lost, Churned, Disqualified, CUSTOMER }. Check both
  fields; if either matches, exclude the account.
- **Recent activity filter** — `latestActivity` (Unix ms timestamp) is older
  than 7 days ago. Convert to a date and compare to today.
  `latestActivity < (now_ms - 7 * 86400 * 1000)` → exclude.

Territory is already applied server-side. Do not re-filter on country here.

What remains is the **eligible pool**.

---

## Step 3 — Score each account

### Stage A — coarse score (ICP + Funnel + Activity)

Run on **every eligible account** using fields already in the API response.

#### 3A. ICP fit (max 20 pts)

Map `customer_fit` directly:

```
STRONG   → 20
MODERATE → 10
WEAK     →  5
(absent  →  5)   # neutral-low default, never 0
```

If `customer_fit` is absent, derive from `estimated_employee_range` using the
DEFAULT CONFIG employee range tiers.

#### 3B. Funnel fit (max 25 pts)

Map `developerStage`:

```
DEPLOY      → 25
BUILD       → 17
EVALUATE    → 10
DISCOVER    →  3
(unknown    →  3)
```

#### 3C. Activity score (max 25 pts)

```
activity_score = min(developerActivityScore * 0.25, 25)
```

The cap at 25 matters — `developerActivityScore` can be arbitrarily large.

#### Stage-A subtotal

```
stage_a = icp_fit + funnel_fit + activity_score     # max 70
```

Sort descending by `stage_a`. Keep the **top 100** (drop to 50 if the eligible
pool is already small). This is the shortlist for Stage B.

---

### Step 3.5 — Compute Stage B data for the shortlist

#### 3.5A — Prior-week activity delta (from activity_trend)

`activity_trend` is an array of `{ date, score }` objects covering 14+ calendar
days. Compute:

```
current_score = sum of score values where date >= (today - 7 days)
prior_score   = sum of score values where date is in (today - 14 days, today - 7 days)
delta_activity = current_score - prior_score
```

If `activity_trend` has fewer than 8 entries covering the prior window, set
`prior_score = 0` and treat the delta as the full current score (this is
conservative — it won't over-inflate the delta since prior = 0 is the floor).

#### 3.5B — Deanon devs and Mgr+ (from reo_get_account_developers)

For each shortlist account, call:
```
reo_get_account_developers(account_id = org_id)
```

Derive:
- `mgr_plus_identified` — boolean: does any returned developer have a title
  containing: Head, Manager, Director, VP, SVP, EVP, CXO, C-level, Founder,
  Co-founder, Partner, Chief? If yes → True.
- `deanon_devs_current` — count of developers returned. Use this alongside
  `deAnonymisedDevCountRange` from the account-list response for card display.

**Note on 3E:** Prior-week deanon dev count is not available from any API
without `execute_open_data_query`. Score 3E as 0 and show *"Δ deanon devs n/a"*.

---

### Stage B — recency score (max 30 pts)

#### 3D. Jump in activity score, last 7 days (max 15 pts)

```
delta_activity >  20   → 15
delta_activity >  10   → 10
delta_activity >   0   →  5
delta_activity ≤   0   →  0
```

#### 3E. % jump in deanon devs (max 5 pts)

Always 0. Show *"Δ deanon devs n/a"* on card. Known limitation — prior-week
dev count requires open_data. The remaining 95 points differentiate accounts
correctly.

#### 3F. Mgr+ identified (max 10 pts)

```
mgr_plus_identified = True   → 10
mgr_plus_identified = False  →  0
```

#### Recency subtotal

```
recency = delta_activity_pts + 0 + mgr_plus_pts     # max 25 effective, 30 theoretical
```

---

### 3G. Composite

```
composite = icp_fit + funnel_fit + activity_score + recency     # max 100
```

Sort the shortlist by `composite` descending. Keep the **top 20** for output.

---

## Step 4 — Enrich the top 20 with signal detail

Developer data is already in hand from Step 3.5B — **do not re-call
`reo_get_account_developers`**.

For activity rows, call:
```
reo_get_account_activities(account_id = org_id)
```
for each of the top 20 accounts.

From the activity response, extract:
- **Top 3 recent activities** — `activity_type`, `activity_source`, `occurred_at`.
  Prefer the most recent *high-weight* events (forks, installs, copy-command,
  pricing page, form capture) over low-signal page visits. If the three most
  recent are all low-signal, surface the newest high-signal event from the last
  7 days alongside two recent page visits so the card has texture.

From `reo_get_account_developers` results already in hand:
- **Known developers** — `name`, `latest_designation` (title), `linkedin_url`.
  If none, show *"— not yet identified"*.

---

## Step 5 — "More prospects" selection

For each of the top 20 accounts, call:
```
reo_list_prospects(
  domain        = account.domain,
  seniority     = ["Founder","C-Level","VP","Director","Head"],
  tech_function = [
    "Artificial Intelligence & Machine Learning",
    "Cloud & Infrastructure",
    "DevOps Platform & Reliability",
    "Backend Engineering",
    "Frontend Engineering",
    "Full Stack",
    "Security & Compliance",
    "Product Design & UX",
    "Engineering Leadership"
  ],
  page_size = 20
)
```

Exclude anyone already in the account's `reo_get_account_developers` list.
Select up to **5 most relevant** remaining people.

Use `techFunctionsCount` from the account-list response to guide ranking:
- High `ai_ml_count` → prefer AI/ML and Engineering Leadership prospects.
- High `devops_platform_reliability_count` or `cloud_infrastructure_count` →
  prefer Cloud & Infrastructure / DevOps prospects.
- If one strong Mgr+ deanon dev already identified, prefer a *different*
  function to broaden committee coverage.
- All else equal: VP > Director > Head.

Return per prospect: `name`, `title`, `tech_function`, `linkedin_url`.
Don't pad to 5 with weaker candidates.

---

## Step 6 — HTML dashboard output

Render a single dark-mode HTML dashboard. Save to outputs folder and share link.

**Header**
- Title: *"🎯 Today's Focus List — [Date]"*
- Subheader: territory · eligible pool size · shortlist size · data freshness
- Legend: ICP 20 · Funnel 25 · Activity 25 · Recency 30
- Banner: *"Δ deanon dev scoring (3E) unavailable — max attainable score: 95"*

**One card per account, ranked 1–20**

Each card must contain:

1. **Header row** — rank, company name, domain, composite score with a bar.
   Bar colours: green ≥ 70 · amber 45–69 · red < 45.

2. **Score breakdown** — four pills: *"ICP 10/20 · Funnel 25/25 · Activity 18/25 · Recency 22/30"*.
   Compact recency line: *"Δ activity +15 · Δ deanon devs n/a · Mgr+ ✓"*.

3. **"Why it's here"** — one concrete sentence. NOT generic.
   Good: *"2 Backend Engineers forked your owned repo and ran the install command in the last 24h."*
   Good: *"Funnel stage = DEPLOY and activity climbed +28 vs last week — mostly docs and product pages."*
   Bad: *"This account has high activity."*  ← never produce this
   Bad: *"Strong ICP fit and good engagement."*  ← never produce this

4. **Top 3 recent activities** — `activity_type · source · relative time`.
   Example: *"COPY_PACKAGE_MANAGER · PRODUCT_JS · 4h ago"*.

5. **Known developers** — from `reo_get_account_developers`. Each row:
   `name · title · LinkedIn ↗`. If empty: *"— not yet identified"*.

6. **More prospects (up to 5)** — Step 5 list.
   Each row: `name · title · tech function · LinkedIn ↗`.

7. **Recommended action** — one prescriptive line:
   - DEPLOY + high recency → *"Call today — expansion signal"*
   - BUILD/EVALUATE + Mgr+ identified → *"Book a meeting with [name]"*
   - DISCOVER + strong ICP + rising activity → *"Send targeted content + monitor"*
   - Strong ICP + flat activity → *"Warm outbound to the Step-5 prospects"*
   - High activity + MODERATE ICP → *"Qualify before investing time"*

**Styling**
- Background `#0f0f12` · card `#1a1a22` · border `#27272e`
- Accent `#6366f1` · text `#e5e7eb` · muted `#9ca3af`
- Score bar: green `#22c55e` · amber `#f59e0b` · red `#ef4444`
- Readable sans-serif, plenty of whitespace. No emoji soup.

---

## Step 7 — Narrative summary

3–5 sentences in chat alongside the dashboard link:
- Dominant pattern across the top 20
- Single most urgent action and why
- Notable outlier (e.g. a DEPLOY account showing churn risk, a cold account
  that suddenly appeared)

---

## Edge cases

- **Fewer than 5 accounts survive Step 2.** Remove `activitySurgeFilterApplied`
  from the request, re-fetch, and show a banner noting surge filter was lifted.
  If still fewer than 5, also widen `fromDate` window to 30 days back and note it.
- **`customer_fit` absent.** Derive ICP score from `estimated_employee_range`
  using DEFAULT CONFIG employee range tiers.
- **`crmAccountStage` and `stage` both absent.** Skip CRM filter; note in
  dashboard banner that CRM filtering was off.
- **`activity_trend` has fewer than 8 entries.** Set `prior_score = 0` for 3D
  computation; show *"Δ activity (partial data)"* on the card instead of the
  points earned.
- **`org_name` blank.** Show domain only, labelled "(unnamed)".

---

## API call summary (what runs and when)

| Step   | API call                      | Runs against         |
|--------|-------------------------------|----------------------|
| Step 1 | `reo_list_accounts` (paginated) | Full tenant          |
| Step 3.5B | `reo_get_account_developers` | Top 100 shortlist only |
| Step 4 | `reo_get_account_activities`  | Top 20 only          |
| Step 5 | `reo_list_prospects`          | Top 20 only          |

Total per-account calls: max ~120 (100 dev lookups + 20 activity + 20 prospects).
No firmographic calls. No audience discovery. No open_data dependency.

---

## DEFAULT CONFIG

Only used if `customer_fit` is absent from the account-list response.

### icp_employee_range
```
p1_ranges: 1001–5000, 5001–10000, 10001–50000, 50001–100000, 100000+
p2_ranges: 251–1000, 51–250
```

### Derivation rule (employee range only, when customer_fit absent)
```
employee_range ∈ p1   → Strong (20)
employee_range ∈ p2   → Medium (10)
any other             → Weak (5)
```

### Mgr+ title keywords
```
Founder, Co-founder, Chief, CXO, CEO, CTO, CIO, CISO, CPO, COO,
Partner, President, VP, SVP, EVP, Head of, Director, Manager
```

### Tech function canonical names (for Step 5 filtering)
```
Artificial Intelligence & Machine Learning
Cloud & Infrastructure
DevOps Platform & Reliability
Backend Engineering
Frontend Engineering
Full Stack
Security & Compliance
Product Design & UX
Engineering Leadership
```
