# PRD 02 -- Care Coordination

**Department Lead:** Aliyah Taylor / Jessica Alcaraz (CC leads)
**Users:** CC user type -- Rachel Rubenstein (CC Manager/QC), ~15 care coordinators across 4 pods (Glutamates, Heavyweights, Catalyst, Pea Pod)
**Psycle user type:** `cc`
**Source projects:** Psycle-Data (`/cc`, `cc_daily`), Dashboard (leads/calls/consults pages, GHL sync)
**Design principle:** Build as a "Rachel command center" from her operational QC needs first (Jul 6 meeting decision)

---

## 1. Data Schema

### Tables Required

**`ghl_contacts`** (existing in Dashboard -- migrate)
- id
- ghl_contact_id (UNIQUE)
- clinic_id (FK -> clinics)
- location_id
- raw_payload (JSONB -- full GHL contact payload)
- sync_source (backfill, webhook, manual)
- first_synced_at, last_synced_at
- created_at, updated_at

**`ghl_appointments`** (existing -- migrate)
- id
- ghl_appointment_id (UNIQUE)
- ghl_contact_id (FK)
- clinic_id (FK)
- calendar_id
- appointment_status (scheduled, showed, no_show, cancelled, rescheduled)
- start_time, end_time
- booked_by (CC name -- used for CC attribution)
- raw_payload (JSONB)
- created_at, updated_at

**`ghl_calls`** (existing -- migrate)
- id
- ghl_contact_id (FK)
- clinic_id (FK)
- call_direction (inbound, outbound)
- call_status (completed, missed, voicemail, no_answer)
- call_duration_seconds
- call_user_id (GHL user = the CC who made/received the call)
- call_user_name
- call_date (DATE, PT-bucketed)
- raw_payload (JSONB)
- created_at, updated_at

**`ghl_messages`** (for SMS/email activity tracking)
- id
- ghl_contact_id (FK)
- clinic_id (FK)
- message_type (SMS, EMAIL)
- direction (inbound, outbound)
- user_id (CC who sent)
- user_name
- event_timestamp
- created_at

**`cc_alias_map`** (maps GHL user names to canonical CC names)
- id
- raw_name (TEXT -- the name as it appears in GHL data, e.g. "johnny", "John S.")
- canonical_name (TEXT -- "John Smith")
- user_id (FK -> users, nullable)
- clinic_id (FK, nullable -- some aliases are clinic-specific)
- created_at

**`field_mappings`** (existing -- per-clinic GHL field -> app field mapping)
- id
- clinic_id (FK)
- data_source_type (ghl, emr)
- app_field (e.g. 'evaluation_status', 'insurance_status')
- app_table (e.g. 'evaluations', 'insurance')
- source_field_id (GHL custom field ID)
- source_field_key, source_field_name
- source (native, custom)
- is_active
- created_at, updated_at
- UNIQUE(clinic_id, data_source_type, app_field)

### Materialized Views

