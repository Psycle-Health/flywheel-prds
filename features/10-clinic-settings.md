# Feature 10 -- Clinic Settings & Accounts

**Dependencies:** 02-clinics-orgs-pods
**Source:** Dashboard prod (settings pages), prototype (accounts matcher mockup)
**Users:** Admin (all settings + accounts matcher), AM (pod clinic settings), Clinic Admin (own clinic settings)

---

## Settings Pages

### Clinic Details (Settings > Clinic Details)

**Fields:**
- Clinic Name, Clinic NPI, Address/City/State/Zip, Phone, Timezone
- GHL Location ID (read-only display)
- GHL Sync Status badge (synced / not synced / syncing / error) + Last Sync timestamp
- EMR Type dropdown (Osmind, Tebra, Athena, DrChrono, Advanced MD, IntakeQ, Other)
- Treatment Modalities multi-select (Spravato, Ketamine, TMS, Med Mgmt)

**Preferred Location Field Mapping:**
- Dropdown populated from clinic's GHL custom fields via `GET /api/v1/clinic/:id/ghl-fields`
- User selects which GHL field = "Preferred Location"
- Saved via `PUT /api/v1/clinic/:id/field-mappings`
- When mapped, value appears as column on all data tables
- READ-ONLY from GHL. NEVER create this field.
- Unmapped = single-location clinic

**Integration Settings:**
- Meta Ad Account ID (`act_XXXXXXXXX`)
- Slack External Channel ID (Psycle channel)
- Slack Internal Channel ID (Staff channel)
- EMR Connection status (from Flywheel EmrConnection)

### Insurance Details (Settings > Insurance Details)

- Per-provider insurance acceptance list
- Add/remove carrier + plan per provider
- Carrier name normalization (editable `insurance_carrier_map`)

### Team (Settings > Team)

- User list for this clinic (from user_clinics junction)
- Invite new user: email, name, role (clinic_admin/clinic_team), user type (provider/billing/manager/owner)
- Provider sub-fields: provider_type dropdown, NPI number (appear only when user type = provider)
- Permission toggles per section (gradient-backed toggle buttons)
- Only Admin can edit toggles; AM + Clinic Admin see read-only
- Deactivate/remove user

### Custom Fields (Settings > Custom Fields)

- List custom fields for this clinic
- Add new: field_name, field_type (text/dropdown/date/number/boolean), dropdown_options, table_target (evaluations/prior_auths/treatments)
- Reorder via drag or display_order
- Deactivate (soft delete -- preserves existing data)

## Accounts Matcher (Settings > Accounts) -- Admin Only

**Source:** Dashboard prototype `mockup-accounts.html`

Maps: Organization -> GHL Location -> Meta Ad Account -> Campaigns -> Result Type

### UI Pattern

- **Org cards** with expand/collapse caret
- **Single-region orgs** (e.g., Emerge): mapping fields directly on org row
- **Multi-region orgs** (e.g., Headlight 2 regions, Nao 6 regions): indented subaccount rows with lavender left accent bar

### Per-Mapping Fields

| Field | Source | UI |
|-------|--------|-----|
| GHL Location | GHL Agency API (all locations) | Dropdown |
| Meta Ad Account | Meta API (all active accounts) | Dropdown |
| Campaigns | Campaigns in selected ad account | Multi-select, shown as lavender pills. Empty = "All Campaigns" |
| Result Type | Configurable | Dropdown: Lead (Form), View Content |

### Actions
- `+ Add Organization` (header CTA) -- creates new org, opens inline edit
- `+ Subaccount` (per org) -- converts to multi-region if single
- Edit (org name inline)
- Rename / Remove per subaccount (Remove = red danger button with confirmation)

### Collapsed Org View
Summary line: ad account ID, campaign count or "All", GHL location count

### API
```
GET    /api/v1/settings/accounts          -- All orgs with their mappings
POST   /api/v1/settings/accounts          -- Create org + initial mapping
PATCH  /api/v1/settings/accounts/:orgId   -- Update org
POST   /api/v1/settings/accounts/:orgId/subaccounts    -- Add subaccount
PATCH  /api/v1/settings/accounts/:orgId/subaccounts/:subId  -- Update mapping
DELETE /api/v1/settings/accounts/:orgId/subaccounts/:subId  -- Remove subaccount
```

## API Endpoints (Settings)

```
GET    /api/v1/clinics/:id/settings       -- Full settings for clinic
PATCH  /api/v1/clinics/:id/settings       -- Update settings
GET    /api/v1/clinics/:id/field-mappings  -- GHL field mappings
PUT    /api/v1/clinics/:id/field-mappings  -- Save field mappings
GET    /api/v1/clinics/:id/ghl-fields      -- Available GHL custom fields (for dropdowns)
GET    /api/v1/clinics/:id/custom-fields   -- Custom field definitions
POST   /api/v1/clinics/:id/custom-fields   -- Create custom field
PATCH  /api/v1/custom-fields/:id           -- Update custom field
```
