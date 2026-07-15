# Dashboard 01 -- Top Ads by Geo & Insurance

**Route:** `/topads`
**Group:** Acquisition
**Users:** Creative team
**Status:** Live (staging twin `/topads-staging`)
**Refresh:** `psycle-topads-refresh` job (~4h); Meta previews/thumbs cached with TTL
**Source project:** Psycle-Data (`build_topads.py`)

---

## 1. Purpose & Users

Best-performing ad-creative leaderboard by state x insurance category, for the Creative team. Answers "which creatives are winning, where, and for which insurance segment?" so the Creative team can double down on what works and retire what doesn't.

## 2. Data Sources (BigQuery)

- `meta.ad_performance_daily`
- `ghl.contact_funnel`
- Intelligence-layer joins: `meta.ad_dim`, `meta.creative_launch_scores_v2`, `meta.ad_intelligence`, `meta.ad_allocation_reco`

## 3. Features

- Creative leaderboard ranked by performance.
- Date windows: 30 / 90 / 365 / all.
- Geo x insurance cells (state x insurance category).
- Live Meta ad-preview modal (video iframe, with an Ads Manager fallback).
- **Staging adds** SCALE / HOLD / CUT allocation intelligence + launch/rejection scores from the intelligence layer.

## 4. Status & Refresh

- **Status:** Live. A `-staging` twin carries the allocation-intelligence enhancements ahead of promotion.
- **Refresh:** `psycle-topads-refresh` Cloud Run job on a ~4h cadence; Meta previews and thumbnails are cached with a TTL to stay within Meta rate limits.

## 5. Known Gaps

- Allocation-intelligence layer (SCALE/HOLD/CUT + launch/rejection scores) is staging-only, awaiting promotion to the live route.
- Depends on UTM passthrough for ad-level attribution; blank insurance-category cells trace to the missing "insurance asked on call" capture (see suite roadmap).
