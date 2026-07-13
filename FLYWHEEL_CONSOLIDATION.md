# Flywheel Consolidation -- Unified Product Requirements

**Purpose:** Single reference for consolidating 5 Psycle projects into the Flywheel platform.
**For:** Allen (CTO), Costa (Founder), engineering team
**Date:** July 11, 2026

---

## The 5 Source Projects

| # | Project | Repo | Stack | Infra | Owner | Status |
|---|---------|------|-------|-------|-------|--------|
| 1 | **Dashboard (prototype)** | `dannyxcosta/Psycle-Dashboard` | Vanilla JS + Express | GCP Cloud Run + Cloud SQL | Costa | Superseded by #2; retains onboarding form, Hope AI chat, account matcher mockups |
| 2 | **Dashboard (production)** | `Psycle-Health/dashboard` | React 19 + Vite + Express + Drizzle + Tailwind | GCP Cloud Run + Cloud SQL | Maryam | **Live** -- team uses daily. 45+ API endpoints, JWT auth, GHL sync, AES-256-GCM encryption |
| 3 | **Psycle-Data (intelligence)** | `Psycle-Health/Psycle-Data` | Python + BigQuery + Flask portal | GCP `ghl-data` (BQ, Cloud Run jobs, IAP portal) | Sebastian | **Live** -- 9 sync jobs, 14 dashboards, ML models, canonical metric definitions |
| 4 | **Internal (ops/finance)** | Private (Costa's GitHub) | React 19 + Vite + Express + Drizzle | GCP Cloud Run + Cloud SQL (separate) | Costa | Scaffolded, not deployed. Financial ops + clinics table + pods |
| 5 | **Flywheel (platform)** | `Psycle-Health/flywheel` | Rails 8 API + React/Vite SPA + ActiveRecord | **AWS** (ECS Fargate + RDS Postgres + S3/CloudFront) | Allen | Pre-build V0. EMR-agnostic scheduling + CH integration |

---

## What Each Project Contributes to the Flywheel

### From Dashboard Production (#2) -- The User-Facing Application Layer

**Auth & Access Control (MIGRATE)**
- Dual auth: Google OAuth + email/password, both produce unified JWT
- JWT claims: `{ sub, email, name, role, clinic_ids, permissions }`
- 4 roles: psycle_admin, psycle_team, clinic_admin, clinic_team
- 6 user types: C1, C2 (agency); Provider, Billing, Manager, Owner (clinic)
- Per-section permission toggles (can_view, can_edit) per user
- Provider sub-fields: provider_type (10 options), NPI number
- Auto-logout after 30min inactivity; secure httpOnly cookies
- `requirePermission()` middleware enforces per-section access
- RLS via Drizzle transactions with session variables

**GHL Contact Sync Pipeline (MIGRATE)**
- Three sync modes: real-time webhook, nightly reconciliation (2AM PT), initial backfill
- Per-clinic field mappings: `clinic_field_mappings` table translates GHL custom field IDs to app fields
- Raw JSONB storage (`ghl_contacts.raw_payload`) -- fields extracted at query time
- GHL appointments + calls sync
- Bottleneck rate limiting (8 concurrent for GHL API)
- SSE streaming progress during sync

**Patient Funnel UI (MIGRATE -- 7 data pages)**

| Page | API Endpoint | KPI Cards | Data Source |
|------|-------------|-----------|-------------|
| Leads | `/api/clinic/:id/leads` | Total, Followed Up, Pending, Consults Booked | GHL contacts |
| Calls | `/api/clinic/:id/calls` | Total, Inbound, Outbound, Avg Duration | GHL conversations |
| Consults | `/api/clinic/:id/consults` | Scheduled, Showed, No Show, Show Rate | GHL appointments |
| Insurances | `/api/clinic/:id/insurance` | Total Submissions, In-Network, OON, In-Network Rate | GHL contacts (VOB fields) |
| New Patients | `/api/clinic/:id/evaluations` | Total Evals, Showed, Qualified, Show Rate | GHL contacts (eval fields) |
| Prior Auths | `/api/clinic/:id/prior-auths` | Approved, Pending, Denied, Approval Rate | GHL contacts (PA fields) |
| Treatments | `/api/clinic/:id/treatments` | Active, Completed, Conversions, Completion Rate | GHL contacts (tx fields) |

Each page follows the canonical pattern: gradient-wrapped KPI cards + sortable/paginated/searchable data table + sticky glassmorphism filter bar + clinic switcher scoping.

**Meta Ads Dashboard (MIGRATE)**
- Full drill-down: accounts -> campaigns -> adsets -> ads with insights
- Winning ads with spend-tiered health scoring + minimum lead filter
- Geo-targeting heatmaps (Mapbox + Census ZCTA boundaries)
- Lead zip code visualization
- Agency map with clinic locations from Facebook Pages API
- 15-min TTL insights cache, 1-hour creative/metadata cache
- Date toggles: L7D / L14D / L28D / custom range
- Ad previews via `GET /{ad_id}/previews?ad_format=DESKTOP_FEED_STANDARD`

**Meta Sync Workers (MIGRATE)**
- Three sync modes: daily full (6AM PT), intraday (every 15min 7AM-10PM PT), initial backfill (37 months)
- Upserts via Drizzle into PostgreSQL
- Dashboard reads from pre-synced DB, never from Meta API directly

**Hope AI Chat (MIGRATE)**
- Claude-powered (Haiku 4.5 default, Sonnet toggle)
- SSE streaming responses
- Tool calling: Claude queries `/api/insights` for live Meta data
- System prompt injects account context per selected subaccount
- Max 4 tool rounds per turn

**Slack Integration (MIGRATE -- basic)**
- HTTP-based via Slack Web API (NOT Socket Mode)
- `conversations.history` + `chat.postMessage`
- Two channels per clinic: external (Psycle) + internal (Staff)
- Single agency-level bot token

**Clinic Settings (MIGRATE)**
- Clinic details form (name, NPI, address, timezone, etc.)
- Preferred Location field mapping (for multi-location GHL subaccounts)
- GHL sync status badge
- Treatment modalities multi-select
- Integration settings (Meta ad account ID, Slack channel IDs)

**File Uploads (MIGRATE)**
- Multer -> Google Cloud Storage (will need S3 equivalent)
- 10MB max; PDF, PNG, JPG, DOC, DOCX
- Key format: `{clinicId}/{fieldKey}/{filename}`
- Signed URLs (15-min expiration) for downloads

**Encryption (MIGRATE)**
- AES-256-GCM field-level encryption for PHI
- Dual-key rotation support (primary + previous)
- Encrypted fields: patient_name, dob, phone, email, insurance IDs, notes across all tables

**Shared UI Components (MIGRATE)**
- DataTable (sortable, paginated, searchable)
- KPICard with gradient-wrap border
- FilterBar (sticky, glassmorphism)
- ClinicSwitcher (role-aware dropdown)
- Sidebar (collapsible, icon-based)
- StatusBadge (color-coded per status enum per theme)
- GradientWrap (signature 1.5px border component)
- Three themes: Light, Dark, Colorful

**Status Enums (MIGRATE -- enforce with validation)**
- Evaluation: Upcoming, Showed, No Show, Rescheduled, CX by Clinic, CX by Patient
- Qualification: Qualified, NQ~Insurance, NQ~Geo, NQ~Medical, NQ~Financial, NQ~Other
- PA: Approved, Not Needed, Cash Pay, Submitted, Pending Submission, Med Clearance Req, Pending~Admin, Appealed, PA Denied, Appeal Denied, RX Sent, N/A
- Treatment: Scheduled, Spravato/Ketamine/TMS/Med Mgmt Conversion, Pending, Non Responsive, Lost~Denied/Cost/Other, Stopped Tx, Psycle F/U, Clinic F/U, N/A
- Network: In Network, Out of Network, OON (w/ Benefits), Insufficient Information, Waitlist (Pending Credentialing), Request Insurance

---

### From Psycle-Data (#3) -- The Intelligence & Data Warehouse Layer

**14 Dashboard Views (CONSOLIDATE into Flywheel UI)**

| Route | Name | Generator | Audience | What It Does |
|-------|------|-----------|----------|-------------|
| `/mb` | Media Buyer Command Center | `build_mb.py` | Media buyers | Per-clinic CPL/CPE/L->E, clinic health RAG (red/amber/green pace), eval-source reconciliation, rejected-ad flags, map with bullseye geo, pod filter chips |
| `/agency` | Acquisition Cockpit | `build_agency.py` | Agency leadership | Cross-clinic acquisition overview, multi-select pod chips, aggregate spend/leads/evals |
| `/heatmap` | Account Manager Cockpit | `build_heatmap.py` | Account managers | Geographic heatmap of leads/evals by ZIP, color-scaled metrics, per-clinic drill-down |
| `/cc` | Care Coordinator Report | `build_cc_report.py` | CC managers | Per-CC daily metrics: dials, connects (>=30s inbound / >=60s outbound), evals booked, show rate, SMS/email activity |
| `/client` | Client Monthly Report | n/a | Clients | Monthly rollup per clinic |
| `/client-report` | Client Performance Report | `build_client_report.py` | Agency + clients | Per-client investment, evals, show rate, conversions, ROI range, retainer, funnel waterfall |
| `/topads` | Top Ads by Geo & Insurance | topads pipeline | Media buyers | Best-performing ads ranked by geo + insurance match, creative launch scores, rejection tracking |
| `/calculator` | Eval Pricing Calculator | `calculator.html` | Sales/leadership | Platform pricing model: state + payer mix + exclusivity + margin -> recommended per-eval price. Uses trailing 12-mo Meta data ($728K / 3,402 evals). Payers tiered A/B/C by cost-per-approved-patient |
| `/insurance-*` | Insurance Intelligence | `build_insurance.py` | Agency | VOB/insurance analysis, carrier mapping, in-network rates |
| `/insurance-marketplace-*` | Insurance Marketplace | `build_insurance_marketplace.py` | Agency | Payer supply/demand matching |
| `/routing-*` | Lead Routing | `build_routing.py` | Ops | ML-driven clinic + CC suggestions per lead |
| `/ask` | Psycle Analyst | `psycle-analyst` service | Pilot (Sebastian + AMs) | Natural language queries against BQ data via vetted SQL templates |
| `/admin` | Portal Admin | `app.py` | Admins | Department-based page access control, user management |

**9 Sync Jobs (Cloud Run jobs -> must become Flywheel background jobs)**

| Job | Schedule | Target |
|-----|----------|--------|
| `ghl-contacts-sync` | Hourly | `ghl_raw.contacts` (BigQuery) |
| `ghl-conversations-sync` | Hourly | `ghl_raw.conversations` |
| `ghl-opportunities-sync` | Every 6h | `ghl_raw.opportunities` |
| `ghl-clinics-sync` | Daily 6AM | `ghl_raw.clinics` |
| `ghl-clinic-custom-values-sync` | Daily 6AM | `ghl_raw` |
| `meta-insights-sync` | Hourly | `meta_raw.ad_insights` |
| `psycle-dashboards-refresh` | Daily 6AM | Dashboard HTML assets |
| `psycle-topads-refresh` | Daily 6:30AM | Top Ads HTML |
| `psycle-heatmap-refresh` | Hourly | Heatmap HTML |

**Canonical Metric Definitions (ADOPT as source of truth)**

These are the most battle-tested, reconciled definitions across the company. The Flywheel MUST use these exact formulas:

| Metric | Definition | Bands |
|--------|-----------|-------|
| **CPL** | `ad_spend / paid_leads` (paid = `ad_id IS NOT NULL`) | <=20 green, 20-35 amber, >35 red |
| **CPE** | `ad_spend / attributable_evals` (created-date cohort, cancelled-excluded) | <=270 green, 270-380 amber, >380 red |
| **L->E** | `leads / evals` (a COUNT, not %; lower = better) | N/A |
| **Show Rate** | `showed / resolved` (resolved = SHOWED + NO SHOW + CX BY PATIENT + CX BY CLINIC; UPCOMING excluded) | N/A |
| **Clinic Health RAG** | RED: week_pct <0.70 AND month_pct < day_frac; AMBER: either; GREEN: neither | N/A |
| **CC Connect** | Completed call >= 30s inbound / >= 60s outbound (not just status=completed) | N/A |
| **CC Attribution** | Actor-based (caller), never `assigned_to` (differs in ~52% of contacts) | N/A |
| **Investment** | Agency model: retainer + ad_spend; Bundled/platform: flat_fee (forward from transition month) | N/A |
| **ROI** | `modeled_revenue / investment` as lo~hi range (SPV: per_session x 32; TMS: per_session x 36) | N/A |

**Critical data rules:**
- All daily buckets: `America/Los_Angeles` (PT), always. UTC causes date-boundary bugs.
- All lead-derived media metrics: paid only (`ad_id IS NOT NULL`) except lead-source panel + ZIP map.
- All eval counts exclude `CX BY PATIENT` / `CX BY CLINIC` except show-rate denominators.
- Three eval date bases exist (tracker scheduled-date, GHL created-date cohort, submission-date) -- always state which.
- Flags are INT64 0/1: use `SUM(is_x)` for counts, `AVG(is_x)` for rates. Never `COUNTIF`.

**BigQuery Data Model (3-layer architecture)**

| Layer | Purpose | Examples |
|-------|---------|---------|
| **Raw** (`*_raw`) | Append-only, verbatim from source | `ghl_raw.contacts`, `meta_raw.ad_insights` |
| **Modeled** (`ghl.*`, `meta.*`) | Dedup-to-latest, typed, semantic | `ghl.evaluations_norm`, `ghl.contact_funnel`, `meta.ad_performance_daily`, `ghl.cc_daily` |
| **Unified/Mart** | Cross-source joins, business logic | `ghl.client_month` (WIP), `ghl.clinic_retainer` |

Key modeled views to replicate in Flywheel's Postgres:
- `ghl.evaluations_norm` -- normalizes 34 messy eval status variants into `is_showed`, `is_conversion`, `tx_outcome`, `tx_modality`, `eval_outcome`, `is_qualified`, `is_pa_*`
- `ghl.contact_funnel` -- per-lead funnel flags (is_lead, is_pre_qualified, is_insurance_submitted, is_in_network, is_consult_scheduled, is_evaluation, is_conversion, is_nq_geo, is_nq_ins) + Meta attribution + geo
- `ghl.cc_daily` -- per-CC per-day activity (dials, connects, evals booked, show rate, SMS, emails)
- `meta.ad_performance_daily` -- per-ad per-day spend + leads + flight window

**ML Models (PORT to Flywheel)**

| Model | Purpose | Status | Key Finding |
|-------|---------|--------|-------------|
| Lead Scoring v2 | P(qualified_eval) at intake | Trained, NOT deployed | ROC-AUC 0.717 (honest, non-leaky). Signal is between clinic/source/modality, not between individual leads. Repurpose for budget allocation, not CC prioritization. |
| No-Show Prediction | Reminder intensity / booking window | Trained, not deployed | `network_fit` dominates (~4x next feature) |
| Lead-Clinic Matching | Route leads to best clinic+CC | TVF `ghl.route_lead(zip5,carrier,modality)` | ROC 0.651; needs distance/same-state constraint |
| Creative Rejection Risk | Pre-flight ad compliance | Trained, not deployed | n-gram AUC inflated; needs v2 |
| Ad Economics | Per-ad margin = price/eval - conf_cpe | View | Foundation for SCALE/HOLD/CUT recommendations |

**Eval Tracker Sync (44 Google Sheets)**
- `eval-tracker-sync` Cloud Run job: syncs 44 clinic Eval Tracker Google Sheets -> `ghl.evaluations` (BigQuery)
- MERGE every 15 minutes
- Uses Drive Workspace Domain-Wide Delegation (DWD) as sebastian@
- This is the canonical eval outcome source (clinics update sheets, not GHL directly)
- Contains PHI (patient_name, notes) -- locked dataset, never logged
- 6,478 rows / 41 clinics

**Additional Services**

| Service | Purpose |
|---------|---------|
| `ghl-webhook-receiver` | Real-time GHL webhooks -> BQ |
| `meta-status-sync` | Ad approval/rejection status tracking |
| `anomaly-monitor` | Data quality alerts |
| `psycle-analyst` | NL->SQL query engine (pilot) |
| `slack-ticketing` | Slack-based ops ticketing |
| `weekly-meta-report` | Automated weekly Meta performance emails |
| `google-ads-backfill` | Google Ads data sync |
| `asana-sync` | Fellow/Asana meeting note sync |

**Department-Based Access Control**
- Portal uses `access.json` in GCS bucket
- Admins, departments (each maps to page keys), users assigned to departments
- User sees union of their departments' pages + any extra_keys

---

### From Dashboard Prototype (#1) -- Product Design Artifacts

**Client Onboarding Form (`/onboarding.html`) -- MIGRATE to Flywheel**
5-step wizard:
1. General Info (contact, org details, locations with services multi-select + per-modality capacity)
2. Business & Compliance (EIN, CP 575 upload, BAA with signature pad + legal text)
3. Staff & Providers (staff members with roles, evaluating providers with NPI + bio + availability)
4. EMR, Policies & Costs (EMR selection, eval cost, cash-pay pricing, deposit policies, cancellation policy, paperwork uploads)
5. Treatment-Specific (Spravato details, TMS details, pricing/reimbursement)

**Accounts Matcher (`/mockup-accounts.html`) -- MIGRATE to Flywheel Settings**
Three-way mapping UI: Organization -> GHL Location -> Meta Ad Account -> Campaigns
- Org cards with expand/collapse
- Single-region orgs: mapping fields directly on org row
- Multi-region orgs: indented subaccount rows with individual mappings
- Campaign pills (multi-select from selected ad account)
- Result Type selector (Lead Form vs View Content) per subaccount
- Actions: + Add Organization, + Subaccount, Edit, Rename, Remove

---

### From Internal App (#4) -- Financial Ops & Organizational Data

**Financial Operations (MIGRATE behind permission gate)**

| Section | KPI Cards | Data Source |
|---------|-----------|-------------|
| Revenues | Total MRR, Growth %, Avg Revenue/Clinic, Total Clients | CSV upload -> Stripe/QB future |
| Upsells | Total Upsell Revenue, Conversion Rate, Avg Value, Pipeline | CSV upload / manual |
| Expenses | Total Monthly, Category Breakdown, Burn Rate, Net Margin | CSV upload -> QB future |
| Contractors | Total Spend, Active Count, Avg Rate, Utilization | CSV upload |
| Efficiency | Net Margin %, Revenue/Employee, CAC, LTV:CAC | Derived from Revenue + Expenses + Ads |

**Pod System (MIGRATE -- already in dashboard but Internal has the implementation)**
- Pod -> members (users) + clinics
- Pod-based access scoping for account managers
- Account managers see total revenue for other pods but NOT per-clinic breakdown
- Super admins see everything

**Database Tables (MIGRATE with `internal_` prefix)**
- `internal_revenues` (clinic_id, period, mrr, contract_value, payment_status)
- `internal_upsells` (clinic_id, upsell_type, value, status, dates)
- `internal_expenses` (clinic_id nullable, category, vendor, amount, recurring flag)
- `internal_contractors` (name, role, rate_type, rate, hours/month)
- `internal_contractor_assignments` (contractor -> clinic/pod)
- `internal_csv_uploads` (upload log with row counts + errors)
- `internal_audit_logs`

**CSV Upload Pipeline (MIGRATE)**
- User selects target section -> uploads CSV -> server validates headers (Zod) -> maps clinic_name to clinic_id -> inserts with success/failure tracking
- Expected formats documented for each table
- Fuzzy clinic name matching with confirmation

**Clinics Table (enriched beyond dashboard)**
- Same GHL location IDs as primary keys
- org_id, city, state, is_active
- Designed with same identifiers for seamless joins

---

### From Flywheel (#5) -- The Foundation Being Built

**EMR-Agnostic Platform (EXISTING -- extend)**
- `Emr::BaseAdapter` interface with IntakeQ as impl #1 of ~18
- Adapter capability spectrum (`availability_capability`: `:native` vs `:best_effort`)
- Clinic -> Provider -> EmrConnection domain model

**Scheduling (P1 -- IN PROGRESS)**
- Coordinator-driven calendar (no patient-facing surface)
- IntakeQ native availability (real open slots via booking-widget API)
- Booking write + pre-write re-query + webhook reconcile
- Confirmation-call backstop workflow
- Net-new master calendar + reschedule
- Appointment + eval-status write-back to CRM

**Connective Health Integration (P2 -- IN PROGRESS)**
- FHIR R4 / US Core 6.1.0 bundle ingestion
- Request-then-receive delivery (trigger -> CH builds async -> S3 drop + webhook)
- Prod latency ~12+ hours
- `ch_payloads` replay store (encrypted raw, durable)
- Source-agnostic mapping engine: `(ch_fhir | ch_pdf | ghl_crm) -> {EMR, provider, field}`
- >=2-antidepressant-trial qualification from MedicationRequest resources
- Ships behind a flip (Keragon fallback)

**HIPAA / RLS (DONE in Flywheel)**
- Postgres RLS + FORCE ROW LEVEL SECURITY on every clinic-scoped table
- `flywheel_app` role (NOT superuser, NOT BYPASSRLS, NOT table owner)
- Per-request `ClinicContext` via `around_action` / `around_perform`
- Connection-checkin hook clears context (pooling safety)
- SQL schema format (`db/structure.sql` preserves policies)
- Proven by `clinic_rls_isolation_test.rb` (drops to unprivileged role, asserts cross-tenant isolation)

**Infrastructure (DONE in Flywheel)**
- AWS ECS Fargate (core) + S3/CloudFront (ops SPA)
- RDS Postgres
- Sidekiq + Redis for background jobs
- Local mocks for CH, GHL, IntakeQ, MinIO (self-contained dev)
- CI/CD: GitHub Actions, path-filtered per app, change-aware deploys

---

## Unified Feature Map for Flywheel

### Tier 1 -- Core Platform (must have for consolidation)

| Feature | Source Project | Migration Complexity |
|---------|--------------|---------------------|
| JWT Auth (Google OAuth + email/password) | Dashboard (#2) | Port from JS to Rails |
| User roles + permission toggles | Dashboard (#2) | Port schema + middleware |
| Clinic/Org/User management | Dashboard (#2) + Internal (#4) | Merge schemas |
| GHL contact sync (webhook + reconciliation + backfill) | Dashboard (#2) | Port from JS to Rails (pattern exists) |
| Per-clinic field mappings | Dashboard (#2) | Port (Flywheel already has `field_mappings`) |
| Patient funnel pages (7 data views) | Dashboard (#2) | React components migrate; API routes rewrite to Rails |
| Meta Ads dashboard (drill-down + winning ads) | Dashboard (#2) | Port Meta API client to Ruby |
| Meta sync workers (daily/intraday/backfill) | Dashboard (#2) + Psycle-Data (#3) | Unify into Sidekiq jobs |
| Clinic settings + account matcher | Dashboard (#1 mockups) + Dashboard (#2) | Build from mockup specs |
| Pod system | Internal (#4) + Dashboard (#2) | Merge pod tables |
| RLS enforcement | Flywheel (#5) already done | Extend to all new tables |
| AES-256-GCM encryption | Dashboard (#2) | Port to Active Record Encryption |
| Audit logging | Dashboard (#2) + Internal (#4) | Unify into single audit system |

### Tier 2 -- Intelligence Layer (high value, complex)

| Feature | Source Project | Migration Complexity |
|---------|--------------|---------------------|
| Media Buyer Command Center (`/mb`) | Psycle-Data (#3) | Rewrite from Python/BQ to Rails+React |
| CC Report (`/cc`) | Psycle-Data (#3) | Rewrite |
| Client Performance Report | Psycle-Data (#3) | Rewrite |
| Acquisition Cockpit (`/agency`) | Psycle-Data (#3) | Rewrite |
| Heatmap (`/heatmap`) | Psycle-Data (#3) | Rewrite (Mapbox integration exists in #2) |
| Top Ads intelligence | Psycle-Data (#3) | Rewrite |
| Canonical metric definitions | Psycle-Data (#3) `METRICS_DICTIONARY.md` | Implement in Rails service layer |
| `evaluations_norm` semantic layer | Psycle-Data (#3) | Port 34-status normalization to Postgres views |
| `contact_funnel` view | Psycle-Data (#3) | Port to Postgres materialized view |
| `cc_daily` view | Psycle-Data (#3) | Port to Postgres |
| Eval Tracker sync (44 Google Sheets) | Psycle-Data (#3) | Sidekiq job + Google Sheets API |
| Eval Pricing Calculator | Psycle-Data (#3) `calculator.html` | Port static data + React component |

### Tier 3 -- EMR & Clinical (Flywheel-native, extend)

| Feature | Source Project | Status |
|---------|--------------|--------|
| EMR adapter interface | Flywheel (#5) | In progress |
| IntakeQ integration | Flywheel (#5) | In progress |
| Scheduling (coordinator calendar) | Flywheel (#5) | In progress |
| Connective Health FHIR ingestion | Flywheel (#5) | In progress |
| Source-agnostic field mapping engine | Flywheel (#5) | In progress |
| >=2-trial qualification | Flywheel (#5) | In progress |

### Tier 4 -- Financial Ops & Advanced

| Feature | Source Project | Migration Complexity |
|---------|--------------|---------------------|
| Revenue tracking | Internal (#4) | Port schema + CSV upload |
| Upsells, Expenses, Contractors | Internal (#4) | Port schema + CSV upload |
| Efficiency metrics (derived) | Internal (#4) | Port calculation logic |
| Hope AI chat (Claude + tool calling) | Dashboard (#1/#2) | Port Anthropic SDK integration |
| Slack integration | Dashboard (#2) | Port (basic HTTP -> Socket Mode) |
| Client onboarding wizard | Dashboard (#1) | Build from mockup spec |
| Lead scoring / ML models | Psycle-Data (#3) | Port BQML models or retrain on Postgres |
| Lead routing | Psycle-Data (#3) | Port `route_lead` logic |
| Psycle Analyst (NL->SQL) | Psycle-Data (#3) | Port or rebuild on Flywheel data |
| Insurance intelligence/marketplace | Psycle-Data (#3) | Rewrite |

---

## Data Layer Consolidation

### The Problem: Same Data, Different Homes

```
GHL API
  |
  +---> Dashboard Postgres (Maryam) -- real-time webhook + nightly reconcile
  |       ghl_contacts (raw JSONB), ghl_appointments, ghl_calls
  |       field_mappings -> query-time extraction
  |
  +---> BigQuery (Sebastian) -- hourly batch sync
  |       ghl_raw.contacts -> ghl.contact_funnel (modeled)
  |       ghl_raw.conversations -> ghl.cc_daily (modeled)
  |       44 Google Sheets -> ghl.evaluations -> ghl.evaluations_norm (canonical evals)
  |
  +---> Flywheel RDS (Allen) -- own sync, own schema
          patients, appointments, ch_payloads, field_mappings

Meta API
  |
  +---> Dashboard Postgres (Maryam) -- meta-sync workers (daily + intraday)
  |       meta_ad_accounts, meta_campaigns, meta_adsets, meta_ads, meta_insights
  |
  +---> BigQuery (Sebastian) -- hourly sync
          meta_raw.ad_insights -> meta.ad_performance_daily (canonical spend)
```

### The Target: Single Flywheel RDS

```
GHL API ---> Flywheel Sidekiq jobs ---> RDS Postgres
  |                                        |
  +-- webhook receiver (real-time)         +-- ghl_contacts (raw JSONB)
  +-- reconciliation job (nightly)         +-- ghl_appointments
  +-- backfill job (on-demand)             +-- ghl_calls
                                           +-- contact_funnel (materialized view)
                                           +-- evaluations_norm (materialized view)
                                           +-- cc_daily (materialized view)

Meta API ---> Flywheel Sidekiq jobs ---> RDS Postgres
  |                                        |
  +-- daily full sync (6AM PT)             +-- meta_ads (with creative data)
  +-- intraday sync (15min, 7AM-10PM)      +-- meta_insights (per-ad per-day)
  +-- backfill (37-month, one-time)        +-- ad_performance_daily (view)

Google Sheets ---> Flywheel Sidekiq job ---> RDS Postgres
  |                                           |
  +-- eval-tracker-sync (every 15min)         +-- evaluations (from 44 sheets)
                                              +-- evaluations_norm (view)

CH/EMR ---> Flywheel (existing) ---> RDS Postgres
                                       |
                                       +-- ch_payloads, patients, appointments
                                       +-- field_mappings (source-agnostic)
```

### Schema Unification

**Shared primary keys (already aligned across all projects):**
- `clinic_id` = GHL location ID (TEXT) -- universal join key
- `ghl_contact_id` = GHL contact ID -- universal patient/lead key
- `pod_id` = UUID -- team grouping key
- `user.email` = Google OAuth email -- identity key
- `org_id` = UUID -- organization key

**Tables to merge/reconcile:**

| Logical Entity | Dashboard (#2) | Internal (#4) | Flywheel (#5) | Psycle-Data (#3) | Target |
|---------------|---------------|--------------|--------------|-----------------|--------|
| Organizations | `organizations` | `organizations` | `clinics` (flat) | -- | `organizations` |
| Clinics | `clinics` (GHL loc ID PK) | `clinics` (same IDs) | `clinics` (ActiveRecord) | `ghl.clinic_lookup` | `clinics` (merge all columns) |
| Users | `users` (JWT claims) | `users` (Google OAuth) | -- | `access.json` | `users` (unified) |
| Pods | `pods`, `pod_members`, `pod_clinics` | Same | -- | -- | `pods` + junction tables |
| GHL Contacts | `ghl_contacts` (raw JSONB) | -- | -- | `ghl_raw.contacts` | `ghl_contacts` |
| Field Mappings | `field_mappings` (per-clinic) | -- | `field_mappings` (source-agnostic) | -- | `field_mappings` (Flywheel's, extended) |
| Evaluations | Extracted from `ghl_contacts` | -- | -- | `ghl.evaluations` (from Sheets) | Both: JSONB extraction + Sheet sync |
| Meta Insights | `meta_insights` (per-ad per-day) | -- | -- | `meta_raw.ad_insights` | `meta_insights` |
| Revenue | -- | `internal_revenues` | -- | `ghl.clinic_retainer` (from Sheets) | `revenues` (merge both sources) |

---

## Key Architectural Decisions for Allen

### 1. BigQuery vs Postgres for Analytics

Sebastian's intelligence layer runs on BigQuery (columnar, great for aggregations). The Flywheel uses RDS Postgres (row-based, great for CRUD). Options:

- **Option A:** Keep BQ as the analytics warehouse, Flywheel Postgres as the operational DB, with ETL between them. Adds complexity but preserves BQ's analytical strengths.
- **Option B:** Move everything to Postgres with materialized views for analytics. Simpler architecture but may struggle with heavy aggregations at scale.
- **Option C:** Use Postgres for operational + real-time, periodically ETL to a data warehouse (BQ, Redshift, or Postgres read replicas) for the intelligence dashboards.

Sebastian's 9 sync jobs, ML models, and the `evaluations_norm`/`contact_funnel`/`cc_daily` views are the most critical pieces to preserve regardless of approach.

### 2. Eval Tracker Sheets vs GHL as Eval Source

The single biggest data reconciliation issue: clinics update eval outcomes in Google Sheets, NOT in GHL. Sebastian's `eval-tracker-sync` (44 sheets, MERGE every 15min) is the canonical eval source. Until clinics adopt a new workflow (updating evals in the Flywheel UI), the Sheet sync must continue.

### 3. Ruby vs JavaScript for Ported Code

Flywheel is Rails; Dashboard is Express/JS. All dashboard API routes, sync workers, and business logic need porting. The Flywheel PRD acknowledges this: "port-from-spec, not invent-from-scratch." Sebastian's Python/BQ code needs rewriting regardless of backend language.

### 4. GCP -> AWS Migration

- Cloud SQL Postgres -> RDS Postgres (straightforward)
- Cloud Run -> ECS Fargate (already done in Flywheel)
- Cloud Storage -> S3 (file uploads, onboarding documents)
- BigQuery -> TBD (see Decision #1)
- Cloud Scheduler + Cloud Run Jobs -> Sidekiq + Redis (already in Flywheel)
- IAP auth -> Flywheel's own auth (JWT + RLS)

### 5. Multi-Tenancy Model

Three-tier hierarchy (Agency -> Organization -> Clinic) must be preserved:
- Agency (Psycle Team): sees everything
- Organization: sees their clinics, toggle between or aggregate
- Clinic: sees only own data
- Additional: Pod-based scoping for account managers (cross-org, assigned clinics only)
- Clinic users CANNOT see top-of-funnel Meta metrics (impressions, clicks, spend, CPL)

---

## Brand System (Unchanged -- Apply to All Flywheel UI)

| Token | Value |
|-------|-------|
| Blue | `#609FFC` |
| Lavender | `#B79CF7` |
| Pink | `#F3A6C8` |
| Orange | `#F87C16` |
| Golden Orange | `#FF9E1F` |
| Deep Purple | `#4D0052` |
| Deep Indigo | `#2A0052` |
| Off White | `#F5F5F5` |
| CTA gradient | `linear-gradient(135deg, #F87C16, #B79CF7)` |
| Border gradient | `linear-gradient(135deg, #F3A6C8, #B79CF7, #609FFC)` |
| Font | DM Sans (300-700) |
| Themes | Light, Dark, Colorful |

---

## Critical Business Rules (Non-Negotiable)

1. **Meta lead counting:** `actions` array where `action_type === "lead"` ONLY. Never `onsite_conversion.lead_grouped`.
2. **Meta CPL:** Use `cost_per_action_type` from API. Never calculate as `spend / leads`.
3. **All timestamps PT:** `America/Los_Angeles` for daily bucketing. UTC causes month-boundary bugs.
4. **Paid-only metrics:** All media reporting uses `ad_id IS NOT NULL` except lead-source panel + ZIP map.
5. **GHL read-only:** Never create custom fields in GHL. Preferred Location is read-only.
6. **Eval counting:** Exclude CX BY PATIENT / CX BY CLINIC from eval counts (keep in show-rate denominators).
7. **CC attribution:** Actor-based (who made the call), not `assigned_to` (differs ~52%).
8. **No PHI in logs:** Patient names, DOB, insurance IDs, clinical notes only in encrypted DB fields.
9. **GHL location ID = clinic PK:** Never generate separate IDs.
10. **Production write guard:** All GHL writes go through dry-run-default, sandbox-only, plan-hash-approved guard. No direct API writes.

---

## Open Items / Risks

| Item | Owner | Risk |
|------|-------|------|
| Cleartext AWS key + ~60 GHL tokens in Integrations sheet | Sebastian (escalated) | HIGH -- security |
| BigQuery vs Postgres analytics decision | Allen | HIGH -- architectural |
| 44 Eval Tracker Google Sheets -> how to migrate clinics off Sheets | Costa + Allen | MEDIUM -- operational change management |
| BAAs not yet in place (Emerge, cloud providers, Anthropic) | Costa | HIGH -- blocks real PHI |
| Lead scoring v1 deployed but leaky; v2 honest but weak within-clinic | Sebastian | MEDIUM -- ML |
| Bundled/platform investment model hard-disabled in code | Sebastian | LOW -- code fix |
| Show-rate denominator inconsistency between generators | Sebastian | LOW -- reconcile to canonical |
| Connective Health prod latency ~12+ hours | Allen + CH | MEDIUM -- async design handles it |
| IntakeQ week-1 sandbox verifications not yet done | Allen | MEDIUM -- blocks P1 |
