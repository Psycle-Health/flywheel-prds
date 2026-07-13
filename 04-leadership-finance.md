# PRD 04 -- Leadership & Finance

**Department Lead:** Danny Costa
**Users:** Admin user type -- Danny Costa, Elijah Zoarski (Founders), + Ops leadership: Rachel Rubenstein, Leila Rishwain
**Psycle user type:** `admin`
**Source projects:** Internal app (revenue/expenses/contractors/efficiency), Psycle-Data (client_master, clinic_retainer), Dashboard (Accounts matcher, onboarding)

---

## 1. Data Schema

### Tables Required

**`internal_revenues`**
- id
- clinic_id (FK -> clinics)
- period_start (DATE), period_end (DATE)
- mrr (NUMERIC 12,2)
- contract_value (NUMERIC 12,2)
- payment_status (current, overdue, cancelled, paused)
- payment_date (DATE)
- source (csv, stripe, quickbooks)
- notes
- created_at, updated_at

**`internal_upsells`**
- id
- clinic_id (FK -> clinics)
- upsell_type (additional_location, premium_tier, add_on_service)
- value (NUMERIC 12,2)
- status (proposed, accepted, declined, active, churned)
- proposed_date, closed_date
- notes
- created_at, updated_at

**`internal_expenses`**
- id
- clinic_id (FK, nullable -- NULL = company-wide)
- category (payroll, software, advertising, office, travel, other)
- vendor
- description
- amount (NUMERIC 12,2)
- expense_date (DATE)
- is_recurring (boolean)
- recurrence_period (monthly, quarterly, annual)
- source (csv, quickbooks)
- notes
- created_at, updated_at

**`internal_contractors`**
- id
- name
- email
- role (designer, developer, consultant)
- rate_type (hourly, monthly, project)
- rate (NUMERIC 10,2)
- hours_per_month (NUMERIC 6,1)
- is_active
- start_date, end_date
- notes
- created_at, updated_at

**`internal_contractor_assignments`**
- id
- contractor_id (FK)
- clinic_id (FK, nullable -- NULL = company-wide)
- pod_id (FK, nullable)
- monthly_cost (NUMERIC 10,2)
- created_at

**`internal_csv_uploads`** (upload audit log)
- id
- uploaded_by (FK -> users)
- target_table (revenues, expenses, upsells, contractors)
- file_name
- rows_imported, rows_failed
- error_log (JSONB)
- status (processing, completed, failed, partial)
- created_at

