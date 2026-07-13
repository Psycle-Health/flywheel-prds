# Dashboard 05 -- Client Monthly Report

**Route:** `/client`
**Group:** Delivery
**Users:** Account Managers
**Status:** Live
**Refresh:** `psycle-dashboards-refresh` (daily 06:00 PT)
**Source project:** Psycle-Data (`q_client.sql`)

---

## 1. Purpose & Users

Per-clinic monthly funnel: eval -> prior-auth -> treatment. The AM-facing monthly rollup of where each clinic's patients are in the post-eval funnel.

## 2. Data Sources (BigQuery)

- `q_client.sql` -> `ghl.contacts` joined to `ghl.contact_funnel`
- Cohort: eval-submission-month. Prior-auth (PA) and treatment (TX) definitions are provisional.

## 3. Features

- Funnel hero with untracked-stage hatching (visually flags stages not yet tracked).
- Clinic selector.
- Month stepper.
- KPI tiles.

## 4. Status & Refresh

- **Status:** Live.
- **Refresh:** `psycle-dashboards-refresh` Cloud Run job, daily at 06:00 PT.

## 5. Known Gaps

- PA and TX stage definitions are provisional -- untracked stages are hatched in the funnel until wired.
- Reads submission-date sources directly rather than the normalized eval layer -- flagged for source-of-truth standardization.
- Still uses a legacy hardcoded palette bridged by `_BRIDGE`; pending Sunset-token migration.
