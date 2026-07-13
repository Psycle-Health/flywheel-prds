# Dashboard 06 -- Client Performance Report

**Route:** `/client-report`
**Group:** Delivery
**Users:** Client-facing (shared with clinics)
**Status:** Live -- **orphan generator (see gaps)**
**Refresh:** n/a -- artifact produced outside the tracked generators
**Source project:** Psycle-Data (serves `client_perf_report.html`)

---

## 1. Purpose & Users

Client-facing ROI report by clinic + period: investment -> treatments -> revenue, with a built-in data-quality suite and print/PDF. The report a clinic sees to understand the return on their spend.

## 2. Data Sources (BigQuery)

- Serves `client_perf_report.html`.
- **No generator for this file exists in the repo** -- the artifact is produced outside the tracked generators. Flagged for reconciliation (see §5 and the suite overview).

## 3. Features

- ROI narrative: investment -> treatments -> revenue.
- Per-clinic reimbursement / LTV.
- Built-in data-quality checks.
- Printable (print/PDF).

## 4. Status & Refresh

- **Status:** Live, but generator-orphaned.
- **Refresh:** No tracked refresh job -- the served HTML is authored outside the repo's generators.

## 5. Known Gaps

- **Reconcile:** `client_perf_report.html` is served at `/client-report` but has no generator in the repo. Either recover/author its generator or document its external source.
- Benchmarked LTV / reimbursement wiring is pending approval; a missing-fields ingestion/flagging plan is on the roadmap.
