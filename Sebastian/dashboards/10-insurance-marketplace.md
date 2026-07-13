# Dashboard 10 -- Insurance Marketplace

**Route:** `/insurance-marketplace-staging`
**Group:** Acquisition
**Users:** Strategy
**Status:** Staging -- awaiting promotion
**Refresh:** via the staging build path
**Source project:** Psycle-Data

---

## 1. Purpose & Users

Supply-coverage view -- which carriers are accepted where, plus coverage gaps (carrier x state with demand but few in-network clinics). A strategy surface for spotting where the network is thin against demand.

## 2. Data Sources (BigQuery)

- `ghl.clinic_profile`
- `ghl.clinic_insurance`

## 3. Features

- Carrier x state matrix.
- Per-clinic supply profile.
- Gaps panel (carrier x state with demand but few in-network clinics).

## 4. Status & Refresh

- **Status:** Staging. Awaiting promotion.
- **Refresh:** Runs through the staging build path.

## 5. Known Gaps

- Staging-only -- awaiting promotion to a live route.
- Coverage-gap detection depends on demand signal quality, which is tied to the same insurance-capture gap as Dashboard 09.
