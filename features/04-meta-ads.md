# Feature 04 -- Meta Sync & Ads Dashboard

**Dependencies:** 01-auth-users, 02-clinics-orgs-pods
**Source:** Dashboard prod (Meta API client, drill-down UI, winning ads, geo/heatmaps), Psycle-Data (meta-insights-sync, ad_performance_daily, ad_dim, rejected_ads)
**Users:** Admin, Growth (full access); AM/CC/Clinic hidden

---

## Schema

```sql
CREATE TABLE meta_ad_accounts (
  id TEXT PRIMARY KEY,                    -- Meta account ID (act_XXXXXXXXX)
  name TEXT,
  currency TEXT DEFAULT 'USD',
  status TEXT,                            -- ACTIVE, DISABLED
  last_synced_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE meta_campaigns (
  id TEXT PRIMARY KEY,
  ad_account_id TEXT NOT NULL REFERENCES meta_ad_accounts(id),
  name TEXT, status TEXT, objective TEXT,
  daily_budget NUMERIC(12,2), lifetime_budget NUMERIC(12,2),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE meta_adsets (
  id TEXT PRIMARY KEY,
  campaign_id TEXT NOT NULL REFERENCES meta_campaigns(id),
  name TEXT, status TEXT,
  targeting JSONB,                        -- geo, age, interests
  optimization_goal TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE meta_ads (
  id TEXT PRIMARY KEY,
  adset_id TEXT NOT NULL REFERENCES meta_adsets(id),
  name TEXT, status TEXT, creative_id TEXT,
  preview_url TEXT,
  first_active_date DATE, last_active_date DATE,
  total_spend NUMERIC(12,2) DEFAULT 0,   -- denormalized for fast ranking
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE meta_insights (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  object_id TEXT NOT NULL,                -- ad/adset/campaign/account ID
  object_type TEXT NOT NULL,              -- ad, adset, campaign, account
  date DATE NOT NULL,                     -- PT-bucketed
  impressions INTEGER DEFAULT 0,
  clicks INTEGER DEFAULT 0,
  spend NUMERIC(12,2) DEFAULT 0,
  actions JSONB,                          -- [{action_type: "lead", value: "5"}, ...]
  cost_per_action_type JSONB,             -- [{action_type: "lead", value: "12.50"}, ...]
  ctr NUMERIC(8,4), cpc NUMERIC(8,2), cpm NUMERIC(8,2),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(object_id, object_type, date)
);

CREATE TABLE meta_ad_rejections (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  ad_id TEXT NOT NULL REFERENCES meta_ads(id),
  review_status TEXT,                     -- rejected, approved, appeal_pending
  rejection_reason TEXT,
  detected_at TIMESTAMPTZ,
  resolved_at TIMESTAMPTZ,
  source TEXT DEFAULT 'meta_api',         -- meta_api, gmail_poll, manual
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Maps clinics to their Meta ad accounts + campaign filters
CREATE TABLE ad_account_mappings (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  clinic_id TEXT NOT NULL REFERENCES clinics(id),
  ad_account_id TEXT NOT NULL REFERENCES meta_ad_accounts(id),
  campaign_filter JSONB,                  -- array of campaign IDs, null = all campaigns
  result_type TEXT DEFAULT 'lead',        -- lead, view_content
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

## Materialized View

```sql
-- mv_ad_performance_daily (replaces Sebastian's meta.ad_performance_daily)
-- Grain: 1 row per (ad_id, date)
CREATE MATERIALIZED VIEW mv_ad_performance_daily AS
SELECT
  mi.object_id AS ad_id,
  a.name AS ad_name,
  a.adset_id,
  ads.campaign_id,
  c.ad_account_id,
  aam.clinic_id,
  mi.date,
  mi.spend,
  mi.impressions,
  mi.clicks,
  -- Lead extraction (LOCKED: action_type = 'lead' ONLY)
  (SELECT (elem->>'value')::integer
   FROM jsonb_array_elements(mi.actions) elem
   WHERE elem->>'action_type' = 'lead'
   LIMIT 1) AS leads,
  -- CPL extraction (LOCKED: from cost_per_action_type, NEVER spend/leads)
  (SELECT (elem->>'value')::numeric
   FROM jsonb_array_elements(mi.cost_per_action_type) elem
   WHERE elem->>'action_type' = 'lead'
   LIMIT 1) AS cpl,
  a.first_active_date,
  a.last_active_date
FROM meta_insights mi
JOIN meta_ads a ON a.id = mi.object_id
JOIN meta_adsets ads ON ads.id = a.adset_id
JOIN meta_campaigns c ON c.id = ads.campaign_id
LEFT JOIN ad_account_mappings aam ON aam.ad_account_id = c.ad_account_id AND aam.is_active
WHERE mi.object_type = 'ad';
```

## Sync Jobs (Sidekiq)

| Job | Schedule | Scope | Source |
|-----|----------|-------|--------|
| MetaDailySyncJob | 6:00 AM PT | All accounts: insights (8 days back), campaigns, adsets, ads, creative | Dashboard prod |
| MetaIntradaySyncJob | Every 15 min, 7AM-10PM PT | Active accounts: today's account + campaign insights only | Dashboard prod |
| MetaBackfillJob | One-time per account | 37-month history | Dashboard prod |
| MetaRejectionSyncJob | Every 6 hours | Check ad review status changes | Psycle-Data |

After each sync: refresh `mv_ad_performance_daily`.

**Meta API rules:**
- Pagination: cursor-based (`paging.cursors.after`) -- must paginate to get all accounts
- Business Manager ID: `819717796169771`
- Caching: 15-min TTL insights, 1-hour creative/metadata (at API level, not DB)
- Exclude accounts in `META_EXCLUDE_ACCOUNTS` denylist (e.g., offboarded clients like Stella Center)

## API Endpoints

```
-- Ads drill-down (from Dashboard prod)
GET /api/v1/meta/accounts                     -- All ad accounts with optional insights
GET /api/v1/meta/campaigns/:accountId         -- Campaigns + insights
GET /api/v1/meta/adsets/:campaignId           -- Adsets + insights
GET /api/v1/meta/ads/:adsetId                 -- Ads + insights
GET /api/v1/meta/insights/:objectId           -- Raw insights for any object
GET /api/v1/meta/winning-ads                  -- Spend-ranked with health scoring
GET /api/v1/meta/ad-preview/:adId             -- Lazy-loaded preview iframe
GET /api/v1/meta/insights-compare             -- Current vs prior period
GET /api/v1/meta/active-psy-accounts          -- Accounts with active "PSY" campaigns

-- Geo (from Dashboard prod)
GET /api/v1/meta/clinic-locations             -- Clinic locations from ad targeting
GET /api/v1/meta/geo-targeting                -- Targeting cities/zips for heatmap
GET /api/v1/meta/geo-insights                 -- Region-level spend/leads/CPL
GET /api/v1/meta/clinic-page-locations        -- Real addresses from Facebook Pages API
GET /api/v1/meta/lead-zips                    -- Lead form ZIP submissions
GET /api/v1/meta/zip-boundaries               -- ZCTA boundaries from Census TIGERweb

-- Account mapping (from prototype mockup)
GET    /api/v1/ad-account-mappings            -- All mappings
POST   /api/v1/ad-account-mappings            -- Create mapping
PATCH  /api/v1/ad-account-mappings/:id        -- Update
DELETE /api/v1/ad-account-mappings/:id        -- Remove
```

## UI Components

**Ads drill-down** (migrate from Dashboard prod React components):
- Hierarchy: Accounts -> Campaigns -> Adsets -> Ads
- KPI summary cards per level (spend, leads, CPL, active count)
- Sortable/paginated table with status badges
- Date picker: L7D / L14D / L28D / custom
- Ad preview modal (lazy-loaded)
- Winning ads view with health scoring + min lead filter (1-10)

**Geo visualization** (migrate from Dashboard prod):
- Mapbox map with clinic locations
- Heatmap overlay for targeting cities/zips
- Lead ZIP code visualization with Census ZCTA boundaries
- Agency map with all clinic pins

**Accounts Matcher** (build from prototype mockup, in Settings):
- Org cards with expand/collapse
- Per-org: GHL Location dropdown, Meta Ad Account dropdown, Campaign multi-select pills, Result Type selector
- Multi-region orgs: indented subaccount rows with individual mappings
