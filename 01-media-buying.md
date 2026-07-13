# PRD 01 -- Media Buying

**Department Lead:** Sebastian Roman
**Users:** Growth user type -- Gene Alder, Sebastian Ramon, Rodrigo Briao
**Psycle user type:** `growth`
**Source projects:** Psycle-Data (`/mb`, `/topads`, `/agency`, meta-sync), Dashboard (Meta Ads drill-down, winning ads, geo/heatmaps)

---

## 1. Data Schema

### Tables Required

**`meta_ad_accounts`**
- id (Meta account ID, e.g. `act_XXXXXXXXX`)
- name
- clinic_id (FK -> clinics, nullable for unmapped accounts)
- currency
- status (ACTIVE, DISABLED)
- last_synced_at
- created_at, updated_at

**`meta_campaigns`**
- id (Meta campaign ID)
- ad_account_id (FK)
- name
- status (ACTIVE, PAUSED, DELETED, ARCHIVED)
- objective
- daily_budget, lifetime_budget
- created_at, updated_at

**`meta_adsets`**
- id (Meta adset ID)
- campaign_id (FK)
- name
- status
- targeting (JSONB -- geo, age, interests)
- optimization_goal
- created_at, updated_at

**`meta_ads`**
- id (Meta ad ID)
- adset_id (FK)
- name
- status
- creative_id
- preview_url
- first_active_date, last_active_date
- total_spend (denormalized for fast ranking)
- created_at, updated_at

**`meta_insights`** (the core analytics table)
- id
- object_id (ad/adset/campaign/account ID)
- object_type (ad, adset, campaign, account)
- date (DATE, PT-bucketed)
- impressions, clicks, spend
- actions (JSONB -- contains lead counts by action_type)
- cost_per_action_type (JSONB -- contains CPL by action_type)
- ctr, cpc, cpm
- created_at

**`meta_ad_rejections`**
- id
- ad_id (FK -> meta_ads)
- review_status (rejected, approved, appeal_pending)
- rejection_reason
- detected_at
- resolved_at
- source (gmail_poll, manual, meta_webhook)
- created_at

**`ad_account_mappings`** (the accounts matcher)
- id
- clinic_id (FK -> clinics)
- ad_account_id (FK -> meta_ad_accounts)
- campaign_filter (JSONB array of campaign IDs, null = all)
- result_type (lead_form, view_content, custom)
- is_active
- created_at, updated_at

### Materialized Views

