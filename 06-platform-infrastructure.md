# PRD 06 -- Platform & Infrastructure

**Department Lead:** Allen (CTO)
**Users:** Engineering team, all other departments (consumed via middleware)
**Source projects:** Flywheel (Rails 8 + RLS + HIPAA), Dashboard production (auth + encryption + sync), Psycle-Data (sync jobs + data pipeline)

---

## 1. Data Schema

### Core Tables (Foundation -- Must Exist Before All Other PRDs)

**`organizations`**
```sql
CREATE TABLE organizations (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  settings JSONB DEFAULT '{}',
  plan TEXT DEFAULT 'starter',  -- marketplace billing tier
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

**`clinics`** (PK = GHL location ID, NEVER generate separate IDs)
```sql
CREATE TABLE clinics (
  id TEXT PRIMARY KEY,                    -- = GHL location ID
  org_id TEXT REFERENCES organizations(id),
  name TEXT NOT NULL,
  slug TEXT,
  address TEXT, city TEXT, state TEXT, zip TEXT,
  phone TEXT, email TEXT, website TEXT,
  timezone TEXT DEFAULT 'America/Denver',
  clinic_npi TEXT,
  emr_type TEXT,
  settings JSONB DEFAULT '{}',           -- Slack IDs, modalities, capacity, screening Qs
  is_active BOOLEAN DEFAULT true,
  onboarding_completed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

**`users`**
```sql
CREATE TABLE users (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  email TEXT UNIQUE NOT NULL,
  name TEXT,
  password_hash TEXT,                     -- bcrypt (null for Google OAuth only)
  avatar_url TEXT,

  -- INTERNAL (Psycle team) vs EXTERNAL (clinic staff)
  is_internal BOOLEAN DEFAULT false,      -- true = Psycle employee, false = clinic user

  -- For INTERNAL users (Psycle team):
  psycle_user_type TEXT,
    -- admin, growth, am, cc, content (NULL for external/clinic users)

  -- For EXTERNAL users (clinic staff):
  role TEXT,
    -- clinic_admin, clinic_team (NULL for internal users)
  clinic_user_type TEXT,
    -- provider, billing, manager, owner (NULL for internal users)
  provider_type TEXT,                     -- null unless clinic_user_type = 'provider'
    -- MD/DO~Psychiatry, MD/DO~Anesthesiology, MD/DO~Neurology,
    -- MD/DO~Pain Management, MD/DO~Emergency Medicine,
    -- PMHNP, NP, CRNA, PA, Other
  npi_number TEXT,                        -- null unless clinic_user_type = 'provider'

  org_id TEXT REFERENCES organizations(id),
  is_active BOOLEAN DEFAULT true,
  employment_type TEXT,                   -- salary, hourly, w9_salary, w9_inv (from team roster)
  work_location TEXT,                     -- in-person, remote, hybrid
  last_login_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Psycle User Types (internal only):**

| Type | Access Scope | Example Users |
|------|-------------|---------------|
| `admin` | Full access to everything -- user management, settings, all data, all departments | Danny, Elijah, Catherine, Allen, Maryam, Nick, Eleeza |
| `growth` | Media buying dashboards, Meta ads, creative intelligence, spend data, assigned pod's clinics | Gene, Sebastian R., Rodrigo |
| `am` | Account management, client reports, leaderboard, patient funnel pages, clinic settings, assigned pod's clinics | Jake (lead), Zohreh, Kyla, Cody, Kerrie, + ~12 more |
| `cc` | CC dashboard, calls, consults, lead follow-up, eval booking, assigned pod's clinics | Rachel (manager), Aliyah, Jessica, Zohreh, Francis, Christine, Jamie, Anne Marie, Mia, David, Miguel, John, Daisy, + others |
| `content` | Content tools (future scope), read-only on dashboards | Gabrys, Connor, Camila, Valeria, Sheena, Zara, Juan, Rosalind |

**`user_clinics`** (which clinics a user can access)
```sql
CREATE TABLE user_clinics (
  user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  clinic_id TEXT NOT NULL REFERENCES clinics(id) ON DELETE CASCADE,
  role TEXT DEFAULT 'viewer',             -- viewer, editor, admin
  created_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (user_id, clinic_id)
);
```

**`user_permissions`** (per-section access toggles, overridable per user)
```sql
CREATE TABLE user_permissions (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  section TEXT NOT NULL,
    -- ads, leads, calls, consults, insurances, new_patients, prior_auths, treatments,
    -- mb, cc_report, topads, agency, heatmap, client_report, settings
  can_view BOOLEAN DEFAULT false,
  can_edit BOOLEAN DEFAULT false,
  UNIQUE(user_id, section)
);
```

**Default permission dispositions by Psycle user type** (applied on user creation, overridable by Admin):

| Section | Admin | Growth | AM | CC |
|---------|-------|--------|----|----|
| ads | view+edit | view+edit | -- | -- |
| leads | view+edit | view | view | view+edit |
| calls | view+edit | -- | view | view |
| consults | view+edit | -- | view | view |
| insurances | view+edit | -- | view+edit | view |
| new_patients | view+edit | -- | view+edit | view |
| prior_auths | view+edit | -- | view+edit | -- |
| treatments | view+edit | -- | view+edit | -- |
| mb | view+edit | view+edit | -- | -- |
| cc_report | view+edit | -- | -- | view |
| topads | view+edit | view+edit | -- | -- |
| agency | view+edit | view+edit | -- | -- |
| heatmap | view+edit | -- | view | -- |
| client_report | view+edit | -- | view | -- |
| settings | view+edit | -- | view | -- |

Content user type gets read-only on dashboards they're granted via permission overrides (no defaults).

**`pods`** (team groupings)
```sql
CREATE TABLE pods (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  name TEXT NOT NULL,
  color TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE pod_members (
  pod_id TEXT NOT NULL REFERENCES pods(id) ON DELETE CASCADE,
  user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role TEXT DEFAULT 'member',             -- lead, member
  PRIMARY KEY (pod_id, user_id)
);

CREATE TABLE pod_clinics (
  pod_id TEXT NOT NULL REFERENCES pods(id) ON DELETE CASCADE,
  clinic_id TEXT NOT NULL REFERENCES clinics(id) ON DELETE CASCADE,
  PRIMARY KEY (pod_id, clinic_id)
);
```

**`audit_logs`** (HIPAA -- every action logged)
```sql
CREATE TABLE audit_logs (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  timestamp TIMESTAMPTZ DEFAULT NOW(),
  user_id TEXT REFERENCES users(id),
  user_email TEXT,
  action TEXT NOT NULL,                   -- VIEW, CREATE, UPDATE, DELETE, LOGIN, LOGOUT
  entity_type TEXT,                       -- evaluations, vob, users, etc.
  entity_id TEXT,
  details JSONB DEFAULT '{}',            -- what changed (old_value -> new_value)
  ip_address TEXT,
  user_agent TEXT
);
```

---

## 2. Authentication & Authorization

### 2A. Dual Auth Flow

**Google OAuth** (primary for Psycle Team):
- Verify Google token at login via `POST /api/v1/auth/google`
- Auto-create `@psyclehealth.com` users as `psycle_team` on first login
- Produce app JWT with embedded claims

**Email/Password** (for clinic users):
- bcrypt hash stored in users.password_hash
- Login via `POST /api/v1/auth/login`
- Produce same app JWT

**JWT Claims:**
```json
// Internal (Psycle team) example:
{
  "sub": "user_id",
  "email": "jake@psyclehealth.com",
  "name": "Jake Inzalaca",
  "is_internal": true,
  "psycle_user_type": "am",
  "pod_ids": ["gluta_a"],
  "clinic_ids": ["ZpNsx02KGCBh1bRMRU6K", "abc123"],  // derived from pod assignments
  "permissions": {
    "insurances": { "view": true, "edit": true },
    "new_patients": { "view": true, "edit": true },
    "ads": { "view": false, "edit": false }
  }
}

// External (clinic user) example:
{
  "sub": "user_id",
  "email": "dr.chen@clinic.com",
  "name": "Dr. Sarah Chen",
  "is_internal": false,
  "role": "clinic_admin",
  "clinic_user_type": "provider",
  "org_id": "org_123",
  "clinic_ids": ["ZpNsx02KGCBh1bRMRU6K"],
  "permissions": {
    "insurances": { "view": true, "edit": true },
    "new_patients": { "view": true, "edit": false },
    "ads": { "view": false, "edit": false }
  }
}
```

**JWT claims drive:**
- RLS session variables (set before every DB query)
- Frontend route guarding (which sidebar sections render)
- API endpoint authorization (which data returned)
- `requirePermission(section, action)` middleware on every route

### 2B. Row-Level Security (RLS)

Flywheel already has RLS implemented. Extend to all new tables:

```sql
-- Session variables set per-request:
--   app.is_internal  = 'true' | 'false'
--   app.user_type    = 'admin' | 'growth' | 'am' | 'cc' | 'content' (internal)
--                      OR 'clinic_admin' | 'clinic_team' (external)
--   app.user_id      = user ID (for pod-based filtering)
--   app.clinic_id    = clinic ID (for clinic-scoped users)
--   app.org_id       = organization ID

-- Internal Psycle users: admin sees all; others see pod-assigned clinics
CREATE POLICY internal_admin_access ON [table] FOR ALL
  USING (current_setting('app.is_internal', true) = 'true'
    AND current_setting('app.user_type', true) = 'admin');

CREATE POLICY internal_pod_access ON [table] FOR ALL
  USING (current_setting('app.is_internal', true) = 'true'
    AND clinic_id IN (
      SELECT pc.clinic_id FROM pod_clinics pc
      JOIN pod_members pm ON pm.pod_id = pc.pod_id
      WHERE pm.user_id = current_setting('app.user_id', true)
    ));

-- External clinic users: see only their assigned clinic(s)
CREATE POLICY clinic_access ON [table] FOR ALL
  USING (current_setting('app.is_internal', true) = 'false'
    AND clinic_id IN (
      SELECT uc.clinic_id FROM user_clinics uc
      WHERE uc.user_id = current_setting('app.user_id', true)
    ));
```

**Production connection contract (from Flywheel README):**
- Runtime connection MUST use `flywheel_app` role (NOT superuser, NOT BYPASSRLS, NOT table owner)
- Migrations run as owner/admin role
- Connection-checkin hook clears clinic context (pooling safety)

### 2C. Encryption

**AES-256-GCM field-level encryption** for all PHI fields:
- patient_name, dob, phone, email (in contact records)
- insurance IDs, plan details
- clinical notes
- Any identifiable data in raw JSONB payloads

**Key rotation:** Every 180 days. Dual-key support during rotation window (primary + previous).

**Rails implementation:** Use Active Record Encryption (built-in Rails 7+) with custom key provider pointing to AWS Secrets Manager.

---

## 3. Data Servicing (Platform-Level)

### 3A. Sidebar Navigation

Collapsible, icon-based sidebar (modeled after GHL):

**Sidebar visibility by Psycle user type:**

| Section | Admin | Growth | AM | CC | Content | Clinic Admin | Clinic Team |
|---------|-------|--------|----|----|---------|-------------|-------------|
| **Agency** | | | | | | | |
| Leaderboard | Y | Y | Y | -- | -- | own row | -- |
| Clinics (org mgmt) | Y | -- | -- | -- | -- | -- | -- |
| **Subaccount** (clinic switcher) | | | | | | | |
| Ads (Meta drill-down) | Y | Y | -- | -- | -- | -- | -- |
| Leads | Y | Y | Y | Y | -- | -- | -- |
| Calls | Y | -- | Y | Y | -- | -- | -- |
| Consults | Y | -- | Y | Y | -- | -- | -- |
| Insurances | Y | -- | Y | Y | -- | Y | Y |
| New Patients (Evals) | Y | -- | Y | Y | -- | Y | Y |
| Prior Auths | Y | -- | Y | -- | -- | Y | Y |
| Treatments | Y | -- | Y | -- | -- | Y | Y |
| **Intelligence** | | | | | | | |
| Media Buyer (`/mb`) | Y | Y | -- | -- | -- | -- | -- |
| CC Report (`/cc`) | Y | -- | -- | Y | -- | -- | -- |
| Top Ads (`/topads`) | Y | Y | -- | -- | -- | -- | -- |
| Acquisition Cockpit | Y | Y | -- | -- | -- | -- | -- |
| Heatmap | Y | -- | Y | -- | -- | -- | -- |
| Client Report | Y | -- | Y | -- | -- | -- | -- |
| **Chat** | | | | | | | |
| Hope AI | Y | Y | Y | Y | -- | -- | -- |
| Psycle (Slack) | Y | Y | Y | Y | -- | -- | -- |
| Staff (Slack) | Y | Y | Y | Y | -- | -- | -- |
| **Settings** | | | | | | | |
| Clinic Details | Y | -- | Y | -- | -- | Y | -- |
| Insurance Details | Y | -- | Y | -- | -- | Y | -- |
| Integrations | Y | -- | -- | -- | -- | -- | -- |
| Team | Y | -- | -- | -- | -- | Y | -- |
| Accounts (mapping) | Y | -- | -- | -- | -- | -- | -- |

All non-Admin internal users are scoped to their assigned pod's clinics.

### 3B. Clinic Switcher

- **Psycle Admin/Team:** Dropdown showing all active clinics + "All Locations" aggregate
- **Org users:** Their clinics + org-level aggregate
- **Clinic users:** Static label showing assigned clinic(s)
- Selection scopes ALL API calls
- Persists in state across navigation

### 3C. Theme System

Three themes: Light (default), Dark, Colorful
- CSS variables on `<html>` element
- LocalStorage persistence
- Brand tokens (see FLYWHEEL_CONSOLIDATION.md for full color system)

### 3D. Hope AI Chat

- Anthropic API (Haiku 4.5 default, Sonnet toggle)
- SSE streaming for real-time responses
- Tool calling: Claude queries insights API for live Meta data
- System prompt injects account context per selected subaccount
- Max 4 tool rounds per turn

### 3E. Slack Integration

- HTTP-based via Slack Web API (upgrade to Socket Mode as future work)
- Two channels per clinic: external (Psycle) + internal (Staff)
- `conversations.history` + `chat.postMessage`
- Single agency-level bot token

### 3F. Psycle Analyst (Future)

- Natural language -> SQL query engine
- Vetted SQL templates only (no arbitrary queries)
- Department-scoped access
- Current: Sebastian's pilot with account managers

---

## 4. Workflow (Infrastructure)

### 4A. GHL Sync Pipeline

**Three sync modes (mirrors Dashboard production):**

| Mode | Trigger | Scope |
|------|---------|-------|
| Real-time | GHL ContactUpdate webhook | Single contact |
| Nightly reconciliation | Sidekiq cron, 2 AM PT | All contacts updated since last sync |
| Initial backfill | One-time per clinic | All contacts for newly connected clinic |

**Pipeline:** Webhook -> fetch full contact via GHL API (always, because webhook payload may be partial) -> upsert ghl_contacts -> extract fields via field_mappings -> refresh materialized views

**Rate limiting:** Bottleneck (8 concurrent for GHL API)
**Non-fatal:** If appointments/calls fail, contact data preserved
**DLQ:** Failed webhooks queued for retry

### 4B. Meta Sync Pipeline

| Mode | Schedule | Scope |
|------|----------|-------|
| Daily full | 6 AM PT | All accounts -- insights (8 days), campaigns, adsets, ads + creative |
| Intraday | Every 15 min, 7 AM - 10 PM PT | Active accounts -- today's account + campaign insights only |
| Initial backfill | One-time | 37-month history per active account |

### 4C. Eval Tracker Sheet Sync (Transitional)

- Every 15 minutes via Sidekiq
- 44 clinic Google Sheets -> evaluations table (MERGE/upsert)
- Google Sheets API with Domain-Wide Delegation
- PHI fields encrypted before storage
- Sunset as clinics adopt Flywheel UI

### 4D. Materialized View Refresh

After each sync cycle completes:
- Refresh mv_contact_funnel
- Refresh mv_cc_daily
- Refresh mv_ad_performance_daily
- Refresh mv_evaluations_norm
- Refresh mv_clinic_spend_daily
- Refresh mv_clinic_efficiency

### 4E. File Upload Pipeline

- Multer for multipart parsing -> S3 (replacing GCS)
- 10 MB max; PDF, PNG, JPG, DOC, DOCX
- Key format: `{clinicId}/{fieldKey}/{filename}`
- Pre-signed URLs (15-min expiration) for downloads
- Used by: onboarding documents, PA paperwork, clinic consents

### 4F. Audit Logging

Every data access, login, and API call logged:
- User identity (from JWT)
- Action (VIEW, CREATE, UPDATE, DELETE, LOGIN, LOGOUT)
- Entity type + ID
- Details (what changed)
- IP address, user agent, timestamp

**HIPAA requirement:** No PHI in logs. Log entity IDs and field names, never field values containing PHI.

---

## 5. Environment Variables

```
# Database
DATABASE_URL              # RDS Postgres connection string

# Auth
JWT_SECRET                # JWT signing key
GOOGLE_CLIENT_ID          # Google OAuth client ID
GOOGLE_CLIENT_SECRET      # Google OAuth client secret

# Encryption
ENCRYPTION_KEY            # AES-256-GCM primary key (AWS Secrets Manager)
ENCRYPTION_KEY_PREVIOUS   # Previous key (during rotation only)

# External APIs
META_ACCESS_TOKEN         # Meta system user token (never-expire)
META_BUSINESS_ID          # 819717796169771
ANTHROPIC_API_KEY         # Claude API key for Hope AI
SLACK_BOT_TOKEN           # Slack bot OAuth token (xoxb-)
MAPBOX_TOKEN              # Mapbox GL JS token

# GHL
GHL_PRIVATE_TOKEN         # GHL Private Integration token (rotate every 90 days)

# File Storage
AWS_S3_BUCKET             # S3 bucket for file uploads
AWS_ACCESS_KEY_ID         # (or use IAM roles on ECS)
AWS_SECRET_ACCESS_KEY

# Background Jobs
REDIS_URL                 # Redis for Sidekiq

# Google Sheets (transitional)
GOOGLE_SHEETS_CREDENTIALS # Service account for eval tracker sync
```

**All secrets via AWS Secrets Manager.** Never plaintext env vars in production.

---

## 6. Migration Checklist

### Phase 1: Foundation (Week 1-2)
- [ ] Organizations, clinics, users, pods tables in Flywheel RDS
- [ ] JWT auth (Google OAuth + email/password)
- [ ] RLS policies on all tables
- [ ] Audit logging middleware
- [ ] Encryption setup (Active Record Encryption)

### Phase 2: Data Consolidation (Week 3-4)
- [ ] Import Maryam's Dashboard Postgres -> Flywheel RDS (ghl_contacts, ghl_appointments, ghl_calls, field_mappings, clinics, users)
- [ ] Import Internal app clinics data (retainers, pod assignments, clinic details)
- [ ] Migrate GHL sync jobs to Sidekiq (webhook receiver, nightly reconciliation)
- [ ] Migrate Meta sync jobs to Sidekiq (daily, intraday)
- [ ] Set up eval tracker Sheet sync as Sidekiq job

### Phase 3: Materialized Views + Intelligence (Week 5-6)
- [ ] Build mv_contact_funnel (port Sebastian's contact_funnel SQL)
- [ ] Build mv_cc_daily (port Sebastian's cc_daily SQL)
- [ ] Build mv_evaluations_norm (port Sebastian's evaluations_norm normalization)
- [ ] Build mv_ad_performance_daily
- [ ] Build mv_clinic_spend_daily with hybrid join logic

### Phase 4: Frontend Migration (Week 7-8)
- [ ] Migrate React components from Dashboard into Flywheel ops app
- [ ] Wire to new Rails API endpoints
- [ ] Theme system, sidebar, clinic switcher
- [ ] Patient funnel pages (7 data views)
- [ ] Meta Ads drill-down

### Phase 5: Intelligence Dashboards (Week 9-10)
- [ ] Media Buyer Command Center (/mb)
- [ ] CC Report (/cc)
- [ ] Client Performance Report
- [ ] Top Ads
- [ ] Heatmap
- [ ] Pricing Calculator

### Phase 6: Financial Ops + Advanced (Week 11-12)
- [ ] Revenue, expenses, upsells, contractors tables + UI
- [ ] CSV upload pipeline
- [ ] Leaderboard (expanded)
- [ ] Accounts matcher
- [ ] Onboarding wizard
- [ ] Hope AI chat migration
