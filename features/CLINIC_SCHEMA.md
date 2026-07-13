# Clinic Schema -- Foundation of the Entire Application

Every feature in Flywheel connects through the clinic. This schema merges data from:
- **Internal app** clinics table (retainers, pods, Mapbox, revenue)
- **Onboarding form** (5-step wizard fields)
- **Dashboard prod** schema-v2.sql (GHL sync, settings JSONB)
- **Psycle-Data** BigQuery (clinic_lookup, client_master, dim_client)
- **Flywheel** existing models (EmrConnection, providers)

---

## 1. Organizations Table (the client/business entity)

```sql
CREATE TABLE organizations (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,

  -- Identity
  name TEXT NOT NULL,                         -- "Headlight Health" (trade name)
  slug TEXT UNIQUE NOT NULL,                  -- URL-friendly: "headlight-health"
  legal_entity_name TEXT,                     -- "Headlight Health LLC" (as declared in EIN)
  business_type TEXT,                         -- Sole Proprietorship, Partnership, Corporation,
                                              -- Co-Operative, LLC, Non-Profit

  -- Tax & Compliance
  tax_id TEXT,                                -- EIN (XX-XXXXXXX)
  cp575_file_url TEXT,                        -- S3 path to uploaded IRS CP 575 document
  clinic_npi TEXT,                            -- Facility-level NPI (org-wide)

  -- BAA
  baa_signer_name TEXT,
  baa_signer_title TEXT,
  baa_signed_at TIMESTAMPTZ,
  baa_signature_url TEXT,                     -- S3 path to signature image
  baa_acknowledged BOOLEAN DEFAULT false,

  -- Commercial Terms (from Psycle-Data client_master + Internal app)
  contract_type TEXT DEFAULT 'agency',        -- agency, platform
  plan TEXT DEFAULT 'starter',                -- marketplace billing tier (future)

  -- Contact
  primary_contact_name TEXT,
  primary_contact_title TEXT,
  primary_contact_email TEXT,
  primary_contact_phone TEXT,

  -- Meta
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 2. Clinics Table (one per physical location = one GHL subaccount)

```sql
CREATE TABLE clinics (
  id TEXT PRIMARY KEY,                        -- = GHL location ID (NEVER generate separate)
  org_id TEXT REFERENCES organizations(id),

  -- Identity
  name TEXT NOT NULL,                         -- "Headlight CA" or "Emerge Ketamine (Woburn, MA)"
  slug TEXT,                                  -- URL-friendly
  phone TEXT,
  email TEXT,
  website TEXT,

  -- Location
  address TEXT,
  city TEXT,
  state TEXT,
  zip TEXT,
  latitude NUMERIC(10,7),                     -- Mapbox pin
  longitude NUMERIC(10,7),                    -- Mapbox pin

  -- Hours & Timezone
  clinic_hours TEXT,                          -- "M - F, 9am - 5pm PST"
  timezone TEXT DEFAULT 'America/Denver',

  -- NPI (location-level, may differ from org-level)
  clinic_npi TEXT,

  -- Services Offered (multi-select from onboarding)
  services_offered JSONB DEFAULT '[]',        -- ["IV Ketamine","Spravato","TMS","IM Ketamine",
                                              --  "Medication Management","Therapy",
                                              --  "Accelerated TMS","IOP","PHP"]

  -- Capacity Per Modality (from onboarding Step 1)
  spravato_chairs INTEGER,
  spravato_sessions_per_day INTEGER,
  spravato_days_per_week INTEGER,
  tms_chairs INTEGER,
  tms_sessions_per_day INTEGER,
  tms_days_per_week INTEGER,

  -- EMR (from onboarding Step 4)
  emr_type TEXT,                              -- osmind, tebra, athena, drchrono,
                                              -- advanced_md, intakeq, other
  emr_appointment_reminders TEXT,             -- yes_sms, yes_email, yes_both, no
  evaluation_locations TEXT,                  -- "In-person, telehealth, etc."
  intake_paperwork_method TEXT,               -- "EMR, PandaDocs, DocuSign, etc."

  -- Costs & Policies (from onboarding Step 4)
  eval_cost TEXT,                             -- "$50"
  cash_pay_financing TEXT,                    -- HSA/FSA, Care Credit, Advance Care Card, etc.
  cash_pay_pricing TEXT,                      -- "6 infusions for $2400..."
  deposit_credited BOOLEAN,                   -- credited toward treatment?
  deposit_refundable BOOLEAN,
  current_monthly_evals INTEGER,              -- baseline at onboarding
  cancellation_policy TEXT,

  -- Spravato-Specific (from onboarding Step 5)
  spravato_offered TEXT,                      -- buy_and_bill_and_pharmacy, pharmacy_only,
                                              -- buy_and_bill_only, no, coming_soon
  spravato_avg_reimbursement TEXT,            -- "Net after med costs"
  spravato_avg_duration TEXT,                 -- <2mo, 2-3mo, 3-6mo, 6-12mo, 12+mo
  spravato_time_to_start TEXT,               -- <=1wk, 1-2wk, 2-4wk, 4+wk
  spravato_with_me BOOLEAN,                   -- enrolled in Spravato with Me?
  janssen_savings_program BOOLEAN,            -- enrolled in Janssen Savings?

  -- TMS-Specific (from onboarding Step 5)
  tms_offered TEXT,                           -- yes, no, coming_soon
  tms_completion_rate TEXT,                   -- 90%+, 75-90%, 50-75%, <50%
  tms_avg_reimbursement TEXT,
  tms_time_to_start TEXT,                     -- <=1wk, 1-2wk, 2-4wk, 4+wk

  -- Care Coordination Screening (from Jun 26 Clinics Tables meeting)
  screening_min_med_trials INTEGER,           -- # of failed med trials required
  screening_depression_diagnosis_required BOOLEAN,
  screening_contraindications JSONB DEFAULT '[]', -- multi-select: bipolar, psychosis, seizures, etc.

  -- PA & Insurance Operations (from onboarding-fields.js)
  pa_approval_rate TEXT,                      -- >75%, 50-75%, <50%, not_sure
  pa_turnaround TEXT,                         -- <48h, 48h-5d, 5-10d, 10d+

  -- Commercial Terms (from Psycle-Data client_master / Internal app)
  retainer_amount NUMERIC(12,2),              -- current monthly retainer
  ad_budget_target NUMERIC(12,2),             -- monthly Meta ad budget
  eval_target INTEGER,                        -- monthly eval goal
  is_bundled BOOLEAN DEFAULT false,           -- bundled/platform pricing model
  bundled_transition_date DATE,               -- pricing model applies from this date

  -- Integration IDs
  meta_ad_account_id TEXT,                    -- act_XXXXXXXXX
  slack_external_channel_id TEXT,             -- Psycle-facing Slack channel
  slack_internal_channel_id TEXT,             -- Staff-facing Slack channel
  ghl_preferred_location_field_id TEXT,       -- mapped GHL custom field for Preferred Location

  -- Connective Health (from Psycle-Data dim_client)
  ch_enabled BOOLEAN DEFAULT false,           -- Connective Health integration active?
  ch_practice_oid TEXT,                       -- CH practice identifier
  ch_api_key_secret_ref TEXT,                 -- AWS Secrets Manager reference (NEVER plaintext)

  -- Uploaded Documents (S3 paths from onboarding)
  paperwork_file_urls JSONB DEFAULT '[]',     -- clinic intake paperwork PDFs
  consent_file_urls JSONB DEFAULT '[]',       -- clinic consent form PDFs

  -- Marketing Assets (from onboarding-fields.js)
  marketing_assets JSONB DEFAULT '[]',        -- ["Facebook Page","Instagram Page","Meta Ad Account",
                                              --  "Google My Business","Google Analytics", etc.]

  -- Sync Status
  ghl_last_synced_at TIMESTAMPTZ,
  ghl_sync_status TEXT DEFAULT 'not_synced',  -- not_synced, syncing, synced, error

  -- Lifecycle
  is_active BOOLEAN DEFAULT true,
  onboarding_completed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_clinics_org ON clinics(org_id);
CREATE INDEX idx_clinics_state ON clinics(state);
CREATE INDEX idx_clinics_active ON clinics(is_active) WHERE is_active = true;
```

---

## 3. Related Tables

### Providers (per-clinic evaluating providers)

```sql
CREATE TABLE providers (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  user_id TEXT REFERENCES users(id),          -- nullable (provider may not be app user)
  name TEXT NOT NULL,
  email TEXT,
  npi_number TEXT,                            -- 10-digit National Provider Identifier
  provider_type TEXT,                         -- MD/DO~Psychiatry, MD/DO~Anesthesiology,
                                              -- MD/DO~Neurology, MD/DO~Pain Management,
                                              -- MD/DO~Emergency Medicine, PMHNP, NP, CRNA, PA, Other
  bio TEXT,                                   -- from onboarding Step 3
  availability TEXT,                          -- "M-W: 9-3, F: 11-3"
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

-- Insurances are PROVIDER-scoped (Jun 26 meeting: provider-specific, not clinic-wide)
CREATE TABLE provider_insurances (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  provider_id TEXT NOT NULL REFERENCES providers(id) ON DELETE CASCADE,
  insurance_carrier TEXT NOT NULL,            -- from INSURANCE_CARRIERS list (500+ options)
  insurance_plan TEXT,
  is_in_network BOOLEAN DEFAULT true,
  is_credentialed BOOLEAN DEFAULT false,      -- from CREDENTIALED_INSURANCE_OPTIONS list
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Ad Account Mappings (the accounts matcher)

```sql
CREATE TABLE ad_account_mappings (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  clinic_id TEXT NOT NULL REFERENCES clinics(id),
  ad_account_id TEXT NOT NULL,                -- Meta ad account (act_XXXXXXXXX)
  campaign_filter JSONB,                      -- array of campaign IDs, null = all
  result_type TEXT DEFAULT 'lead',            -- lead, view_content
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Pods (team groupings)

```sql
CREATE TABLE pods (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  name TEXT NOT NULL,                         -- Glutamates, Heavyweights, Catalyst, Pea Pod
  color TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE pod_clinics (
  pod_id TEXT NOT NULL REFERENCES pods(id) ON DELETE CASCADE,
  clinic_id TEXT NOT NULL REFERENCES clinics(id) ON DELETE CASCADE,
  PRIMARY KEY (pod_id, clinic_id)
);
```

### Custom Fields (clinic-specific columns for post-eval data)

```sql
CREATE TABLE clinic_custom_fields (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  clinic_id TEXT NOT NULL REFERENCES clinics(id),
  field_name TEXT NOT NULL,
  field_type TEXT NOT NULL,                   -- text, dropdown, date, number, boolean
  dropdown_options JSONB,                     -- for dropdown type
  display_order INTEGER DEFAULT 0,
  table_target TEXT NOT NULL,                 -- evaluations, prior_authorizations, treatments
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Clinic Activity Log

```sql
CREATE TABLE clinic_activity_log (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  clinic_id TEXT NOT NULL REFERENCES clinics(id),
  event_type TEXT NOT NULL,                   -- location_added, location_removed, goal_updated,
                                              -- retainer_changed, onboarding_completed,
                                              -- provider_added, emr_connected, etc.
  description TEXT,
  details JSONB DEFAULT '{}',
  performed_by TEXT REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 4. Field Inventory (all clinic fields, by source)

| Field | Table.Column | Onboarding Step | Internal App | Psycle-Data BQ | Dashboard Prod |
|-------|-------------|-----------------|-------------|----------------|----------------|
| GHL Location ID | clinics.id (PK) | -- | clinics.id | clinic_lookup.location_id | clinics.id |
| Org ID | clinics.org_id | -- | clinics.org_id | -- | clinics.org_id |
| Clinic Name | clinics.name | Step 1 | clinics.name | clinic_lookup.clinic_label | clinics.name |
| Address | clinics.address | Step 1 (per location) | -- | -- | clinics.address |
| City | clinics.city | Step 1 | clinics.city | -- | clinics.city |
| State | clinics.state | Step 1 | clinics.state | -- | clinics.state |
| ZIP | clinics.zip | Step 1 | -- | -- | clinics.zip |
| Lat/Lng | clinics.latitude/longitude | -- | Mapbox API | -- | -- |
| Phone | clinics.phone | Step 1 | -- | -- | clinics.phone |
| Email | clinics.email | Step 1 | -- | -- | -- |
| Website | clinics.website | Step 1 | -- | -- | -- |
| Hours | clinics.clinic_hours | Step 1 | -- | -- | -- |
| Timezone | clinics.timezone | -- | -- | client_master.tz | clinics.timezone |
| Clinic NPI | clinics.clinic_npi | Step 3 | -- | dim_client.npi | -- |
| Org NPI | organizations.clinic_npi | Step 3 | -- | dim_client.npi | -- |
| Legal Entity | organizations.legal_entity_name | Step 2 | -- | dim_client.legal_entity | -- |
| Tax ID / EIN | organizations.tax_id | Step 2 | -- | dim_client.tax_id | -- |
| Business Type | organizations.business_type | Step 2 | -- | -- | -- |
| CP 575 | organizations.cp575_file_url | Step 2 (upload) | -- | -- | -- |
| BAA Signed | organizations.baa_* | Step 2 | -- | -- | -- |
| Primary Contact | organizations.primary_contact_* | Step 1 | -- | -- | -- |
| Contract Type | organizations.contract_type | -- | -- | client_master.contract_type, dim_client.contract_type | -- |
| Services Offered | clinics.services_offered | Step 1 (multi-select) | -- | -- | settings JSONB |
| SPV Capacity | clinics.spravato_chairs/sessions/days | Step 1 | -- | -- | -- |
| TMS Capacity | clinics.tms_chairs/sessions/days | Step 1 | -- | -- | -- |
| EMR Type | clinics.emr_type | Step 4 | -- | -- | -- |
| EMR Reminders | clinics.emr_appointment_reminders | Step 4 | -- | -- | -- |
| Eval Locations | clinics.evaluation_locations | Step 4 | -- | -- | -- |
| Paperwork Method | clinics.intake_paperwork_method | Step 4 | -- | -- | -- |
| Eval Cost | clinics.eval_cost | Step 4 | -- | -- | -- |
| Cash-Pay Financing | clinics.cash_pay_financing | Step 4 | -- | -- | -- |
| Cash-Pay Pricing | clinics.cash_pay_pricing | Step 4 | -- | -- | -- |
| Deposit Credited | clinics.deposit_credited | Step 4 | -- | -- | -- |
| Deposit Refundable | clinics.deposit_refundable | Step 4 | -- | -- | -- |
| Monthly Evals Baseline | clinics.current_monthly_evals | Step 4 | -- | -- | -- |
| Cancellation Policy | clinics.cancellation_policy | Step 4 | -- | -- | -- |
| Spravato Offered | clinics.spravato_offered | Step 5 | -- | -- | -- |
| Spravato Reimbursement | clinics.spravato_avg_reimbursement | Step 5 | -- | -- | -- |
| Spravato Duration | clinics.spravato_avg_duration | Step 5 | -- | -- | -- |
| Spravato Time to Start | clinics.spravato_time_to_start | Step 5 | -- | -- | -- |
| Spravato with Me | clinics.spravato_with_me | Step 5 | -- | -- | -- |
| Janssen Savings | clinics.janssen_savings_program | Step 5 | -- | -- | -- |
| TMS Offered | clinics.tms_offered | Step 5 | -- | -- | -- |
| TMS Completion Rate | clinics.tms_completion_rate | Step 5 | -- | -- | -- |
| TMS Reimbursement | clinics.tms_avg_reimbursement | Step 5 | -- | -- | -- |
| TMS Time to Start | clinics.tms_time_to_start | Step 5 | -- | -- | -- |
| Screening Min Trials | clinics.screening_min_med_trials | -- (Jun 26 meeting) | -- | -- | -- |
| Screening Depression Req | clinics.screening_depression_diagnosis_required | -- (Jun 26 meeting) | -- | -- | -- |
| Screening Contraindications | clinics.screening_contraindications | -- (Jun 26 meeting) | -- | -- | -- |
| PA Approval Rate | clinics.pa_approval_rate | onboarding-fields.js | -- | -- | -- |
| PA Turnaround | clinics.pa_turnaround | onboarding-fields.js | -- | -- | -- |
| Retainer | clinics.retainer_amount | -- | internal_revenues | client_master.retainer_usd | -- |
| Ad Budget Target | clinics.ad_budget_target | -- | -- | client_master.ad_budget | -- |
| Eval Target (Goal) | clinics.eval_target | -- | -- | client_master.eval_target | -- |
| Is Bundled | clinics.is_bundled | -- | -- | client_master.is_bundled | -- |
| Meta Ad Account | clinics.meta_ad_account_id | -- | -- | -- | settings JSONB |
| Slack External | clinics.slack_external_channel_id | -- | -- | -- | settings JSONB |
| Slack Internal | clinics.slack_internal_channel_id | -- | -- | -- | settings JSONB |
| Preferred Location Field | clinics.ghl_preferred_location_field_id | -- | -- | -- | field_mappings |
| CH Enabled | clinics.ch_enabled | -- | -- | dim_client.ch_flag | -- |
| Marketing Assets | clinics.marketing_assets | onboarding-fields.js | -- | -- | -- |
| Clinic Paperwork | clinics.paperwork_file_urls | Step 4 (upload) | -- | -- | -- |
| Clinic Consents | clinics.consent_file_urls | Step 4 (upload) | -- | -- | -- |
| Pod | pod_clinics | -- | pod_clinics | clinic_lookup.pod | -- |
| Provider Name/NPI/Type/Bio | providers.* | Step 3 | -- | -- | -- |
| Provider Insurances | provider_insurances.* | Step 3 | -- | -- | -- |
| Provider Availability | providers.availability | Step 3 | -- | -- | -- |
| Provider Locations | provider_clinics.* | Step 3 | -- | -- | -- |
