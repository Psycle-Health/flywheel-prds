# Feature 05 -- Contact Funnel & Analytics Views

**Dependencies:** 03-ghl-sync, 04-meta-ads
**Source:** Psycle-Data (contact_funnel, evaluations_norm, cc_daily, METRICS_DICTIONARY.md)
**Users:** All dashboards and reports consume these views. No direct UI -- this is the analytics engine.

This is the most critical feature for data integrity. Every metric, dashboard, and report reads from these views. Sebastian's `METRICS_DICTIONARY.md` is the canonical source for all formulas.

---

## Schema

### Flywheel-Owned Tables (SOT for qualification onward; evals sync from GHL)

```sql
-- Evaluations (eval status/date/time/scheduled_by synced FROM GHL; qualification onward is Flywheel-owned)
CREATE TABLE evaluations (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  ghl_contact_id TEXT REFERENCES ghl_contacts(ghl_contact_id),
  clinic_id TEXT NOT NULL REFERENCES clinics(id),
  patient_name TEXT,                      -- AES-256-GCM encrypted
  scheduled_date DATE,
  evaluation_time TIME,
  date_of_evaluation DATE,
  appointment_type TEXT,
  treatment_modality TEXT,                -- Spravato, TMS, Ketamine, Med Mgmt, Multiple
  evaluation_status TEXT,                 -- Upcoming, Showed, No Show, Rescheduled, CX by Clinic, CX by Patient
  qualification_status TEXT,              -- Qualified, NQ~Insurance, NQ~Geo, NQ~Medical, NQ~Financial, NQ~Other
  scheduled_by TEXT,
  source TEXT,
  insurance_plan TEXT,
  preferred_location TEXT,
  notes_encrypted TEXT,                   -- AES-256-GCM
  sheet_source_id TEXT,                   -- tracks Google Sheet origin during transition
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE vob (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  ghl_contact_id TEXT REFERENCES ghl_contacts(ghl_contact_id),
  clinic_id TEXT NOT NULL REFERENCES clinics(id),
  patient_name TEXT,                      -- encrypted
  dob TEXT,                               -- encrypted
  collected_by TEXT,
  submission_date DATE,
  p_insurance_name TEXT, p_insurance_id TEXT, p_insurance_plan_name TEXT,
  s_insurance_name TEXT, s_insurance_id TEXT, s_insurance_plan_name TEXT,
  network_status TEXT,                    -- In Network, Out of Network, OON (w/ Benefits),
                                          -- Insufficient Information, Waitlist, Request Insurance
  cpt_codes TEXT,
  pa_needed BOOLEAN,
  referral_needed BOOLEAN,
  spravato_benefit_type TEXT,
  copay_coinsurance TEXT,
  final_patient_responsibility TEXT,
  insurance_comments TEXT,                -- encrypted
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE prior_authorizations (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  ghl_contact_id TEXT REFERENCES ghl_contacts(ghl_contact_id),
  clinic_id TEXT NOT NULL REFERENCES clinics(id),
  patient_name TEXT,                      -- encrypted
  pa_status TEXT,                         -- Approved, Not Needed, Cash Pay, Submitted,
                                          -- Pending Submission, Med Clearance Req, Pending~Admin,
                                          -- Appealed, PA Denied, Appeal Denied, RX Sent, N/A
  pa_submission_date_time TIMESTAMPTZ,
  provider TEXT,
  buy_bill_pharmacy TEXT,
  p_insurance_carrier TEXT, p_insurance_plan_type TEXT,
  s_insurance_carrier TEXT, s_insurance_plan_type TEXT,
  pa_paperwork_url TEXT,
  pa_receipt_date DATE, pa_expiration_date DATE,
  approved_sessions INTEGER,
  notes_encrypted TEXT,                   -- encrypted
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE treatments (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  ghl_contact_id TEXT REFERENCES ghl_contacts(ghl_contact_id),
  clinic_id TEXT NOT NULL REFERENCES clinics(id),
  patient_name TEXT,                      -- encrypted
  tx_modality TEXT,                       -- Spravato, Ketamine, TMS, Med Mgmt
  tx_status TEXT,                         -- Scheduled, Spravato/Ketamine/TMS/Med Mgmt Conversion,
                                          -- Pending, Non Responsive, Lost~Denied/Cost/Other,
                                          -- Stopped Tx, Psycle F/U, Clinic F/U, N/A
  p_insurance_type TEXT, s_insurance_type TEXT,
  tx_start_date DATE,
  tx_approved BOOLEAN,
  num_completed INTEGER DEFAULT 0,
  num_treatments INTEGER,
  cpt_codes TEXT,
  progress_notes_req_by_payer BOOLEAN,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Materialized Views (the analytics engine)

**`mv_evaluations_norm`** -- Port of Sebastian's `ghl.evaluations_norm`
Normalizes 34+ messy status variants into canonical boolean flags. This is the SINGLE semantic layer for eval outcomes. Never re-derive status regexes per query.

```sql
CREATE MATERIALIZED VIEW mv_evaluations_norm AS
SELECT
  e.*,
  -- Eval outcome flags
  (evaluation_status = 'Showed')::int AS is_showed,
  (evaluation_status = 'No Show')::int AS is_no_show,
  (evaluation_status IN ('CX by Clinic', 'CX by Patient'))::int AS is_cancelled,

  -- Conversion flags (from treatment_status)
  (tx_status ~* 'conversion')::int AS is_conversion,
  (tx_status ~* 'schedul|conversion')::int AS is_tx_scheduled,

  -- Treatment outcome
  CASE
    WHEN tx_status ~* 'conversion' THEN 'converted'
    WHEN tx_status ~* 'schedul' THEN 'scheduled'
    WHEN tx_status ~* 'lost|denied|cost|other' THEN 'lost'
    WHEN tx_status ~* 'pending|non responsive' THEN 'pending'
    WHEN tx_status IS NULL OR tx_status = '' THEN 'none'
    ELSE 'other'
  END AS tx_outcome,

  -- Qualification
  (qualification_status ~* 'qualif' AND qualification_status !~* 'not|nq')::int AS is_qualified,

  -- PA flags
  (pa.pa_status ~* 'approv|pend|submit|deni')::int AS is_pa_pending,
  (pa.pa_status ~* 'approv' AND pa.pa_status !~* 'pend')::int AS is_pa_approved