**`mv_contact_funnel`** (replaces Sebastian's `ghl.contact_funnel`)
- Grain: 1 row per contact
- Columns (all INT 0/1 flags):
  - is_lead, is_pre_qualified, is_insurance_submitted, is_in_network
  - is_consult_scheduled, is_consult_showed
  - is_evaluation, is_evaluation_showed
  - is_conversion, is_nq_geo, is_nq_ins
- Plus: ad_id (Meta attribution), source_normalized, postal_code, clinic_id, pod, created_date (PT)
- Rules: use `SUM(is_x)` for counts, `AVG(is_x)` for rates. Never COUNTIF.

**`mv_cc_daily`** (replaces Sebastian's `ghl.cc_daily`)
- Grain: 1 row per (CC name, date)
- Columns:
  - cc_name (canonical, via cc_alias_map)
  - date (PT-bucketed)
  - dials (COUNTIF message_type = 'CALL' per actor)
  - connects (completed calls >= 30s inbound / >= 60s outbound)
  - talk_time_seconds
  - evals_booked (COUNTIF is_evaluation=1, grouped by booked_by-resolved CC)
  - shows, no_shows, cx_by_patient, cx_by_clinic (resolved eval outcomes)
  - show_rate (shows / resolved, where resolved = shows + no_shows + cx_patient + cx_clinic)
  - sms_out, sms_in, emails_out
  - consults_scheduled, consults_showed
- CC Attribution rule (LOCKED): attribute to the **actor** (who made the call / booked the eval), NEVER `assigned_to`. Differs in ~52% of contacts.

### Sync Jobs (Sidekiq)

**GhlContactSyncJob** -- webhook-triggered (real-time) + nightly reconciliation (2 AM PT)
- Webhook: receives ContactUpdate, fetches full contact via GHL API, upserts into ghl_contacts
- Nightly: queries all contacts with updatedAt > last_sync, re-fetches and upserts
- Rate limited via Bottleneck (8 concurrent)

**GhlAppointmentSyncJob** -- nightly (2 AM PT)
- Fetches appointments per clinic, upserts into ghl_appointments

**GhlCallSyncJob** -- hourly
- Fetches conversations/calls per clinic, upserts into ghl_calls

**GhlMessageSyncJob** -- hourly
- Fetches SMS/email events per clinic, upserts into ghl_messages

**MaterializedViewRefreshJob** -- after each sync cycle
- Refreshes mv_contact_funnel, mv_cc_daily

---

## 2. Data Servicing

### 2A. CC Performance Dashboard (Rachel's Command Center)

**Current source:** Sebastian's `build_cc_report.py` + `cc_daily` view
**Target:** React page in Flywheel, designed around Rachel's QC workflow

**Layout (two-tiered, per Jul 6 meeting):**

**Tier 1 -- Overview (10 KPI tiles with color-coded health):**

| Metric | Definition | Health Threshold |
|--------|-----------|-----------------|
| Calls Made | Total dials today | 60-100/day baseline (adjust if consults scheduled) |
| Connection Rate | Connects / dials | TBD by Rachel |
| Connections | Completed calls >= duration threshold | -- |
| Talk Time | Total call duration | TBD |
| Consults Scheduled | Appointments booked | -- |
| Consults Showed | Showed / scheduled | TBD |
| Evals Booked | Evaluations attributed to CC | -- |
| Show Rate | Showed / resolved (locked formula) | TBD |
| Insurance Collected | VOBs collected by CC | -- |
| SMS Sent | Outbound SMS count | TBD |

Color coding: GREEN (on target), AMBER (needs review), RED (concern)

**Tier 2 -- Per-CC Detail Table:**
- One row per care coordinator
- All Tier 1 metrics as columns
- Sortable, filterable by pod
- Click row -> drill-down to that CC's daily activity log
- Date range filter

**Tier 3 -- Individual CC Drill-Down:**
- Daily activity timeline
- Call-by-call log with duration, direction, outcome
- Evals booked with show/no-show outcomes
- Trend charts (calls/day, show rate over time)

**API endpoints:**
```
GET /api/v1/cc/dashboard
  ?date_start&date_end&pod_ids[]
  Response: { kpis: {...}, coordinators: [...] }

GET /api/v1/cc/:ccName/activity
  ?date_start&date_end
  Response: { daily_metrics: [...], calls: [...], evals: [...] }
```

### 2B. Leads Page

**Current source:** Dashboard production
**Target:** Migrate React components

**KPI Cards:** Total Leads, Followed Up, Pending, Consults Booked
**Table columns:** Contact name, phone, created date, followup status, followup date/time, consultation date/time, consultation status, clinic name, location, source
**Features:** Sortable, paginated (200 max/page), searchable, date-filtered, clinic-switcher scoped

**API:** `GET /api/v1/clinic/:clinicId/leads`

### 2C. Calls Page

**Current source:** Dashboard production
**Target:** Migrate React components

**KPI Cards:** Total Calls, Inbound, Outbound, Avg Duration
**Table columns:** Contact name, phone, direction, user (CC), disposition, date/time, duration
**Features:** Same standard table features

**API:** `GET /api/v1/clinic/:clinicId/calls`

### 2D. Consults Page

**Current source:** Dashboard production (stub)
**Target:** Build out fully

**KPI Cards:** Scheduled, Showed, No Show, Show Rate
**Table columns:** Contact name, appointment date/time, calendar, status (scheduled/showed/no_show/cancelled/rescheduled), booked_by, source
**Features:** Same standard table features

**API:** `GET /api/v1/clinic/:clinicId/consults`

---

## 3. Workflow

### 3A. GHL Webhook Processing
- **Trigger:** GHL sends ContactUpdate webhook
- **Action:** Fetch full contact via API -> upsert ghl_contacts -> refresh affected materialized views
- **Non-fatal:** if appointment or call sync fails, contact data is preserved
- **DLQ:** Failed webhooks queued for retry (3 attempts with exponential backoff)

### 3B. CC Health Alerts
- **Trigger:** CC daily metrics fall below health thresholds (defined per metric by Rachel)
- **Action:** Automated Slack notification to Rachel with CC name + which metrics are red
- **Conditional logic:** "If calls < 60 BUT consults_scheduled > 3, then not a concern" (the if/then/but rules from Jul 6 meeting)

### 3C. Show Rate Degradation Alert
- **Trigger:** A clinic's 7-day rolling show rate drops below threshold
- **Action:** Slack notification to assigned account manager + CC manager
- **Rationale:** Show rate reflects conversation quality + CRM hygiene; early warning prevents eval waste

### 3D. Speed-to-Lead Monitoring
- **Trigger:** New lead created in GHL (webhook)
- **Action:** Start 5-minute timer. If no call logged by a CC within 5 minutes, escalate via Slack notification
- **Future:** Integrate with lead scoring to prioritize which leads trigger urgent alerts

### 3E. Eval Tracker Sheet Sync (transitional)
- **Trigger:** Scheduled job, every 15 minutes
- **Action:** Sync 44 clinic Eval Tracker Google Sheets -> evaluations table (MERGE/upsert)
- **Authentication:** Google Sheets API with Domain-Wide Delegation
- **PHI handling:** patient_name and notes are encrypted at rest; never logged
- **Sunset plan:** As clinics adopt Flywheel for eval tracking, remove individual sheets from sync; retain as fallback during transition
