# Feature 06 -- Patient Funnel Pages

**Dependencies:** 03-ghl-sync, 05-contact-funnel
**Source:** Dashboard prod (7 data pages, canonical New Patients design), Psycle-Data (evaluations_norm)
**Users:** Admin (all clinics), AM (pod clinics), CC (leads/calls/consults only), Clinic Admin/Team (own clinic, evals onward)

All 7 pages follow the same canonical pattern established by the New Patients page. Build New Patients first, then replicate.

---

## Shared UI Pattern (apply to ALL pages)

1. **KPI summary cards** (4 per page) -- gradient-wrapped (1.5px border), large value (32px, weight 700, tabular-nums), uppercase label, optional delta indicator (green up / red down vs prior period)
2. **Data table** -- wrapped in `card-gradient-wrap`, sortable columns, pagination (200 max/page), search input, date range filter (respects toolbar picker)
3. **Filter bar** -- sticky, glassmorphism (`backdrop-filter: blur(16px)`), contains search + column visibility toggle
4. **Clinic switcher** -- scopes all API calls. Internal users see pod clinics; clinic users see own clinic(s).
5. **Status badges** -- color-coded per status enum, per theme (Light/Dark/Colorful). Colors from Dashboard prod design system.
6. **Custom field columns** -- clinics with `clinic_custom_fields` see additional columns appended to their table view
7. **Row click** -> contact detail view (tabbed: Overview, Evals, Insurance, PAs, Treatments, Notes)

## Pages

### 1. New Patients (Evaluations) -- CANONICAL REFERENCE

**Visibility:** Admin, AM, CC (view), Clinic Admin (edit), Clinic Team (view)
**API:** `GET /api/v1/clinic/:clinicId/evaluations?page&limit&search&startDate&endDate`
**Data source:** `evaluations` table (Flywheel-owned) + `mv_evaluations_norm` for flags

**KPI Cards:** Total Evals | Showed | Qualified | Show Rate

**Columns:**
| Column | Source | Type |
|--------|--------|------|
| Patient Name | evaluations.patient_name (decrypted) | Text |
| Eval Date | evaluations.date_of_evaluation | Date |
| Eval Time | evaluations.evaluation_time | Time |
| Appointment Type | evaluations.appointment_type | Text |
| Treatment Modality | evaluations.treatment_modality | Badge |
| Eval Status | evaluations.evaluation_status | Badge |
| Qualification Status | evaluations.qualification_status | Badge |
| Scheduled By | evaluations.scheduled_by | Text |
| Source | evaluations.source | Text |
| Insurance Plan | evaluations.insurance_plan | Text |
| Preferred Location | evaluations.preferred_location | Text (only if mapped) |
| *Custom fields* | clinic_custom_field_values | Per-clinic |

**Eval data (status, date, time, scheduled_by):** Synced from GHL, not directly editable in Flywheel.
**Editable by:** Admin, AM (pod clinics), Clinic Admin (own clinic). CC and Clinic Team = view-only.
**Editable fields:** qualification_status, treatment_modality, notes, custom field values. All edits logged in audit_logs.

### 2. Insurances (VOB)

**Visibility:** Admin, AM, CC (view), Clinic Admin (edit), Clinic Team (view)
**API:** `GET /api/v1/clinic/:clinicId/insurance?page&limit&search&startDate&endDate`
**Data source:** `vob` table

**KPI Cards:** Total Submissions | In-Network | Out-of-Network | In-Network Rate

**Columns:** Patient Name, DOB, Submission Date, Collected By, Primary Insurance (name + plan), Secondary Insurance, Network Status (badge), PA Needed (Y/N badge), Referral Needed, Copay/Coinsurance, Final Patient Responsibility, Spravato Benefit Type, CPT Codes, Insurance Comments (truncated, expandable), Preferred Location

### 3. Prior Authorizations

**Visibility:** Admin, AM, Clinic Admin (edit), Clinic Team (view). CC = hidden.
**API:** `GET /api/v1/clinic/:clinicId/prior-auths?page&limit&search&startDate&endDate`
**Data source:** `prior_authorizations` table (Flywheel-owned)

**KPI Cards:** Approved | Pending | Denied | Approval Rate

**Columns:** Patient Name, PA Status (badge), Submission Date, Provider, Buy & Bill/Pharmacy, Primary/Secondary Insurance, Paperwork URL, Receipt Date, Expiration Date, Approved Sessions, Notes

### 4. Treatments

**Visibility:** Admin, AM, Clinic Admin (edit), Clinic Team (view). CC = hidden.
**API:** `GET /api/v1/clinic/:clinicId/treatments?page&limit&search&startDate&endDate`
**Data source:** `treatments` table (Flywheel-owned)

**KPI Cards:** Active | Completed | Conversions | Completion Rate

**Columns:** Patient Name, Modality (badge), Tx Status (badge), Primary/Secondary Insurance, Start Date, Approved (Y/N), Completed/Total Sessions, CPT Codes, Progress Notes Required

### 5. Leads

**Visibility:** Admin, Growth, AM, CC. Clinic users = hidden (top-of-funnel).
**API:** `GET /api/v1/clinic/:clinicId/leads?page&limit&search&startDate&endDate`
**Data source:** `ghl_contacts` with field mapping extraction

**KPI Cards:** Total Leads | Followed Up | Pending | Consults Booked

**Columns:** Contact Name, Phone, Created Date, Followup Status, Followup Date/Time, Consultation Date/Time, Consultation Status, Clinic Name, Location, Source

### 6. Calls

**Visibility:** Admin, AM, CC. Clinic users = hidden.
**API:** `GET /api/v1/clinic/:clinicId/calls?page&limit&search&startDate&endDate`
**Data source:** `ghl_calls` table

**KPI Cards:** Total Calls | Inbound | Outbound | Avg Duration

**Columns:** Contact Name, Phone, Direction, User (CC), Disposition, Date/Time, Duration

### 7. Consults

**Visibility:** Admin, AM, CC. Clinic users = hidden.
**API:** `GET /api/v1/clinic/:clinicId/consults?page&limit&search&startDate&endDate`
**Data source:** `ghl_appointments` (filtered by consultation calendars)

**KPI Cards:** Scheduled | Showed | No Show | Show Rate

**Columns:** Contact Name, Appointment Date/Time, Calendar, Status (badge), Booked By, Source

## Contact Detail View

Triggered by clicking any row in any table. Shows the full picture for one patient/contact.

**Header:** Name, phone, email, source, created date, clinic
**Timeline:** Chronological list of all events (calls, messages, appointments, status changes)
**Tabs:** Overview | Evaluations | Insurance | PAs | Treatments | Notes

Each tab shows the relevant records for this contact from the respective tables. Editable inline for users with edit permissions.

## Workflows

- **Edit -> audit:** Every field edit logs to `audit_logs` (user, timestamp, old_value -> new_value, entity)
- **Edit -> optional GHL writeback:** Behind feature flag. When enabled, status changes write back to GHL contact custom fields. Initially disabled.
- **Custom field columns:** Automatically appear per-clinic based on `clinic_custom_fields`. No code change needed to add a new custom field.
- **PA expiration alerts:** Daily job checks `pa_expiration_date`. 30/14/7-day warnings via Slack + in-app badge.
