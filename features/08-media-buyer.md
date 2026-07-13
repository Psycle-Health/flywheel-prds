# Feature 08 -- Media Buyer Command Center

**Dependencies:** 04-meta-ads, 05-contact-funnel
**Source:** Psycle-Data (`/mb`, `/agency`, `/topads`, `/heatmap`, `calculator.html`, METRICS_DICTIONARY)
**Users:** Admin, Growth

This feature consolidates Sebastian's 5 intelligence dashboards into Flywheel React pages.

---

## 8A. Media Buyer Command Center (`/mb`)

**Source:** Sebastian's `build_mb.py` (~2000 lines Python -> BQ -> static HTML)

### Layout
- **Top:** RAG summary tiles (RED/AMBER/GREEN clinic count, filtered by selected pods)
- **Filter bar:** Date range, pod multi-select chips, search
- **Main table:** One row per clinic with all key metrics

### Per-Clinic Columns

| Column | Source | Bands |
|--------|--------|-------|
| Clinic | clinics | -- |
| Pod | pod_clinics | -- |
| Health | RAG formula | RED/AMBER/GREEN chips (clickable, re-sorts tier to top) |
| Spend | mv_clinic_spend_daily | -- |
| Leads (paid) | mv_contact_funnel (ad_id IS NOT NULL) | -- |
| CPL | spend / paid_leads | <=20 green, 20-35 amber, >35 red |
| Evals | mv_contact_funnel.is_evaluation (created-date cohort, includes all with submission date) | -- |
| CPE | spend / attributable_evals | <=270 green, 270-380 amber, >380 red |
| L->E | leads / evals (COUNT, lower=better) | -- |
| Show Rate | showed / resolved | -- |
| Goal | clinic_retainers.eval_target | -- |
| MTD | evaluations by scheduled_date, current month | -- |
| Pace % | MTD / (goal * day_of_month / days_in_month) | -- |
| OOG % | AVG(is_nq_geo) over leads with known flag | -- |
| Eval Source | Breakdown: paid / organic / google | -- |
| Rejected Ads | Count of currently-rejected ads for this clinic | Icon: red X for active rejection |

### Interactions
- Click clinic row -> campaign-level sub-table
- Click RAG chip -> sort that tier to top
- Map panel with bullseye per clinic (Mapbox) centered on CLINIC address (not HQ)
- Rejected ad icon: red X = currently rejected; yellow flag = previously rejected, now active

### API
```
GET /api/v1/media/command-center?date_start&date_end&pod_ids[]&search
```

---

## 8B. Acquisition Cockpit (`/agency`)

**Source:** Sebastian's `build_agency.py`

Cross-clinic acquisition overview for leadership + growth team.

- Multi-select pod filter chips
- Aggregate KPIs: Total Spend, Total Leads, Total Evals, Avg CPL, Avg CPE
- Per-clinic breakdown table (spend, leads, evals, CPL, CPE)
- Trend charts (spend/leads/evals over time, filterable by pod)

### API
```
GET /api/v1/media/acquisition?date_start&date_end&pod_ids[]
```

---

## 8C. Top Ads (`/topads`)

**Source:** Sebastian's topads pipeline (build_topads.py, creative_launch_scores, ad_economics, rejected_ads)

### Three tabs:

**Active:** Best-performing ads ranked by geo + insurance performance
- Thumbnail, ad name, account, spend, leads, CPL, CPE, flight dates
- Active status dot (based on last_active_date)
- Creative launch scores (composite: CPE shrink, funnel depth, geo match, recency)

**Launch:** New/recent ads with launch scoring
- Rejection-aware scoring (rejection_penalty, n_rejections)
- launch_score = base + geo_bonus + funnel_bonus + recency_bonus - rejection_penalty

**Rejected:** Currently-rejected ads with review_status lifecycle
- Ad name, rejection reason, detected_at, current status
- Distinct from "previously rejected but now active" (shown in Active tab with yellow flag)

### API
```
GET /api/v1/media/top-ads?tab=active|launch|rejected&date_start&date_end&pod_ids[]
```

---

## 8D. Heatmap / AM Cockpit

**Source:** Sebastian's `build_heatmap.py` (2,052 lines) + Dashboard prod (Mapbox integration)

Geographic performance visualization for account managers.

- Mapbox map with ZIP-level data (Census ZCTA boundaries)
- Color-scaled metrics by ZIP: lead count, eval count, in-network rate, CPL
- Per-clinic drill-down (click clinic pin -> see its geographic footprint)
- Pod view filtering (multi-select chips)
- Clinic locations from Facebook Pages API

**Color scale calibration:** Base on realistic averages of best-performing clinics + general market (Sebastian's punchlist note: current scales need recalibration)

### API
```
GET /api/v1/media/heatmap?pod_ids[]&metric=leads|evals|cpl|in_network_rate
GET /api/v1/meta/lead-zips?clinicId
GET /api/v1/meta/zip-boundaries
```

---

## 8E. Eval Pricing Calculator

**Source:** Sebastian's `calculator.html` (self-contained static page)

Platform pricing model: given a clinic's state + payer mix + exclusivity + margin -> recommended per-eval price.

### Inputs
- State/market dropdown (20 states with trailing 12-month data)
- Payer checkboxes (tiered A/B/C by cost-per-approved-patient): Anthem, Medicaid, Cigna, Medicare, Aetna, UHC, BCBS, Tricare, VA, Cash/Self-Pay
- Quick-select buttons: All, Clear, Tier A only, Commercial only, Govt only
- Market exclusivity toggle: Exclusive / Shared
- Gross margin slider: 0-120%, default 55%

### Outputs
- Recommended price per eval (large display)
- Geo base CPE (state blended)
- Payer cost index (selected mix)
- Acceptance rate (usable supply)
- Effective cost / deliverable eval
- Deliverable evals/month
- Cost per approval
- Price per showed eval
- Formula breakdown

### Data
Currently hardcoded from trailing 12-month Meta-attributed data ($728K / 3,402 evals). In Flywheel, compute from live data refreshed monthly via a Sidekiq job that aggregates mv_ad_performance_daily + mv_contact_funnel by state.

### API
```
GET /api/v1/media/calculator-data   -- state-level aggregates + payer tiers
```

---

## 8F. Insurance Intelligence (Staging)

**Source:** Sebastian's `build_insurance.py` + `build_insurance_marketplace.py`

- Carrier analysis across clinics (in-network rates by carrier)
- Insurance marketplace: payer supply/demand matching
- `insurance_carrier_map`: editable mapping table for carrier name normalization (carriers have many variant names)

### API
```
GET /api/v1/media/insurance-intelligence?pod_ids[]
GET /api/v1/media/insurance-marketplace
GET /api/v1/insurance-carrier-map          -- editable mapping
PUT /api/v1/insurance-carrier-map/:id
```

---

## Workflows

- **Ad Rejection Alert:** Meta status change to DISAPPROVED -> Slack notification to growth channel
- **Spend Threshold Alert:** Daily spend exceeds budget -> Slack to assigned growth + AM
- **Clinic Health Change:** RAG goes GREEN -> RED -> Slack to pod lead + growth
- **Weekly Performance Report:** Monday 8 AM PT, auto-generated per-clinic summary (from Sebastian's weekly-meta-report service)
- **Calculator Data Refresh:** Monthly Sidekiq job recomputes state-level CPE + payer tier costs from live data
