# PRD 03 -- Account Management

**Department Lead:** Jake Inzalaca (AM Lead)
**Users:** AM user type -- 3 Account Managers: Kyla Davidson (Heavyweights), Cody Shimokochi (Catalyst), Kerrie Clark (Pea Pod). Jake covers Glutamates + oversees all AMs.
**Psycle user type:** `am`
**Source projects:** Psycle-Data (`/heatmap`, `/client-report`, `/insurance`, evaluations_norm), Dashboard (patient funnel pages, clinic settings)

---

## 1. Data Schema

### Tables Required

**`evaluations`** (Flywheel-owned, SOT for post-eval data)
- id
- ghl_contact_id (FK -> ghl_contacts, the universal join key)
- clinic_id (FK -> clinics)
- patient_name (AES-256-GCM encrypted)
- scheduled_date (DATE)
- evaluation_time (TIME)
- date_of_evaluation (DATE)
- appointment_type
- treatment_modality (Spravato, TMS, Ketamine, Med Mgmt, Multiple)
- evaluation_status (Upcoming, Showed, No Show, Rescheduled, CX by Clinic, CX by Patient)
- qualification_status (Qualified, NQ~Insurance, NQ~Geo, NQ~Medical, NQ~Financial, NQ~Other)
- scheduled_by
- source
- insurance_plan
- preferred_location (from GHL field mapping, if mapped)
- notes_encrypted (AES-256-GCM)
- sheet_source_id (nullable -- tracks which Google Sheet this came from during transition)
- created_at, updated_at

**`vob`** (verification of benefits -- Flywheel-owned for post-eval, GHL-synced for intake)
- id
- ghl_contact_id (FK)
- clinic_id (FK)
- patient_name (encrypted)
- dob (encrypted)
- collected_by
- submission_date
- p_insurance_name, p_insurance_id (encrypted), p_insurance_plan_name
- s_insurance_name, s_insurance_id (encrypted), s_insurance_plan_name
- network_status (In Network, Out of Network, OON w/ Benefits, Insufficient Information, Waitlist, Request Insurance)
- cpt_codes
- pa_needed (boolean)
- referral_needed (boolean)
- spravato_benefit_type
- copay_coinsurance
- final_patient_responsibility
- insurance_comments (encrypted)
- created_at, updated_at

**`prior_authorizations`** (Flywheel-owned)
- id
- ghl_contact_id (FK)
- clinic_id (FK)
- patient_name (encrypted)
- pa_status (Approved, Not Needed, Cash Pay, Submitted, Pending Submission, Med Clearance Req, Pending~Admin, Appealed, PA Denied, Appeal Denied, RX Sent, N/A)
- pa_submission_date_time
- provider
- buy_bill_pharmacy
- p_insurance_carrier, p_insurance_plan_type
- s_insurance_carrier, s_insurance_plan_type
- pa_paperwork_url
- pa_receipt_date, pa_expiration_date
- approved_sessions
- notes_encrypted (AES-256-GCM)
- created_at, updated_at

**`treatments`** (Flywheel-owned)
- id
- ghl_contact_id (FK)
- clinic_id (FK)
- patient_name (encrypted)
- tx_modality (Spravato, Ketamine, TMS, Med Mgmt)
- tx_status (Scheduled, Spravato/Ketamine/TMS/Med Mgmt Conversion, Pending, Non Responsive, Lost~Denied/Cost/Other, Stopped Tx, Psycle F/U, Clinic F/U, N/A)
- p_insurance_type, s_insurance_type
- tx_start_date
- tx_approved (boolean)
- num_completed, num_treatments
- cpt_codes
- progress_notes_req_by_payer
- created_at, updated_at

**`clinic_custom_fields`** (clinics can add their own columns)
- id
- clinic_id (FK)
- field_name
- field_type (text, dropdown, date, number, boolean)
- dropdown_options (JSONB, nullable)
- display_order
- table_target (evaluations, prior_authorizations, treatments)
- is_active
- created_at, updated_at

**`clinic_custom_field_values`**
- id
- custom_field_id (FK)
- record_id (the evaluation/PA/treatment record ID)
- record_type (evaluations, prior_authorizations, treatments)
- value (TEXT)
- created_at, updated_at

