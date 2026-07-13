# PRD 05 -- Clinic-Facing

**Department Lead:** Catherine Jee
**Users:** External (clinic) users -- providers, billing, managers, owners across ~51 clinics. NOT Psycle internal users.
**Clinic user types:** `provider`, `billing`, `manager`, `owner` (each with `clinic_admin` or `clinic_team` role)
**Source projects:** Dashboard production (patient funnel pages, settings), Dashboard prototype (onboarding), Flywheel (scheduling, EMR integration)
**Key constraint:** Clinic users CANNOT see top-of-funnel Meta metrics (impressions, clicks, spend, CPL)

---

## 1. Data Schema

### Tables Required (mostly shared with PRDs 02-03, clinic-scoped views)

**`clinics`** (enriched -- merges Dashboard + Internal + Flywheel)
- id (TEXT PK = GHL location ID)
- org_id (FK -> organizations)
- name
- slug
- address, city, state, zip
- phone, email, website
- timezone (default America/Denver)
- clinic_npi (facility-level NPI, distinct from provider NPI)
- settings (JSONB -- Slack channel IDs, treatment modalities, screening questions, capacity)
- emr_type (osmind, tebra, athena, drchrono, advanced_md, intakeq, other)
- is_active
- onboarding_completed_at
- created_at, updated_at

**`organizations`** (client = multi-location group)
- id (TEXT PK, UUID)
- name
- slug
- settings (JSONB)
- plan (starter, premium -- for future marketplace billing)
- is_active
- created_at, updated_at

**Key distinction (from Jun 26 Clinics Tables meeting):**
- `organizations.id` = client/org level (e.g., "Headlight")
- `clinics.id` = location level (e.g., "Headlight CA", "Headlight WA")
- These MUST be separate -- the old schema conflated them, blocking multi-location aggregation

**`providers`** (evaluating providers, per-clinic)
- id
- clinic_id (FK)
- user_id (FK -> users, nullable -- provider may also be a user)
- name
- email
- npi_number (10-digit)
- provider_type (MD/DO~Psychiatry/Anesthesiology/Neurology/Pain Management/Emergency Medicine, PMHNP, NP, CRNA, PA, Other)
- bio
- availability (TEXT -- e.g., "M-W: 9-3, F: 11-3")
- is_active
- created_at, updated_at

**`provider_locations`** (which locations a provider serves)
- provider_id (FK)
- clinic_id (FK)
- PRIMARY KEY (provider_id, clinic_id)

**`provider_insurances`** (insurances are provider-scoped, per Jun 26 meeting)
- id
- provider_id (FK)
- insurance_carrier
- insurance_plan
- is_in_network (boolean)
- created_at

**`clinic_custom_fields`** (see PRD 03 for schema)
**`clinic_custom_field_values`** (see PRD 03 for schema)

**`users`** (unified auth -- see PRD 06 for full schema)
- Clinic-relevant fields: role (clinic_admin, clinic_team), user_type (provider, billing, manager, owner), provider_type, npi_number
- user_clinics junction table controls which clinics a user can access

---

## 2. Data Servicing

### 2A. Clinic Dashboard (home view when clinic user logs in)

**Not yet built -- new for Flywheel**

**KPI Cards (visible to clinic users -- NO spend/CPL data):**
- Evaluations This Month (count)
- Show Rate (showed / resolved)
- Insurance Submissions (count)
- Treatments Started (count)

**Quick Actions:**
- View upcoming evaluations
- View pending PAs
- View active treatments

### 2B. Patient Funnel Pages (Clinic-Scoped View)

Same pages as PRD 03 (Evaluations, Insurances, PAs, Treatments) but with clinic user restrictions:

**Access rules (clinic users are always external, never Psycle employees):**

| Section | Clinic Admin | Clinic Team | Psycle AM (for reference) | Psycle CC (for reference) |
|---------|-------------|-------------|--------------------------|--------------------------|
| Ads | Hidden | Hidden | Hidden | Hidden |
| Leads | Hidden | Hidden | Pod clinics, view | Pod clinics, view+edit |
| Calls | Hidden | Hidden | Pod clinics, view | Pod clinics, view |
| Consults | Hidden | Hidden | Pod clinics, view | Pod clinics, view |
| Evaluations | Own clinic, edit | Own clinic, view | Pod clinics, edit | Pod clinics, view |
| Insurances | Own clinic, edit | Own clinic, view | Pod clinics, edit | Pod clinics, view |
| Prior Auths | Own clinic, edit | Own clinic, view | Pod clinics, edit | Hidden |
| Treatments | Own clinic, edit | Own clinic, view | Pod clinics, edit | Hidden |
| Settings | Own clinic | Hidden | Pod clinics | Hidden |