FROM evaluations e
LEFT JOIN prior_authorizations pa ON pa.ghl_contact_id = e.ghl_contact_id
  AND pa.clinic_id = e.clinic_id;
```

**`mv_contact_funnel`** -- Port of Sebastian's `ghl.contact_funnel`
Per-lead funnel flags with Meta attribution + geo. All flags are INT 0/1.

```sql
CREATE MATERIALIZED VIEW mv_contact_funnel AS
SELECT
  gc.ghl_contact_id,
  gc.clinic_id,
  -- Date bucketed in PT (CRITICAL: prevents month-boundary bugs)
  DATE(gc.raw_payload->>'dateAdded' AT TIME ZONE 'America/Los_Angeles') AS created_date,

  -- Funnel flags (all INT 0/1; use SUM for counts, AVG for rates)
  1 AS is_lead,
  CASE WHEN gc.raw_payload->>'preQualified' IS NOT NULL THEN 1 ELSE 0 END AS is_pre_qualified,
  CASE WHEN v.submission_date IS NOT NULL THEN 1 ELSE 0 END AS is_insurance_submitted,
  CASE WHEN v.network_status = 'In Network' THEN 1 ELSE 0 END AS is_in_network,
  CASE WHEN ga.appointment_status = 'scheduled' AND ga.calendar_id IN (...) THEN 1 ELSE 0 END AS is_consult_scheduled,
  CASE WHEN ga.appointment_status = 'showed' AND ga.calendar_id IN (...) THEN 1 ELSE 0 END AS is_consult_showed,
  CASE WHEN e.id IS NOT NULL THEN 1 ELSE 0 END AS is_evaluation,  -- includes ALL evals with submission date (incl. cancellations)
  CASE WHEN en.is_showed = 1 THEN 1 ELSE 0 END AS is_evaluation_showed,
  CASE WHEN en.is_conversion = 1 THEN 1 ELSE 0 END AS is_conversion,

  -- DQ flags
  CASE WHEN gc.raw_payload->>'nqGeo' = 'true' THEN 1 ELSE 0 END AS is_nq_geo,
  CASE WHEN gc.raw_payload->>'nqIns' = 'true' THEN 1 ELSE 0 END AS is_nq_ins,

  -- Meta attribution
  mpa.ad_id,
  gc.raw_payload->>'source' AS source_raw,
  -- ... source_normalized logic

  -- Geo
  gc.raw_payload->>'postalCode' AS postal_code,
  pc.name AS pod

