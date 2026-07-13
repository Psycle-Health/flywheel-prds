# Feature 02 -- Clinics, Orgs & Pods

**Dependencies:** 01-auth-users
**Source:** Dashboard prod (clinics table), Internal app (enriched clinics + orgs + pods), Psycle-Data (clinic_lookup, client_master), Jun 26 "Clinics Tables" meeting

**Critical fix:** Client ID (org) MUST be separated from Location ID (clinic). Old schema conflated them, blocking multi-location aggregation.

---

## Schema

```sql
CREATE TABLE organizations (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  clinic_npi TEXT,                        -- facility-level NPI (org-wide)
  tax_id TEXT,                            -- EIN
  legal_entity_name TEXT,                 -- business name as declared in EIN
  business_type TEXT,                     -- LLC, Corporation, Sole Proprietorship, etc.
  contract_type TEXT DEFAULT 'agency',    -- agency, platform (from client_master)
  settings JSONB DEFAULT '{}',
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE clinics (
  id TEXT PRIMARY KEY,                    -- = GHL location ID (NEVER generate separate)
  org_id TEXT REFERENCES organizations(id),
  name TEXT NOT NULL,
  slug TEXT,
  address TEXT, city TEXT, state TEXT, zip TEXT,
  phone TEXT, email TEXT, website TEXT,
  timezone TEXT DEFAULT 'America/Denver',
  clinic_npi TEXT,                        -- location-level NPI (if different from org)
  emr_type TEXT,                          -- osmind, tebra, athena, drchrono, advanced_md, intakeq, other
  latitude NUMERIC(10,7),                 -- for Mapbox
  longitude NUMERIC(10,7),
  settings JSONB DEFAULT '{}',           -- Slack channel IDs, treatment modalities, capacity,
                                          -- screening questions, eval cost, deposit policies
  is_active BOOLEAN DEFAULT true,
  onboarding_completed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE providers (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  user_id TEXT REFERENCES users(id),      -- nullable (provider may not be an app user)
  name TEXT NOT NULL,
  email TEXT,
  npi_number TEXT,
  provider_type TEXT,                     -- MD/DO~Psychiatry, PMHNP, NP, CRNA, PA, etc.
  bio TEXT,
  availability TEXT,                      -- "M-W: 9-3, F: 11-3"
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Providers can serve multiple locations
CREATE TABLE provider_clinics (
  provider_id TEXT NOT NULL REFERENCES providers(id) ON DELETE CASCADE,
  clinic_id TEXT NOT NULL REFERENCES clinics(id) ON DELETE CASCADE,
  PRIMARY KEY (provider_id, clinic_id)
);

-- Insurances are PROVIDER-scoped (Jun 26 meeting: "insurances are provider-specific")
CREATE TABLE provider_insurances (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  provider_id TEXT NOT NULL REFERENCES providers(id) ON DELETE CASCADE,
  insurance_carrier TEXT NOT NULL,
  insurance_plan TEXT,
  is_in_network BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE pods (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  name TEXT NOT NULL,                     -- Glutamates, Heavyweights, Catalyst, Pea Pod
  color TEXT,                             -- for UI badge coloring
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE pod_members (
  pod_id TEXT NOT NULL REFERENCES pods(id) ON DELETE CASCADE,
  user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  PRIMARY KEY (pod_id, user_id)
);

CREATE TABLE pod_clinics (
  pod_id TEXT NOT NULL REFERENCES pods(id) ON DELETE CASCADE,
  clinic_id TEXT NOT NULL REFERENCES clinics(id) ON DELETE CASCADE,
  PRIMARY KEY (pod_id, clinic_id)
);

-- Clinic-specific custom fields (for post-eval data clinics can customize)
CREATE TABLE clinic_custom_fields (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  clinic_id TEXT NOT NULL REFERENCES clinics(id),
  field_name TEXT NOT NULL,
  field_type TEXT NOT NULL,               -- text, dropdown, date, number, boolean
  dropdown_options JSONB,                 -- for dropdown type
  display_order INTEGER DEFAULT 0,
  table_target TEXT NOT NULL,             -- evaluations, prior_authorizations, treatments
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE clinic_custom_field_values (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  custom_field_id TEXT NOT NULL REFERENCES clinic_custom_fields(id) ON DELETE CASCADE,
  record_id TEXT NOT NULL,                -- the evaluation/PA/treatment ID
  record_type TEXT NOT NULL,              -- evaluations, prior_authorizations, treatments
  value TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(custom_field_id, record_id, record_type)
);

-- Activity log per clinic (all changes tracked)
CREATE TABLE clinic_activity_log (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  clinic_id TEXT NOT NULL REFERENCES clinics(id),
  event_type TEXT NOT NULL,               -- location_added, location_removed, goal_updated,
                                          -- retainer_changed, onboarding_completed, etc.
  description TEXT,
  details JSONB DEFAULT '{}',
  performed_by TEXT REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## API Endpoints

```
-- Organizations
GET    /api/v1/organizations              -- List orgs (admin only)
POST   /api/v1/organizations              -- Create org
GET    /api/v1/organizations/:id          -- Org details + its clinics
PATCH  /api/v1/organizations/:id          -- Update org

-- Clinics
GET    /api/v1/clinics                    -- All clinics (scoped by RLS/pod)
GET    /api/v1/clinics/:id                -- Clinic detail
PATCH  /api/v1/clinics/:id                -- Update clinic settings
GET    /api/v1/clinics/:id/activity       -- Activity log

-- Providers
GET    /api/v1/clinics/:id/providers      -- Providers at this clinic
POST   /api/v1/clinics/:id/providers      -- Add provider
PATCH  /api/v1/providers/:id              -- Update provider
GET    /api/v1/providers/:id/insurances   -- Provider's insurance list
POST   /api/v1/providers/:id/insurances   -- Add insurance

-- Pods
GET    /api/v1/pods                       -- List pods with member counts
GET    /api/v1/pods/:id                   -- Pod detail with members + clinics
PATCH  /api/v1/pods/:id                   -- Update pod
POST   /api/v1/pods/:id/members           -- Add member
POST   /api/v1/pods/:id/clinics           -- Assign clinic to pod

-- Custom Fields
GET    /api/v1/clinics/:id/custom-fields  -- List custom fields for clinic
POST   /api/v1/clinics/:id/custom-fields  -- Create custom field
PATCH  /api/v1/custom-fields/:id          -- Update custom field
```

## UI Components

- **Clinic Switcher** (top nav): Admin sees all + "All Locations". AM/CC see pod clinics. Clinic users see assigned clinic(s). Selection scopes all API calls.
- **Org management page** (admin only): list orgs, click to see clinics under each
- **Clinic detail page**: settings form with all fields above
- **Provider management**: list per clinic, add/edit with NPI + insurance acceptance
- **Pod management** (admin only): assign users + clinics to pods

## Data Migration

**From Dashboard prod Postgres:**
- `clinics` table -> map directly (same GHL location ID PKs)
- `organizations` -> create from unique org groupings

**From Internal app:**
- Enriched clinic data (city, state, retainers, pod assignments, Mapbox coordinates)
- Pod structure (Glutamates, Heavyweights, Catalyst, Pea Pod)

**From Psycle-Data BigQuery:**
- `ghl.clinic_lookup` -> reconcile clinic_label / pod / region mappings
- `ghl.client_master` -> contract_type, retainer, ad budgets, eval targets
- `ghl.dim_client` -> NPI, tax_id, legal_entity, CH flag

**From Dashboard prototype:**
- Onboarding form field definitions (services offered, capacity, screening questions) -> clinic settings JSONB
