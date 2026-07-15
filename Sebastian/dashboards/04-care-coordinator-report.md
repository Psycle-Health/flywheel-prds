# Dashboard 04 -- Care Coordinator Report

**Route:** `/cc`
**Group:** Delivery
**Users:** Ops / Account Managers (people-sensitive -- performance by named CC)
**Status:** Live (staging twin `/cc-staging`)
**Refresh:** `psycle-dashboards-refresh` (daily 06:00 PT)
**Source project:** Psycle-Data (`q_cc.sql`)

---

## 1. Purpose & Users

Weekly & quarterly performance of every Care Coordinator, by name; scope by pod or new-hire filter. People-sensitive -- surfaces individual CC performance for Ops and AMs.

## 2. Data Sources (BigQuery)

- `q_cc.sql` ->
  - `ghl.cc_daily`
  - `ghl.contacts`
  - `ghl_raw.message_events`
  - `ghl.appointments`
  - `ghl.cc_roster`
  - `ghl.ghl_users`

## 3. Features

- Date presets including QTD.
- Connection rate.
- Show-rate maturity badge (resolved / unresolved).
- 5-way eval-status breakdown.
- New-hire filter derived from hire date.
- Per-CC drilldown.

## 4. Status & Refresh

- **Status:** Live, with a `-staging` twin.
- **Refresh:** `psycle-dashboards-refresh` Cloud Run job, daily at 06:00 PT.

## 5. Known Gaps

- People-sensitive by design -- attribution uses actor `userId` (the caller), never `assigned_to`, to keep CC credit correct.
- No known open defects specific to this surface beyond the suite-wide palette/source-of-truth alignment work.