FROM ghl_contacts gc
LEFT JOIN vob v ON v.ghl_contact_id = gc.ghl_contact_id
LEFT JOIN ghl_appointments ga ON ga.ghl_contact_id = gc.ghl_contact_id
LEFT JOIN evaluations e ON e.ghl_contact_id = gc.ghl_contact_id
LEFT JOIN mv_evaluations_norm en ON en.ghl_contact_id = gc.ghl_contact_id
LEFT JOIN mv_ad_performance_daily mpa ON ... -- attribution join
LEFT JOIN pod_clinics pc ON pc.clinic_id = gc.clinic_id;
```

**`mv_cc_daily`** -- Port of Sebastian's `ghl.cc_daily`
Per-CC per-day activity metrics. Source of record for CC performance.

```sql
CREATE MATERIALIZED VIEW mv_cc_daily AS
SELECT
  cam.canonical_name AS cc_name,
  DATE(c.call_date AT TIME ZONE 'America/Los_Angeles') AS date,

  -- Dials
  COUNT(*) FILTER (WHERE c.id IS NOT NULL) AS dials,

  -- Connects (LOCKED: >=30s inbound, >=60s outbound)
  COUNT(*) FILTER (WHERE c.call_status = 'completed' AND (
    (c.call_direction = 'inbound' AND c.call_duration_seconds >= 30) OR
    (c.call_direction = 'outbound' AND c.call_duration_seconds >= 60) OR
    (c.call_direction IS NULL AND c.call_duration_seconds >= 30)
  )) AS connects,

  SUM(c.call_duration_seconds) FILTER (WHERE c.call_status = 'completed') AS talk_time_seconds,

  -- Evals booked (by booked_by, resolved via cc_alias_map)
  COUNT(DISTINCT e.id) FILTER (WHERE e.id IS NOT NULL) AS evals_booked,

  -- Show rate components
  COUNT(*) FILTER (WHERE e.evaluation_status = 'Showed') AS shows,
  COUNT(*) FILTER (WHERE e.evaluation_status = 'No Show') AS no_shows,
  COUNT(*) FILTER (WHERE e.evaluation_status IN ('CX by Clinic','CX by Patient')) AS cancellations,

  -- Messaging
  COUNT(*) FILTER (WHERE m.message_type = 'SMS' AND m.direction = 'outbound') AS sms_out,
  COUNT(*) FILTER (WHERE m.message_type = 'SMS' AND m.direction = 'inbound') AS sms_in,
  COUNT(*) FILTER (WHERE m.message_type = 'EMAIL' AND m.direction = 'outbound') AS emails_out

FROM ghl_calls c
LEFT JOIN cc_alias_map cam ON cam.raw_name = c.call_user_name
LEFT JOIN evaluations e ON e.scheduled_by = cam.canonical_name
  AND DATE(e.scheduled_date) = DATE(c.call_date AT TIME ZONE 'America/Los_Angeles')
LEFT JOIN ghl_messages m ON m.user_name = c.call_user_name
  AND DATE(m.event_timestamp AT TIME ZONE 'America/Los_Angeles') = DATE(c.call_date AT TIME ZONE 'America/Los_Angeles')
GROUP BY cam.canonical_name, DATE(c.call_date AT TIME ZONE 'America/Los_Angeles');
```

**`mv_clinic_spend_daily`** -- Clinic-level spend rollup

Uses Sebastian's hybrid disjoint partition join:
- PRIMARY: clinics present in `ad_account_mappings` use mapped ad_account_id + campaign_filter
- FALLBACK: ~11 unmapped clinics join via `ad_id` -> `mv_contact_funnel` location
- Spend with NULL clinic attribution reported separately, never folded into a clinic

### Refresh Schedule

All materialized views refresh via `MaterializedViewRefreshJob` (Sidekiq):
- After each GHL sync cycle
- After each Meta sync cycle
- On-demand via admin trigger

## Eval Tracker Sheet Sync (Transitional)

**Job:** EvalTrackerSyncJob (Sidekiq cron, every 15 min)
**Source:** 44 clinic Google Sheets (eval trackers) -> `evaluations` table
**Auth:** Google Sheets API with Domain-Wide Delegation
**PHI:** patient_name and notes encrypted before storage; never logged
**Matching:** Maps sheet rows to `ghl_contact_id` where possible; creates records where not
**Sunset:** As clinics adopt Flywheel UI for eval tracking, remove sheets from sync one by one

## Locked Metric Definitions

From Sebastian's METRICS_DICTIONARY.md -- these MUST be used exactly:

| Metric | Formula | Bands |
|--------|---------|-------|
| CPL | `spend / paid_leads` (paid = ad_id IS NOT NULL) | <=20 green, 20-35 amber, >35 red |
| CPE | `spend / attributable_evals` (created-date cohort, includes all evals with submission date) | <=270 green, 270-380 amber, >380 red |
| L->E | `leads / evals` (COUNT not %; lower=better) | -- |
| Show Rate | `showed / resolved` (resolved = showed + no_show + CX_patient + CX_clinic) | -- |
| Connect | Completed call >= 30s inbound / >= 60s outbound | -- |
| CC Attribution | Actor (caller/booker), NEVER assigned_to | -- |
| Investment | Agency: retainer + ad_spend. Bundled: flat_fee (forward from transition month) | -- |
| Out-of-Geo % | AVG(is_nq_geo) over leads WITH a known geo flag (NULL denominator excluded) | -- |
| Clinic Health RAG | RED: week<0.70 AND month<day_frac. AMBER: either. GREEN: neither | -- |

**Eval counting rule:** Include ALL evaluations with an evaluation submission date/time (including CX BY PATIENT / CX BY CLINIC). Cancellations are part of the eval count AND the show-rate denominator.

**GHL owns eval data:** Eval status, date, time, and scheduled_by sync FROM GHL. Flywheel is SOT from qualification onward.

**Three eval date bases (always state which):**
1. Eval-tracker `scheduled_date` (canonical/operational -- headlines, goals)
2. Contact `created_date` cohort (for CPE/L->E so spend/leads/evals share one cohort)
3. Eval `submission_date` (secondary reconciliation)
