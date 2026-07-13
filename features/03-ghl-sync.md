# Feature 03 -- GHL Sync Pipeline

**Dependencies:** 01-auth-users, 02-clinics-orgs-pods
**Source:** Dashboard prod (webhook + nightly reconcile + backfill), Psycle-Data (9 BQ sync jobs, ghl-sync service, ghl-webhook-receiver)
**Users:** All (data flows through this to power everything else)

---

## Schema

```sql
-- Raw GHL contact storage (single source of truth for GHL data)
CREATE TABLE ghl_contacts (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  ghl_contact_id TEXT UNIQUE NOT NULL,
  clinic_id TEXT REFERENCES clinics(id),
  location_id TEXT NOT NULL,
  raw_payload JSONB NOT NULL,             -- full GHL contact JSON, never modified
  sync_source TEXT DEFAULT 'backfill',    -- backfill, webhook, reconciliation
  first_synced_at TIMESTAMPTZ DEFAULT NOW(),
  last_synced_at TIMESTAMPTZ DEFAULT NOW(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_ghl_contacts_clinic ON ghl_contacts(clinic_id);
CREATE INDEX idx_ghl_contacts_location ON ghl_contacts(location_id);
CREATE INDEX idx_ghl_contacts_synced ON ghl_contacts(last_synced_at);
CREATE INDEX idx_ghl_contacts_payload ON ghl_contacts USING GIN(raw_payload);
CREATE INDEX idx_ghl_contacts_email ON ghl_contacts((raw_payload->>'email'))
  WHERE raw_payload->>'email' IS NOT NULL;
CREATE INDEX idx_ghl_contacts_date_added ON ghl_contacts((raw_payload->>'dateAdded'));

CREATE TABLE ghl_appointments (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  ghl_appointment_id TEXT UNIQUE NOT NULL,
  ghl_contact_id TEXT REFERENCES ghl_contacts(ghl_contact_id),
  clinic_id TEXT REFERENCES clinics(id),
  location_id TEXT NOT NULL,
  calendar_id TEXT,
  appointment_status TEXT,
  start_time TIMESTAMPTZ,
  end_time TIMESTAMPTZ,
  booked_by TEXT,                         -- CC name (used for CC attribution)
  raw_payload JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE ghl_calls (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  ghl_contact_id TEXT REFERENCES ghl_contacts(ghl_contact_id),
  clinic_id TEXT REFERENCES clinics(id),
  call_direction TEXT,                    -- inbound, outbound
  call_status TEXT,                       -- completed, missed, voicemail, no_answer
  call_duration_seconds INTEGER,
  call_user_id TEXT,                      -- GHL user ID = the CC
  call_user_name TEXT,
  call_date TIMESTAMPTZ,
  raw_payload JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE ghl_messages (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  ghl_contact_id TEXT REFERENCES ghl_contacts(ghl_contact_id),
  clinic_id TEXT REFERENCES clinics(id),
  message_type TEXT,                      -- SMS, EMAIL
  direction TEXT,                         -- inbound, outbound
  user_id TEXT,                           -- CC who sent
  user_name TEXT,
  event_timestamp TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Per-clinic field mappings (GHL custom field ID -> app field)
CREATE TABLE field_mappings (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  clinic_id TEXT NOT NULL REFERENCES clinics(id),
  data_source_type TEXT NOT NULL DEFAULT 'ghl',
  app_field TEXT NOT NULL,                -- 'evaluation_status', 'insurance_status', etc.
  app_table TEXT NOT NULL,                -- 'evaluations', 'insurance', 'leads'
  source_field_id TEXT,                   -- GHL custom field ID
  source_field_key TEXT,
  source_field_name TEXT,                 -- human-readable
  source TEXT NOT NULL DEFAULT 'custom',  -- 'native' or 'custom'
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(clinic_id, data_source_type, app_field)
);

-- CC name normalization (GHL user names are inconsistent)
CREATE TABLE cc_alias_map (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  raw_name TEXT NOT NULL,                 -- "johnny", "John S."
  canonical_name TEXT NOT NULL,           -- "John Smith"
  user_id TEXT REFERENCES users(id),
  clinic_id TEXT REFERENCES clinics(id),  -- some aliases are clinic-specific
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Sync run tracking
CREATE TABLE sync_logs (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  clinic_id TEXT NOT NULL REFERENCES clinics(id),
  data_source_type TEXT NOT NULL,         -- ghl_contacts, ghl_appointments, ghl_calls, etc.
  sync_type TEXT NOT NULL,                -- full, incremental, webhook, backfill
  status TEXT NOT NULL DEFAULT 'running', -- running, completed, failed
  records_synced INTEGER DEFAULT 0,
  records_failed INTEGER DEFAULT 0,
  error_message TEXT,
  started_at TIMESTAMPTZ DEFAULT NOW(),
  completed_at TIMESTAMPTZ
);
```

