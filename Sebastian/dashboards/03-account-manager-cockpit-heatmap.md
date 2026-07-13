# Dashboard 03 -- Account Manager Cockpit (Heat Map)

**Route:** `/heatmap`
**Group:** Delivery
**Users:** Account Managers
**Status:** Live (staging twin `/heatmap-staging`)
**Refresh:** `psycle-heatmap-refresh` (hourly)
**Source project:** Psycle-Data (`q_heatmap.sql`)

---

## 1. Purpose & Users

US ZIP-code choropleth of lead/eval distribution on a real Google Maps base, plus a clinic table and recent tasks. Click a clinic to cross-filter. Gives Account Managers a geographic read on where leads and evals are concentrated per clinic.

## 2. Data Sources (BigQuery)

- `q_heatmap.sql` ->
  - `ghl.contact_funnel`
  - `ghl.clinic_lookup`
  - `meta.ad_insights`
  - `geo_us_boundaries.zip_codes` (simplified polygons)

## 3. Features

- Choropleth fill by lead/eval volume, CPE, or lead->eval.
- Client-side clinic / insurance / source / date filters.
- State rollup.
- Google Maps key sourced from Secret Manager.
- Pod chips.

## 4. Status & Refresh

- **Status:** Live, with a `-staging` twin.
- **Refresh:** `psycle-heatmap-refresh` Cloud Run job, hourly.

## 5. Known Gaps

- Pod-filtered views can render zero metrics / no ZIPs due to a pod-label mismatch in the JS filter.
- Color scale needs recalibration to realistic benchmarks.
- Uses `meta.ad_insights` while other surfaces use `meta.ad_performance_daily` -- flagged for source-of-truth standardization.
- Clinic-table <-> map cross-filter is on the roadmap but not yet shipped.
