# Dashboard 08 -- Eval Pricing Calculator

**Route:** `/calculator`
**Group:** Pricing
**Users:** Finance
**Status:** Live
**Refresh:** n/a -- static, all computation client-side (no BigQuery)
**Source project:** Psycle-Data (hand-authored static tool)

---

## 1. Purpose & Users

Per-eval price by state, payer mix & exclusivity -- a static, hand-authored client-side tool (no BigQuery). Lets Finance quote a per-eval price given a clinic's state, expected payer mix, and whether the engagement is exclusive.

## 2. Data Sources (BigQuery)

- None -- all computation is client-side. No warehouse dependency.

## 3. Features

- State selector.
- Payer-mix inputs.
- Exclusivity toggle.
- Outputs a per-eval price.

## 4. Status & Refresh

- **Status:** Live.
- **Refresh:** None required -- the tool is static HTML/JS with no data pipeline.

## 5. Known Gaps

- Still uses a legacy hardcoded palette bridged by `_BRIDGE`; pending Sunset-token migration.
- Pricing coefficients are hand-authored -- no automated tie to warehouse reimbursement data.