### Materialized Views

**`mv_evaluations_norm`** (replaces Sebastian's `ghl.evaluations_norm`)
- Grain: 1 row per (sheet_source_id or internal_id, contact, eval_date)
- Normalizes 34 messy status variants into canonical flags:
  - is_showed, is_no_show, is_cancelled
  - is_conversion, is_tx_scheduled
  - tx_outcome (none, converted, scheduled, lost, pending, other)
  - tx_modality (Spravato, TMS, Ketamine, Med Mgmt)
  - eval_outcome
  - is_qualified, is_pa_pending, is_pa_approved
- This is the MOST IMPORTANT view -- all eval-derived metrics read from here

**`mv_client_monthly`** (the intended one-stop mart)
- Grain: 1 row per (clinic_id, month)
- Columns: clinic_id, month, spend, leads, evals, evals_showed, conversions, retainer, investment, cpl, cpe, show_rate, roi_lo, roi_hi

---

## 2. Data Servicing

### 2A. Account Manager Cockpit / Heatmap (`/heatmap`)

**Current source:** Sebastian's `build_heatmap.py` (2,052 lines)
**Target:** React page with Mapbox integration

**Features:**
- Geographic heatmap of leads/evals by ZIP (Mapbox + Census ZCTA boundaries)
- Color-scaled metrics (calibrate to realistic averages of best-performing clinics)
- Per-clinic drill-down: click clinic -> see its leads/evals/spend geographically
- Pod view filtering (multi-select pod chips)
- ZIP-level data: lead count, eval count, in-network rate, CPL per ZIP

**Known bugs to fix (from Sebastian's punchlist):**
- Pod views show 0 for all metrics -- likely pod-label mismatch in JS filter
- Color scales need recalibration to realistic baselines

### 2B. Client Performance Report (`/client-report`)

**Current source:** Sebastian's `build_client_report.py`
**Target:** React page

**Per-client view (one page per clinic, accessible by account manager + clinic admin):**

**KPI Cards:**
- Investment (retainer + ad_spend, or flat_fee for bundled)
- Total Evals
- Show Rate
- Conversions
- ROI Range (lo~hi, assumption-based, flagged as such)

**Funnel Waterfall:**
- Leads -> Insurance Submitted -> In-Network -> Consults -> Evals -> Showed -> Qualified -> PA -> Treatment
- Conversion rate between each stage
- Cost per result at each stage

**Monthly Trend Table:**
- One row per month
- Columns: spend, leads, CPL, evals, show rate, conversions, investment, ROI range

**Data rules:**
- Investment: agency model = `retainer + ad_spend`; bundled/platform = flat_fee (forward from transition month only)
- ROI revenue model: SPV = per_session(175/300/450) x 32 sessions; TMS = per_session(200/300/400) x 36 sessions. Shown as lo~hi range.
- ROI is an assumption range, NEVER presented as actuals

### 2C. Leaderboard

**Current source:** Dashboard production (basic) + Internal app (enriched)
**Target:** Unified leaderboard in Flywheel

**Columns (expanded per Jul 6 meeting):**
- Clinic name, pod, account manager
- Evals (MTD, goal, pace %)
- Show rate
- **PA rate** (new -- approved / total PAs)
- **Started treatments** (new)
- **PA lead time ranking** (new -- percentile vs all other clients, e.g. "bottom 80th percentile")
- Treatment conversion rate by modality
- CPL, CPE (agency view only -- hidden from clinic users)

**Access:**
- Admin: full leaderboard, all clinics
- Growth: full leaderboard, all clinics (read-only)
- AM: full leaderboard, all clinics (can compare their pod vs others)
- CC: hidden (CCs use CC dashboard instead)
- Clinic users: own row only

### 2D. Patient Funnel Pages (Evaluations, Insurances, PAs, Treatments)

**Current source:** Dashboard production (7 data pages)
**Target:** Migrate React components, wire to Flywheel-owned tables

All follow the canonical pattern:
- Gradient-wrapped KPI cards at top
- Sortable/paginated/searchable data table below
- Sticky glassmorphism filter bar
- Clinic switcher scoping
- Date range picker (L7D / L14D / L28D / custom)
- Status badges with per-theme color coding (see Dashboard PRD for full color map)

**New Patients (Evaluations) page:**
- KPI Cards: Total Evals, Showed, Qualified, Show Rate
- Table: patient name, eval date/time, appointment type, modality (badge), eval status (badge), qualification status (badge), scheduled_by, source, insurance plan, preferred location
- **Custom fields from clinic_custom_fields appear as additional columns**

**Insurances page:**
- KPI Cards: Total Submissions, In-Network, OON, In-Network Rate
- Table: patient name, DOB, submission date, collected_by, primary/secondary insurance, network status (badge), PA needed, referral needed, copay, final patient responsibility, spravato benefit type, CPT codes, comments

**Prior Auths page:**
- KPI Cards: Approved, Pending, Denied, Approval Rate
- Table: patient name, PA status (badge), submission date, provider, buy & bill/pharmacy, insurances, paperwork URL, receipt/expiration dates, approved sessions, notes

**Treatments page:**
- KPI Cards: Active, Completed, Conversions, Completion Rate
- Table: patient name, modality, tx status (badge), insurances, start date, approved (Y/N), completed/total sessions, CPT codes, progress notes required

### 2E. Insurance Intelligence

**Current source:** Sebastian's `build_insurance.py` + `build_insurance_marketplace.py`
**Target:** React pages

**Insurance Intelligence:**
- Carrier analysis across clinics
- In-network rates by carrier
- `insurance_carrier_map` (editable mapping table for carrier name normalization)

**Insurance Marketplace:**
- Payer supply/demand matching
- Which clinics accept which carriers
- Geographic coverage analysis

---

## 3. Workflow

### 3A. Source-of-Truth Handoff (GHL -> Flywheel)

The critical architecture for this department:

```
GHL (source of truth)              Flywheel (source of truth)
----------------------------       ---------------------------------
Lead creation                  ->  Eval status, date, time
Insurance submission           ->  Qualification status
In-network determination       ->  PA status, submission, approval
Consult scheduling/show        ->  Treatment status, modality, sessions
                                   Custom clinic fields
                                   Anything imported from Sheets
```

- **Sync direction for evals onward:** Flywheel -> GHL (optional writeback, not mandatory)
- **Clinics edit eval/qualification/PA/treatment data in Flywheel**, not GHL
- **Custom fields** (e.g., Delex's PA columns) live only in Flywheel via `clinic_custom_fields`

**Who can edit what:**

| Data | Admin | AM | CC | Clinic Admin | Clinic Team |
|------|-------|----|----|-------------|-------------|
| Eval status/qualification | Full | Own pod clinics | Own pod clinics | Own clinic | View only |
| PA status | Full | Own pod clinics | No | Own clinic | View only |
| Treatment status | Full | Own pod clinics | No | Own clinic | View only |
| Custom fields | Full | Own pod clinics | No | Own clinic | View only |

### 3B. Eval Status Update Propagation
- **Trigger:** Account manager or clinic user updates eval status in Flywheel
- **Action:** Update evaluations table -> refresh mv_evaluations_norm -> optional GHL writeback (behind feature flag)
- **Audit:** Log every status change in audit_logs with user identity + timestamp

### 3C. PA Expiration Alerts
- **Trigger:** Daily job checks prior_authorizations.pa_expiration_date
- **Action:** 30-day, 14-day, 7-day warnings via Slack + in-app notification to assigned account manager
- **Display:** PA expiration badge in the PAs table

### 3D. Treatment Milestone Tracking
- **Trigger:** num_completed reaches 50%, 75%, 100% of num_treatments
- **Action:** In-app notification to clinic + account manager
- **Future:** Automated re-authorization workflow when sessions near expiration

### 3E. Google Sheet Import (transitional + ongoing)
- **Trigger:** Account manager uploads CSV/Sheet data for a clinic
- **Action:** Parse, validate (Zod), map clinic_name -> clinic_id, upsert into evaluations/PAs/treatments
- **Validation:** Required fields enforced, status values validated against enums, dates parsed (ISO 8601 or MM/DD/YYYY)
- **Error handling:** Upload log with row counts + per-row errors, surfaced in UI
- **Use case:** Clinics with historical Sheet data that hasn't been in GHL; ongoing for clinics that add custom columns not synced from GHL
