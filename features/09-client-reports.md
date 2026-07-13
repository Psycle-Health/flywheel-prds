# Feature 09 -- Client Reports & Leaderboard

**Dependencies:** 05-contact-funnel, 06-patient-funnel
**Source:** Psycle-Data (`/client-report`, client_master, clinic_retainer), Dashboard prod (leaderboard), Internal app (retainers)
**Users:** Admin (all), AM (pod clinics), Clinic Admin/Team (own clinic report only)

---

## Schema (additional tables)

```sql
CREATE TABLE clinic_retainers (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  clinic_id TEXT NOT NULL REFERENCES clinics(id),
  month DATE NOT NULL,                    -- first of month
  retainer_amount NUMERIC(12,2),
  contract_type TEXT DEFAULT 'agency',    -- agency, platform
  ad_budget_target NUMERIC(12,2),
  eval_target INTEGER,
  is_bundled BOOLEAN DEFAULT false,
  bundled_transition_date DATE,           -- bundled pricing applies from this date forward
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(clinic_id, month)
);
```

### Materialized View

```sql
-- mv_client_monthly: the one-stop mart for per-clinic per-month rollup
CREATE MATERIALIZED VIEW mv_client_monthly AS
SELECT
  c.id AS clinic_id,
  c.name AS clinic_name,
  DATE_TRUNC('month', d.date) AS month,
  p.name AS pod_name,

  -- Spend
  SUM(d.spend) AS spend,
  SUM(d.leads) AS leads,
  CASE WHEN SUM(d.leads) > 0 THEN SUM(d.spend) / SUM(d.leads) ELSE NULL END AS cpl,

  -- Evals (from evaluations, scheduled_date, CX excluded)
  COUNT(DISTINCT e.id) FILTER (WHERE e.evaluation_status NOT IN ('CX by Clinic','CX by Patient')) AS evals,
  COUNT(DISTINCT e.id) FILTER (WHERE e.evaluation_status = 'Showed') AS evals_showed,
  COUNT(DISTINCT e.id) FILTER (WHERE en.is_conversion = 1) AS conversions,

  -- Show rate
  CASE WHEN COUNT(DISTINCT e.id) FILTER (WHERE e.evaluation_status IN ('Showed','No Show','CX by Clinic','CX by Patient')) > 0
    THEN COUNT(DISTINCT e.id) FILTER (WHERE e.evaluation_status = 'Showed')::numeric /
         COUNT(DISTINCT e.id) FILTER (WHERE e.evaluation_status IN ('Showed','No Show','CX by Clinic','CX by Patient'))
    ELSE NULL END AS show_rate,

  -- Retainer + investment
  cr.retainer_amount,
  cr.eval_target,
  cr.contract_type,
  CASE
    WHEN cr.is_bundled AND d.date >= cr.bundled_transition_date THEN cr.retainer_amount
    ELSE COALESCE(cr.retainer_amount, 0) + SUM(d.spend)
  END AS investment

FROM clinics c
LEFT JOIN pod_clinics pc ON pc.clinic_id = c.id
LEFT JOIN pods p ON p.id = pc.pod_id
LEFT JOIN mv_clinic_spend_daily d ON d.clinic_id = c.id
LEFT JOIN evaluations e ON e.clinic_id = c.id
  AND DATE_TRUNC('month', e.scheduled_date) = DATE_TRUNC('month', d.date)
LEFT JOIN mv_evaluations_norm en ON en.id = e.id
LEFT JOIN clinic_retainers cr ON cr.clinic_id = c.id
  AND cr.month = DATE_TRUNC('month', d.date)
WHERE c.is_active = true
GROUP BY c.id, c.name, DATE_TRUNC('month', d.date), p.name,
         cr.retainer_amount, cr.eval_target, cr.contract_type, cr.is_bundled, cr.bundled_transition_date;
```

## Client Performance Report

**One page per clinic. Accessible by:** Admin, AM (pod clinics), Clinic Admin/Team (own clinic -- no spend/CPL data shown to clinic users)

### KPI Cards
- Investment (retainer + ad_spend; or flat_fee for bundled)
- Total Evals
- Show Rate
- Conversions
- ROI Range (lo~hi, assumption-based, flagged as such)

### Funnel Waterfall
- Leads -> Insurance Submitted -> In-Network -> Consults -> Evals -> Showed -> Qualified -> PA Approved -> Treatment Started
- Conversion rate between each adjacent stage
- Cost per result at each stage (hidden from clinic users)

### Monthly Trend Table
One row per month: spend, leads, CPL, evals, show rate, conversions, investment, ROI range

### ROI Model (Assumption-Based)
```
Revenue (modeled):
  SPV: per_session × 32 sessions (per_session options: $175, $300, $450)
  TMS: per_session × 36 sessions (per_session options: $200, $300, $400)

ROI = modeled_revenue / investment, shown as lo~hi range
```
**Always flagged as assumption, never presented as actuals.**

### API
```
GET /api/v1/reports/client/:clinicId?date_start&date_end
  Response: { kpis, funnel_waterfall, monthly_trend, roi }

GET /api/v1/reports/client/:clinicId/pdf   -- generate downloadable PDF (future)
```

## Leaderboard

**Expanded per Jul 6 meeting to include PA + treatment data.**

### Columns

| Column | Source | Visible To |
|--------|--------|-----------|
| Clinic Name | clinics | All |
| Pod | pod_clinics | Internal only |
| AM | pod_members (AM type) | Internal only |
| Evals (MTD) | evaluations, current month | All |
| Eval Goal | clinic_retainers.eval_target | All |
| Pace % | MTD / (goal * day_fraction) | All |
| Show Rate | showed / resolved | All |
| PA Rate | PA approved / total PAs | All |
| Started Treatments | treatments where tx_status contains 'Conversion' or 'Scheduled' | All |
| PA Lead Time Rank | Percentile vs all clinics (avg days from eval to PA approval) | Internal only |
| Treatment Conv Rate | conversions / showed_evals, by modality | Internal only |
| CPL | spend / paid_leads | Internal only (hidden from clinic) |
| CPE | spend / attributable_evals | Internal only (hidden from clinic) |

### Access
- **Admin:** Full leaderboard, all clinics, all columns
- **Growth:** Full leaderboard, read-only
- **AM:** Full leaderboard (can compare their pod vs others)
- **CC:** Hidden
- **Clinic users:** Own row only, no spend/cost columns

### API
```
GET /api/v1/reports/leaderboard?date_start&date_end&pod_ids[]&sort_by&order
```

## Workflows

- **Leaderboard auto-refresh:** Updates on every materialized view refresh cycle
- **Client report delivery:** Weekly auto-generated summary emailed to clinic admin + assigned AM (future)
- **PA lead time ranking:** Computed nightly; if a clinic falls to bottom 20th percentile, Slack notification to AM for client conversation
