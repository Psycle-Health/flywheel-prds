# Dashboard 09 -- Insurance Intelligence

**Route:** `/insurance-staging`
**Group:** Delivery
**Users:** Account Managers
**Status:** Staging -- awaiting promotion to a live route
**Refresh:** via the staging build path
**Source project:** Psycle-Data (VOB-sourced)

---

## 1. Purpose & Users

Per-clinic accepted-carrier profile + network-wide carrier reach, from VOB-sourced data. Aggregate carrier counts, not PHI. Helps AMs understand which carriers a clinic accepts and how that compares to the network.

## 2. Data Sources (BigQuery)

- `ghl.clinic_insurance` (`n_patients`, in/out-of-network, pending, `pct_in_network`)

## 3. Features

- Carrier tiles.
- Clinic filter.
- Sortable carrier table with in-network %.
- Uses Sunset Afterglow tokens.

## 4. Status & Refresh

- **Status:** Staging. Awaiting promotion to a live route.
- **Refresh:** Runs through the staging build path.

## 5. Known Gaps

- Staging-only -- awaiting eyeball -> live promotion.
- Blank insurance-category coverage depends on adding a GHL "insurance asked on call" capture field (suite roadmap).
- Aggregate carrier counts only -- deliberately no PHI.