## Sync Modes

### 1. Real-Time Webhook (from Dashboard prod)
- **Trigger:** GHL sends `ContactUpdate` webhook to `POST /api/v1/webhooks/ghl`
- **Action:** Verify webhook signature -> extract contactId + locationId -> fetch FULL contact via GHL API (webhook payload may be partial) -> upsert `ghl_contacts`
- **Rate limit:** Bottleneck (8 concurrent GHL API calls)
- **DLQ:** Failed webhooks queued in Sidekiq retry (3 attempts, exponential backoff)

### 2. Nightly Reconciliation (from Dashboard prod)
- **Schedule:** Sidekiq cron, 2:00 AM PT daily
- **Action:** For each active clinic: query GHL for contacts with `updatedAt > last_sync` -> fetch + upsert each
- **Catches:** Missed webhooks, GHL-side changes that didn't trigger webhooks

### 3. Initial Backfill (from Dashboard prod)
- **Trigger:** On-demand when connecting a new clinic
- **Action:** Paginate all contacts for the location -> upsert all -> SSE progress streaming to UI
- **Also backfills:** appointments, calls for the clinic

### 4. Appointments Sync
- **Schedule:** Sidekiq cron, nightly 2:00 AM PT
- **Action:** Fetch appointments per clinic, upsert `ghl_appointments`

### 5. Calls/Conversations Sync
- **Schedule:** Sidekiq cron, hourly
- **Action:** Fetch conversations per clinic, upsert `ghl_calls` + `ghl_messages`

## GHL API Access Pattern

```
Agency Token (env: GHL_PRIVATE_TOKEN, rotates every 90 days)
  └─> GET /locations (list all agency locations)
      └─> Exchange for location access token (valid 1 day, refresh valid 1 year)
          └─> GET /contacts (per location)
          └─> GET /contacts/:id (full contact with customFields)
          └─> GET /calendars/appointments (per location)
          └─> GET /conversations (per location)
```

**Required scopes:** contacts.readonly, contacts/customField.readonly, calendars/appointments.readonly, locations.readonly, oauth

## API Endpoints

```
-- Sync management
GET    /api/v1/clinic/:clinicId/sync/status    -- GHL config + last sync status
POST   /api/v1/clinic/:clinicId/sync/start     -- Trigger manual re-sync
GET    /api/v1/clinic/:clinicId/sync/stream    -- SSE progress during sync
POST   /api/v1/install/stream                  -- Initial GHL setup (OAuth -> backfill)

-- Field mappings
GET    /api/v1/clinic/:clinicId/field-mappings  -- Active mappings
PUT    /api/v1/clinic/:clinicId/field-mappings  -- Save mappings
GET    /api/v1/clinic/:clinicId/ghl-fields      -- Available GHL custom fields (for dropdown)

-- Webhooks
POST   /api/v1/webhooks/ghl                    -- GHL webhook receiver (ContactUpdate, etc.)
```

## Key Design Decisions

1. **Raw JSONB storage:** Store full GHL payloads. Extract fields at query time via `field_mappings`. No schema changes when GHL adds/changes fields.
2. **Per-clinic field mappings:** Each clinic has different GHL custom field IDs for the same logical fields. The mapping table translates.
3. **Non-fatal sync:** If appointments or calls fail, contact data is preserved.
4. **Never create GHL custom fields** programmatically. Read-only mapping to existing fields.
5. **Preferred Location** field is read-only -- never create it, only map to it.
6. **Always fetch full contact** on webhook (not just webhook payload) because payload may be partial.

## Data Migration

**From Dashboard prod Postgres:**
- `ghl_contacts`, `ghl_appointments`, `field_mappings`, `sync_logs` -> direct migration
- `ghl_calls` -> migrate (Dashboard calls this table `calls` with slightly different columns)

**From Psycle-Data BigQuery (reference, not migration):**
- Sebastian's 5 GHL sync jobs write to BQ. Once Flywheel's sync is running, these BQ jobs can be retired or pointed at Flywheel's Postgres instead.
- `ghl.clinic_lookup` -> reconcile clinic name/label mappings with Flywheel's clinics table
- `cc_alias_map` -> port Sebastian's CC name resolution logic