**`mv_ad_performance_daily`** (replaces Sebastian's `meta.ad_performance_daily`)
- Grain: 1 row per (ad_id, date)
- Columns: ad_id, ad_name, adset_id, campaign_id, ad_account_id, clinic_id, date, spend, impressions, clicks, leads (extracted from actions JSONB), cpl (extracted from cost_per_action_type JSONB), first_active_date, last_active_date
- Refresh: after each meta sync job completes

**`mv_clinic_spend_daily`** (clinic-level rollup)
- Grain: 1 row per (clinic_id, date)
- Columns: clinic_id, date, spend, leads, impressions, clicks, avg_cpl
- Join method: hybrid disjoint partition (Sebastian's canonical logic -- clinics with `clinic_label` in ad data use direct; ~11 unmapped clinics fall back to `ad_id` -> `contact_funnel` location join)

### Sync Jobs (Sidekiq)

**MetaDailySyncJob** -- runs 6:00 AM PT daily
- Fetches all accounts, campaigns, adsets, ads + insights for past 8 days
- Upserts into meta_insights, meta_campaigns, meta_adsets, meta_ads
- Refreshes materialized views after completion

**MetaIntradaySyncJob** -- runs every 15 min, 7 AM - 10 PM PT
- Fetches today's account + campaign insights only (not ad-level)
- Lightweight update for real-time spend monitoring

**MetaBackfillJob** -- one-time per account
- 37-month historical backfill
- Rate-limited via Sidekiq throttle

### Critical Data Rules

1. **Leads** MUST come from `actions` array where `action_type === "lead"`. NEVER use `onsite_conversion.lead_grouped`.
2. **CPL** MUST come from `cost_per_action_type` where `action_type === "lead"`. NEVER calculate as `spend / leads`.
3. **Pagination** on ad accounts requires cursor-based pagination (`paging.cursors.after`).
4. **All dates PT-bucketed**: `DATE(timestamp AT TIME ZONE 'America/Los_Angeles')`.
5. **Paid-only metrics**: all media reporting uses leads/evals where `ad_id IS NOT NULL`, except lead-source panel and ZIP map.

---

## 2. Data Servicing

### 2A. Media Buyer Command Center (`/mb`)

**Current source:** Sebastian's `build_mb.py` (Python -> BigQuery -> static HTML)
**Target:** React page in Flywheel ops app, backed by Rails API

**Layout:**
- Top: RAG summary tiles (RED/AMBER/GREEN clinic count for selected pod)
- Filter bar: date range, pod multi-select chips, search
- Main table: one row per clinic

**Per-clinic columns:**

| Column | Source | Bands |
|--------|--------|-------|
| Clinic Name | clinics table | -- |
| Pod | pod_clinics | -- |
| Health | Clinic Health RAG formula | RED/AMBER/GREEN |
| Spend | mv_ad_performance_daily | -- |
| Leads (paid) | contact_funnel (ad_id IS NOT NULL) | -- |
| CPL | spend / paid_leads | <=20 green, 20-35 amber, >35 red |
| Evals (attributable) | contact_funnel.is_evaluation, created-date cohort, cancelled-excluded | -- |
| CPE | spend / attributable_evals | <=270 green, 270-380 amber, >380 red |
| L->E | leads / evals (COUNT, not %; lower=better) | -- |
| Show Rate | showed / resolved (resolved = SHOWED + NO SHOW + CX BY PATIENT + CX BY CLINIC) | -- |
| Monthly Goal | eval target from clinic_retainer/client_master | -- |
| MTD Evals | evaluations by scheduled_date, current month | -- |
| Pace % | MTD / (goal * day_fraction) | -- |
| Out-of-Geo % | AVG(is_nq_geo) over leads with known geo flag | -- |
| Eval Source | breakdown of eval origins (paid vs organic vs google) | -- |

**Clinic Health RAG formula (locked):**
```
day_frac = day_of_month / days_in_month (PT)
RED:   week_pct_to_goal < 0.70 AND month_pct_to_goal < day_frac
AMBER: week_pct_to_goal < 0.70 OR  month_pct_to_goal < day_frac
GREEN: neither
```

**Interactions:**
- Click clinic row -> expand to campaign-level sub-table
- RAG chip click -> re-sorts that tier to top
- Map with bullseye per clinic (Mapbox) -- must center on CLINIC location, not HQ (bug fix from Sebastian's punchlist)

**API endpoints:**
```
GET /api/v1/media/command-center
  ?date_start=2026-06-01&date_end=2026-06-30
  &pod_ids[]=pod_alpha&pod_ids[]=pod_beta
  Response: { clinics: [...], rag_summary: { red: 5, amber: 12, green: 30 } }
```

### 2B. Meta Ads Drill-Down

**Current source:** Dashboard production (Maryam's React app)
**Target:** Migrate existing React components into Flywheel

**Hierarchy:** Accounts -> Campaigns -> Adsets -> Ads
Each level shows: name, status, spend, impressions, clicks, leads, CPL, CTR

**Per level, display:**
- KPI summary cards (total spend, leads, avg CPL, active count)
- Sortable/paginated table
- Date range picker (L7D / L14D / L28D / custom)

**Ad-level features:**
- Ad preview modal (lazy-loaded via `GET /{ad_id}/previews?ad_format=DESKTOP_FEED_STANDARD`)
- Winning ads view with spend-tiered health scoring + minimum lead filter (1-10 dropdown)
- Rejection status badge (currently rejected vs previously-rejected-now-active)

**API endpoints (existing, migrate):**
```
GET /api/v1/meta/accounts
GET /api/v1/meta/campaigns/:accountId
GET /api/v1/meta/adsets/:campaignId
GET /api/v1/meta/ads/:adsetId
GET /api/v1/meta/insights/:objectId
GET /api/v1/meta/winning-ads
GET /api/v1/meta/ad-preview/:adId
GET /api/v1/meta/insights-compare  (current vs prior period)
```

### 2C. Top Ads Intelligence (`/topads`)

**Current source:** Sebastian's topads pipeline (Python -> BQ -> static HTML)
**Target:** React page in Flywheel

**Features:**
- Ads ranked by geo + insurance performance
- Creative launch scores (composite: CPE shrink, funnel depth, geo match, recency)
- Rejection-aware scoring (rejection_penalty, n_rejections)
- Three tabs: Active, Launch, Rejected
- Per-ad: thumbnail, ad name, spend, leads, CPL, CPE, flight dates, active dot
- Rejected tab: currently-rejected ads with review_status lifecycle

### 2D. Acquisition Cockpit (`/agency`)

**Current source:** Sebastian's `build_agency.py`
**Target:** React page in Flywheel

**Features:**
- Cross-clinic acquisition overview
- Multi-select pod filter chips
- Aggregate KPIs: total spend, total leads, total evals, avg CPL, avg CPE
- Per-clinic breakdown table
- Trend charts (spend/leads/evals over time)

### 2E. Geo Heatmap

**Current source:** Dashboard production (Mapbox) + Sebastian's `build_heatmap.py`
**Target:** Unified React page

**Features:**
- Lead ZIP code visualization (Census ZCTA boundaries)
- Geo-targeting cities/zips for targeting heatmap
- Per-clinic drill-down by geography
- Clinic locations from Facebook Pages API
- Color-scaled metrics (recalibrate to realistic averages of best-performing clinics)
- Pod view filtering

### 2F. Eval Pricing Calculator

**Current source:** Sebastian's `calculator.html` (static, self-contained)
**Target:** React page in Flywheel

**What it does:** Platform pricing model for per-eval pricing
**Inputs:** State/market, payer mix (tier A/B/C checkboxes), market exclusivity, gross margin slider
**Outputs:** Recommended price/eval, geo base CPE, payer cost index, acceptance rate, effective cost, deliverable evals/mo, cost/approval, price/showed eval
**Data:** Trailing 12-month Meta-attributed data ($728K spend / 3,402 evals across 20 states). Payers ranked by cost-per-approved-patient.

**Data needs:** This calculator's state-level data and payer tier costs should be computed from live Flywheel data rather than hardcoded, refreshed monthly.

---

## 3. Workflow

### 3A. Ad Rejection Monitoring
- **Trigger:** MetaStatusSyncJob detects ad status change to DISAPPROVED
- **Action:** Create `meta_ad_rejections` record, fire Slack notification to media buyer channel
- **UI:** Rejection badge on ad in `/mb` and `/topads`; distinct icon for "currently rejected" vs "previously rejected but now active"

### 3B. Spend Alerts
- **Trigger:** Daily spend for a clinic exceeds budget threshold (from clinic settings)
- **Action:** Slack notification to assigned media buyer + account manager
- **Future:** Auto-pause campaigns via Meta API (requires careful guardrails)

### 3C. Clinic Health Alerts
- **Trigger:** Clinic health RAG changes from GREEN to RED
- **Action:** Slack notification to pod lead + media buyer
- **Data:** Uses the locked RAG formula above

### 3D. Weekly Meta Performance Report
- **Trigger:** Scheduled job, Monday 8 AM PT
- **Action:** Generate per-clinic performance summary, post to Slack or send via email
- **Current source:** Sebastian's `weekly-meta-report` service

---

## Appendix: Metric Definitions (Locked)

These definitions are from Sebastian's canonical `METRICS_DICTIONARY.md` and MUST be used exactly:

| Metric | Formula | Notes |
|--------|---------|-------|
| CPL | `ad_spend / paid_leads` | Paid = ad_id IS NOT NULL. Leads=0 -> show "--" |
| CPE | `ad_spend / attributable_evals` | Attributable = created-date cohort, cancelled-excluded, paid |
| L->E | `leads / evals` | A COUNT, not %. Lower = better. Evals=0 -> "--" |
| Show Rate | `showed / resolved` | Resolved = SHOWED + NO SHOW + CX BY PATIENT + CX BY CLINIC. UPCOMING excluded |
| Out-of-Geo % | `AVG(is_nq_geo)` over leads with known geo flag | Denominator excludes leads where is_nq_geo IS NULL |
| Investment | Agency: `retainer + ad_spend`; Bundled: `flat_fee` (forward from transition month) | |
| ROI | `modeled_revenue / investment` as lo~hi range | Assumption-based, flagged as such |
