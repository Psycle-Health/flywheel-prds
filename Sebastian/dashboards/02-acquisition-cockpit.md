# Dashboard 02 -- Acquisition Cockpit

**Route:** `/agency`
**Group:** Acquisition
**Users:** Finance (also leadership + growth)
**Status:** Live (staging twin `/agency-staging`)
**Refresh:** `psycle-dashboards-refresh` (daily 06:00 PT)
**Source project:** Psycle-Data (`q_agency.sql`, `compact.py`)

---

## 1. Purpose & Users

Meta spend -> eval-efficiency cockpit rolled up at agency / pod / clinic level. The leadership-and-finance view of acquisition efficiency across the whole book.

## 2. Data Sources (BigQuery)

- `q_agency.sql` -> `meta.ad_performance_daily` joined to `ghl.clinics`
- Grain: clinic x pod x state x month, compacted via `compact.py`

## 3. Features

- Monthly KPI tiles: leads / evals / CPL / CPE.
- Month stepper.
- Time-series + funnel charts.
- Sortable per-clinic table with cost-health coloring.
- Multi-select pod chips.

## 4. Status & Refresh

- **Status:** Live, with a `-staging` twin for eyeball-before-promote.
- **Refresh:** `psycle-dashboards-refresh` Cloud Run job, daily at 06:00 PT.

## 5. Known Gaps

- Still uses a legacy hardcoded palette bridged by `_BRIDGE`; pending migration to the Sunset Afterglow token system.
