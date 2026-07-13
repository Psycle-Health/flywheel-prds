# Dashboard 07 -- Media Buyer Command Center

**Route:** `/mb`
**Group:** Media Buying
**Users:** Media buyers
**Status:** Live (staging twin `/mb-staging`)
**Refresh:** own `mb_job`; also bucket-editable for fast iteration
**Source project:** Psycle-Data (`build_mb.py` -- the largest generator in the suite)

---

## 1. Purpose & Users

The media-buyer cockpit: red clients by eval pace, per-clinic performance, campaign drill, and lead geography -- the largest generator in the suite. The daily working surface for media buyers to find under-pacing clinics and drill into why.

## 2. Data Sources (BigQuery)

- `meta.ad_performance_daily`
- `meta.ad_status` (active flag)
- `ghl.evaluations` (goals / pace)
- `ghl.asana_tasks`
- `ghl.dim_clinic`
- `ghl.clinic_geo`

## 3. Features

- One master full-roster clinic table (zero-filled, worst-first) with Red/Amber/Green sort chips + pod-aware RAG tiles.
- Each row expands to:
  - a ZIP choropleth,
  - a campaign -> adset -> ad funnel drill (active = green),
  - an eval-source split,
  - recent Asana tasks.
- Global date picker.
- Out-of-geo % (OOG).

## 4. Status & Refresh

- **Status:** Live, with a `-staging` twin.
- **Refresh:** Own `mb_job` Cloud Run job; also directly bucket-editable for fast iteration without a full rebuild.

## 5. Known Gaps

- Rejected-ad icon should distinguish "currently rejected" from "rejected-before-but-running."
- Eval-source section must sum to the client-row eval total.
- Bundled-clinic map recenters to HQ instead of the clinic.
- Still carries inline copies of `common.py` helpers pending migration to the shared layer.
- Reads submission-date sources directly rather than the normalized eval layer -- flagged for source-of-truth standardization.
- Roadmap: map cross-filter by campaign / adset / ad node.