**Edit capabilities (clinic_admin only):**
- Update evaluation_status, qualification_status
- Update PA status, dates, sessions
- Update treatment status, sessions completed
- Add/edit custom field values
- All edits logged in audit_logs

**Custom fields:**
- Columns from `clinic_custom_fields` appear as additional table columns
- Clinic admins can add new custom fields (with approval workflow or self-serve TBD)
- Custom fields do NOT sync to GHL -- they exist only in Flywheel

### 2C. Clinic Settings

**Accessible by:** Psycle Admin, Psycle Team, Clinic Admin

**Clinic Details tab:**
- Name, NPI, address, phone, timezone
- GHL Location ID (read-only)
- GHL Sync Status badge (synced / not synced / syncing / error)
- Last Sync timestamp
- Treatment modalities multi-select (Spravato, Ketamine, TMS, Med Mgmt)

**Preferred Location Field Mapping:**
- Dropdown populated from clinic's GHL custom fields via API
- Maps which GHL field = "Preferred Location"
- When mapped, value appears as column on all data tables
- READ-ONLY from GHL -- never create this field
- If unmapped, clinic is single-location

**Integration Settings:**
- Meta Ad Account ID (text, `act_XXXXXXXXX`)
- Slack External Channel ID (Psycle channel)
- Slack Internal Channel ID (Staff channel)
- EMR connection status (from Flywheel's EmrConnection model)

**Insurance Details tab:**
- Per-provider insurance acceptance list
- Carrier name normalization (via insurance_carrier_map)

**Team tab:**
- List of users with access to this clinic
- Invite new user (creates user + sends email)
- Role assignment (clinic_admin, clinic_team)
- User type assignment (provider, billing, manager, owner)
- Permission toggles per section (gradient-backed toggle buttons)
- Only Psycle Admin can edit permission toggles

### 2D. Contact Detail View

**Current source:** Dashboard production (tabbed interface)
**Target:** Migrate to Flywheel

**Per-contact view (click a row in any table):**
- Contact info: name, phone, email, source, created date
- Timeline: all events (calls, messages, appointments, status changes) in chronological order
- Tabs: Overview, Evaluations, Insurance, PAs, Treatments, Notes
- Each tab shows the relevant records for this contact
- Edit inline (clinic_admin) or view-only (clinic_team)

---

## 3. Workflow

### 3A. Clinic User Invitation
- **Trigger:** Psycle Admin or Clinic Admin adds a user via Team settings
- **Action:** Create user record, assign clinic access via user_clinics, send invitation email with login link
- **Auth:** Clinic users use email/password login (may not have Google accounts); Psycle team uses Google OAuth
- **Default permissions:** Based on user_type selection (provider, billing, manager, owner -- each has preset permission dispositions, overridable by Psycle Admin)

### 3B. Custom Field Creation
- **Trigger:** Clinic admin requests a new field (via settings or inline on table)
- **Action:** Create clinic_custom_fields record, column appears on relevant table for that clinic only
- **Scope:** Custom fields are per-clinic -- they don't appear for other clinics
- **Types supported:** text, dropdown (with configurable options), date, number, boolean
- **No GHL sync:** Custom fields exist only in Flywheel

### 3C. Eval Data Import from Sheets
- **Trigger:** Account manager uploads a clinic's historical eval tracker Sheet
- **Action:** Parse Sheet -> validate -> map fields -> upsert into evaluations table
- **Matching:** Match contacts by ghl_contact_id where possible; create orphan records where not
- **Use case:** Onboarding clinics that have years of eval data in Google Sheets

### 3D. Scheduling Integration (from Flywheel P1)
- **Trigger:** Coordinator books an eval via Flywheel scheduler
- **Action:** Creates appointment in EMR (IntakeQ etc.) via adapter -> creates evaluation record in Flywheel -> optional GHL writeback
- **Confirmation call backstop:** If scheduling was dumb-write (no native availability), CC confirms via phone and updates status
- **Status sync:** IntakeQ webhook updates appointment status -> Flywheel updates evaluation record

### 3E. Connective Health Data Population (from Flywheel P2)
- **Trigger:** CH delivers FHIR bundle for a patient at this clinic
- **Action:** Ingest bundle -> extract medication history, diagnoses, insurance -> populate contact record -> map to EMR fields -> write to IntakeQ
- **Qualification:** >= 2 antidepressant trials detected -> mark as qualified
- **Feature-flagged:** Behind a flip; Keragon remains as fallback

### 3F. Auto-Logout
- **Trigger:** 30 minutes of inactivity
- **Action:** Session invalidated, user redirected to login
- **HIPAA requirement:** Prevents unattended access to PHI
