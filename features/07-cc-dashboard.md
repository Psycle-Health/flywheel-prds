# Feature 07 -- CC Dashboard

**Dependencies:** 03-ghl-sync, 05-contact-funnel
**Source:** Psycle-Data (`/cc`, `cc_daily`, `cc_daily_METRICS.md`), Jul 6 "App Overview & Training" meeting
**Users:** Admin, CC (Rachel + all coordinators)
**Design principle:** "Rachel command center" -- built from her QC workflow, not technical assumptions. Iterate 10% per week.

---

## Data Source

`mv_cc_daily` materialized view (defined in Feature 05). No additional tables needed.

## API Endpoints

```
GET /api/v1/cc/dashboard
  ?date_start&date_end&pod_ids[]
  Response: {
    kpis: { total_dials, total_connects, avg_show_rate, ... },
    coordinators: [
      { cc_name, dials, connects, connection_rate, talk_time, consults_scheduled,
        consults_showed, evals_booked, shows, no_shows, show_rate, insurance_collected,
        sms_out, sms_in, emails_out }
    ]
  }

GET /api/v1/cc/:ccName/activity
  ?date_start&date_end
  Response: {
    daily_metrics: [...],   -- per-day breakdown
    calls: [...],           -- individual call records
    evals: [...]            -- evals booked by this CC
  }

GET /api/v1/cc/health-standards
  Response: { metrics: [{ name, baseline, conditional_rules }] }

PUT /api/v1/cc/health-standards   -- Admin only
  Body: { metrics: [...] }
```

## UI Layout

### Tier 1 -- Overview KPI Tiles (top of page)

10 KPI tiles with color-coded health (GREEN/AMBER/RED based on configurable thresholds):

| # | Metric | Definition | Baseline |
|---|--------|-----------|----------|
| 1 | Calls Made | Total dials | 60-100/day (adjust if consults scheduled) |
| 2 | Connection Rate | connects / dials | TBD by Rachel |
| 3 | Connections | Calls >= 30s inbound / >= 60s outbound | -- |
| 4 | Talk Time | Total call duration (hours:minutes) | TBD |
| 5 | Consults Scheduled | Appointments booked | -- |
| 6 | Consults Showed | Showed / scheduled consults | TBD |
| 7 | Evals Booked | Evaluations attributed to CC (by booked_by) | -- |
| 8 | Show Rate | showed / resolved (locked formula) | TBD |
| 9 | Insurance Collected | VOBs with collected_by = this CC | -- |
| 10 | SMS Sent | Outbound SMS count | TBD |

### Tier 2 -- Per-CC Table

One row per care coordinator. All 10 metrics as columns. Plus:
- Pod assignment
- Date range filter
- Sortable by any column
- Click row -> Tier 3 drill-down

### Tier 3 -- Individual CC Drill-Down

- Daily activity chart (calls/connects/evals over time)
- Call-by-call log: contact name, direction, duration, status, timestamp
- Evals booked: contact name, clinic, eval date, show/no-show outcome
- Trend line: show rate rolling 7-day average

## Health Standards (Configurable)

Admin can configure per-metric thresholds with conditional logic:

```json
{
  "calls_made": {
    "green": ">= 60",
    "red": "< 40",
    "conditionals": [
      { "if": "consults_scheduled >= 3", "then": "adjust_baseline_to: 30" }
    ]
  }
}
```

**If/then/but rules** (from Jul 6 meeting): "60-100 calls/day is baseline, but adjust if consultations were scheduled that day"

## Workflows

- **CC Health Alerts:** When daily metrics fall below RED thresholds, Slack notification to Rachel with CC name + which metrics are red
- **Show Rate Degradation:** 7-day rolling show rate drops below threshold -> Slack to Rachel + assigned AM
- **Speed-to-Lead:** New lead webhook -> 5-min timer. No call logged -> escalation notification.
- **Weekly CC Summary:** Monday 8 AM PT, auto-generated summary of prior week's CC performance posted to ops Slack channel