**`clinic_retainers`** (replaces Sebastian's `ghl.clinic_retainer`)
- id
- clinic_id (FK)
- month (DATE -- first of month)
- retainer_amount (NUMERIC 12,2)
- contract_type (agency, platform)
- ad_budget_target (NUMERIC 12,2)
- eval_target (INTEGER)
- is_bundled (BOOLEAN)
- bundled_transition_date (DATE, nullable)
- created_at, updated_at

**`ad_account_mappings`** (the accounts matcher -- see PRD 01 for schema)

### Materialized Views

**`mv_clinic_efficiency`** (derived from revenue + expenses + ads)
- Grain: 1 row per (clinic_id, month)
- Columns: clinic_id, month, mrr, ad_spend, total_expense, contractor_cost, net_margin, margin_pct, cac, ltv_estimate

---

## 2. Data Servicing

### 2A. Revenue Dashboard

**KPI Cards:** Total MRR, MRR Growth %, Average Revenue per Clinic, Total Active Clients
**Table:** Per-clinic monthly revenue -- clinic name, pod, MRR, contract value, payment status, start date, contract type (agency/platform)
**Access:** Super admins see all; account managers see pod totals for other pods, clinic-level for own pod

### 2B. Upsells Dashboard

**KPI Cards:** Total Upsell Revenue, Upsell Conversion Rate, Average Upsell Value, Pipeline Value
**Table:** Per-clinic upsell tracking -- clinic, pod, upsell type, value, status, proposed/closed dates, notes

### 2C. Expenses Dashboard

**KPI Cards:** Total Monthly Expenses, Category Breakdown, Burn Rate, Net Margin
**Table:** Expense line items -- category, vendor, amount, date, recurring flag, notes
**Chart:** Category breakdown pie/donut chart

### 2D. Contractors Dashboard

**KPI Cards:** Total Contractor Spend, Active Contractors, Average Rate, Utilization
**Table:** Contractor records -- name, role, rate, hours/month, total cost, assigned clinics/pods

### 2E. Efficiency Dashboard

**KPI Cards:** Net Margin %, Revenue per Employee, CAC, LTV:CAC Ratio
**Table:** Per-clinic efficiency -- clinic, pod, revenue, ad spend, total cost, margin, ROI
**Note:** This section is derived -- no direct data input. Pulls from revenue + expenses + ads.

### 2F. Accounts Matcher (Settings > Accounts)

**Current source:** Dashboard prototype mockup (`mockup-accounts.html`)
**Target:** React page in Flywheel Settings

**Purpose:** Map the three-way relationship: Organization -> GHL Location -> Meta Ad Account -> Campaigns

**UI Pattern:**
- Org cards with expand/collapse toggle
- **Single-region orgs:** mapping fields directly on org row (GHL Location dropdown, Meta Ad Account dropdown, Campaigns multi-select, Result Type selector)
- **Multi-region orgs:** indented subaccount rows, each with own mapping
- Campaign pills (lavender tags from selected ad account's campaigns; empty = "All Campaigns")
- Result Type: Lead (Form) or View Content per subaccount

**Actions:**
- "+ Add Organization" (header CTA)
- "+ Subaccount" (per org -- converts to multi-region)
- Edit (org name), Rename / Remove (per subaccount)

**Dropdowns populated from:**
- GHL Location: GHL Agency API (read-only list of all agency locations)
- Meta Ad Account: Meta API (all active ad accounts under business manager)
- Campaigns: campaigns in the selected ad account

### 2G. Client Onboarding Wizard

**Current source:** Dashboard prototype (`onboarding.html`)
**Target:** React multi-step form in Flywheel

**5 steps with progress bar:**

**Step 1 -- General Info:**
- Primary contact (name, title, email, phone)
- Organization details (clinic name, phone, email, website)
- Locations (address, hours, email, services multi-select per location)
- Per-location capacity: chairs, sessions/day, days/week per modality (Spravato, TMS)
- "+ Add Location" for multi-site clinics

**Step 2 -- Business & Compliance:**
- Business registration (name as declared in EIN, address, Tax ID, registration type)
- CP 575 upload (IRS confirmation of EIN)
- BAA with scrollable legal text, signature pad, practice name, signer name/title/date, checkbox acknowledgment

**Step 3 -- Staff & Providers:**
- Staff members (name, email, phone, role: Office Manager/Front Desk/Billing Admin/PA Admin/Clinic Admin)
- Evaluating providers (name, email, NPI, provider type, bio, availability)

**Step 4 -- EMR, Policies & Costs:**
- EMR selection (Osmind, Tebra, Athena, DrChrono, Advanced MD, IntakeQ, Other)
- Appointment reminders, eval locations, intake paperwork method
- Eval cost, cash-pay financing/pricing, deposit policies, cancellation policy
- File uploads: clinic paperwork + consents

**Step 5 -- Treatment-Specific:**
- Spravato: buy & bill vs pharmacy, reimbursement/session, duration, Spravato with Me, Janssen Savings
- TMS: offering status, completion rate, reimbursement/session, start time

**On submit:** Creates organization + clinic(s) + providers + staff in database. Sends invitation emails to staff users.

### 2H. Clinic Activity Log

**Current source:** Internal app design (not yet built)
**Target:** Build in Flywheel

Per-clinic timeline of all changes:
- Location additions/removals
- Goal updates
- Proposal signatures
- Retainer payments
- Subscription changes
- Onboarding completion milestones

---

## 3. Workflow

### 3A. CSV Upload Pipeline
- **Trigger:** Super admin uploads CSV for revenues, expenses, upsells, or contractors
- **Action:** Validate headers (Zod) -> map clinic_name to clinic_id (fuzzy match with confirmation) -> insert rows -> log in internal_csv_uploads
- **Validation:** Numeric fields validated, dates parsed (ISO 8601 or MM/DD/YYYY), required fields enforced
- **UI:** Upload results screen showing X imported, Y failed, with per-row error details

### 3B. Payment Overdue Alert
- **Trigger:** Revenue record payment_status changes to 'overdue'
- **Action:** Slack notification to Danny + assigned account manager
- **Escalation:** 30-day, 60-day, 90-day warnings

### 3C. Onboarding Completion Workflow
- **Trigger:** Clinic submits onboarding form
- **Action:**
  1. Create org + clinic + provider + staff records
  2. Send invitation emails to all staff/providers
  3. Create GHL subaccount mapping task
  4. Create Meta ad account setup task
  5. Log in clinic activity log
  6. Notify assigned pod lead via Slack

### 3D. Contract Renewal Tracking
- **Trigger:** clinic_retainer.contract_end_date approaching (90, 60, 30 days)
- **Action:** In-app notification + Slack to account manager
- **Display:** Renewal status badge in revenue table
