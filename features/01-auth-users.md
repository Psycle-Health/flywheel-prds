# Feature 01 -- Auth & Users

**Dependencies:** None (foundation)
**Source:** Dashboard prod (JWT + Google OAuth + email/password), Internal app (Google OAuth + pod-based access)

---

## Schema

```sql
CREATE TABLE users (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  email TEXT UNIQUE NOT NULL,
  name TEXT,
  password_hash TEXT,                     -- bcrypt (null for Google OAuth only)
  avatar_url TEXT,
  is_internal BOOLEAN DEFAULT false,      -- true = Psycle employee

  -- Internal users (Psycle team)
  psycle_user_type TEXT,                  -- admin, growth, am, cc, content (null for external)

  -- External users (clinic staff)
  role TEXT,                              -- clinic_admin, clinic_team (null for internal)
  clinic_user_type TEXT,                  -- provider, billing, manager, owner (null for internal)
  provider_type TEXT,                     -- null unless clinic_user_type = 'provider'
  npi_number TEXT,                        -- null unless clinic_user_type = 'provider'

  org_id TEXT REFERENCES organizations(id),
  employment_type TEXT,                   -- salary, hourly, w9_salary, w9_inv
  work_location TEXT,                     -- in-person, remote, hybrid
  is_active BOOLEAN DEFAULT true,
  last_login_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE user_clinics (
  user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  clinic_id TEXT NOT NULL REFERENCES clinics(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (user_id, clinic_id)
);

CREATE TABLE user_permissions (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  section TEXT NOT NULL,                  -- ads, leads, calls, consults, insurances,
                                          -- new_patients, prior_auths, treatments,
                                          -- mb, cc_report, topads, agency, heatmap,
                                          -- client_report, settings
  can_view BOOLEAN DEFAULT false,
  can_edit BOOLEAN DEFAULT false,
  UNIQUE(user_id, section)
);

CREATE TABLE audit_logs (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  timestamp TIMESTAMPTZ DEFAULT NOW(),
  user_id TEXT REFERENCES users(id),
  user_email TEXT,
  action TEXT NOT NULL,                   -- VIEW, CREATE, UPDATE, DELETE, LOGIN, LOGOUT
  entity_type TEXT,
  entity_id TEXT,
  details JSONB DEFAULT '{}',
  ip_address TEXT,
  user_agent TEXT
);
```

## Auth Flow

**Google OAuth** (Psycle team -- `@psyclehealth.com`):
1. Frontend redirects to Google consent screen
2. Google returns auth code -> `POST /api/v1/auth/google`
3. Server verifies token, auto-creates user if `@psyclehealth.com` and not in DB
4. Returns app JWT in secure httpOnly cookie

**Email/Password** (clinic users):
1. `POST /api/v1/auth/login` with email + password
2. Server verifies bcrypt hash
3. Returns same app JWT

**JWT claims:**
```json
{
  "sub": "user_id",
  "email": "jake@psyclehealth.com",
  "name": "Jake Inzalaca",
  "is_internal": true,
  "psycle_user_type": "am",
  "pod_ids": ["glutamates"],
  "clinic_ids": ["ZpNsx02KGCBh1bRMRU6K", "abc123"],
  "permissions": { "new_patients": { "view": true, "edit": true } }
}
```

**Session:** 30-min inactivity auto-logout. Secure httpOnly cookies. Token refresh mechanism.

## API Endpoints

```
POST   /api/v1/auth/google          -- Google OAuth login
POST   /api/v1/auth/login           -- Email/password login
POST   /api/v1/auth/logout          -- Invalidate session
POST   /api/v1/auth/refresh         -- Refresh JWT
GET    /api/v1/users                -- List users (admin only)
POST   /api/v1/users                -- Create user (admin + clinic_admin for their clinic)
PATCH  /api/v1/users/:id            -- Update user
DELETE /api/v1/users/:id            -- Deactivate user
GET    /api/v1/users/:id/permissions -- Get permission toggles
PUT    /api/v1/users/:id/permissions -- Set permission toggles (admin only)
```

## UI Components

- Login page (Google OAuth button + email/password form)
- User management table (admin only, in Settings > Team)
- User creation/edit modal with conditional fields:
  - Role/type dropdowns (psycle_user_type or clinic_user_type based on is_internal)
  - Provider sub-fields (provider_type, NPI) appear only when clinic_user_type = provider
  - Clinic access multi-select (or auto-assigned via pod for internal users)
  - Permission toggles (gradient-backed toggle buttons per section)

## Default Permission Dispositions

Applied on user creation, overridable by Admin:

| Section | admin | growth | am | cc | provider | billing | manager | owner |
|---------|-------|--------|----|----|----------|---------|---------|-------|
| ads | view+edit | view+edit | -- | -- | -- | -- | -- | -- |
| leads | view+edit | view | view | view+edit | -- | -- | -- | -- |
| calls | view+edit | -- | view | view | -- | -- | -- | -- |
| consults | view+edit | -- | view | view | -- | -- | -- | -- |
| insurances | view+edit | -- | view+edit | view | view+edit | view+edit | view+edit | view+edit |
| new_patients | view+edit | -- | view+edit | view | view+edit | -- | view+edit | view+edit |
| prior_auths | view+edit | -- | view+edit | -- | view+edit | view+edit | view+edit | view+edit |
| treatments | view+edit | -- | view+edit | -- | view+edit | -- | view+edit | view+edit |

## RLS

```sql
-- Session variables set per-request from JWT:
SET LOCAL app.is_internal = 'true';
SET LOCAL app.user_type = 'am';
SET LOCAL app.user_id = 'user_uuid';

-- Admin: see all
CREATE POLICY admin_access ON [table] FOR ALL
  USING (current_setting('app.is_internal', true) = 'true'
    AND current_setting('app.user_type', true) = 'admin');

-- Internal non-admin: see pod-assigned clinics
CREATE POLICY internal_pod_access ON [table] FOR ALL
  USING (current_setting('app.is_internal', true) = 'true'
    AND clinic_id IN (
      SELECT pc.clinic_id FROM pod_clinics pc
      JOIN pod_members pm ON pm.pod_id = pc.pod_id
      WHERE pm.user_id = current_setting('app.user_id', true)));

-- External: see assigned clinics only
CREATE POLICY clinic_access ON [table] FOR ALL
  USING (current_setting('app.is_internal', true) = 'false'
    AND clinic_id IN (
      SELECT uc.clinic_id FROM user_clinics uc
      WHERE uc.user_id = current_setting('app.user_id', true)));
```

Runtime connection uses `flywheel_app` role (NOT superuser, NOT BYPASSRLS).

## Workflows

- **Auto-create on Google login:** If `@psyclehealth.com` email not in DB, create user with `psycle_user_type = null` (admin must assign type)
- **Invitation flow:** Admin creates user -> system sends invitation email with login link
- **Audit:** Every login/logout logged. Every permission change logged.
