# Dashboard Suite -- Overview

**Doc type:** Retrospective / as-built PRD
**Status:** Living -- reflects shipped state
**Owner:** Sebastian -- BI/Reporting lane
**As of:** 2026-07-13
**Portal:** `psycle-dashboards` (Cloud Run, us-central1, project `ghl-data`)
**Source project:** Psycle-Data (`psycle-intelligence/dashboards`)

> One IAP-protected portal turning the GHL + Meta + eval-tracker warehouse into role-scoped decision surfaces for media buying, account management, care coordination, and clinic reporting.

This is the suite-level overview. Each dashboard has its own PRD in this folder (`01-` through `12-`), fully detailed with purpose, data sources, features, status, refresh, and known gaps.

---

## 1. Executive Summary

The suite is a single internal web portal that renders self-contained HTML dashboards from one private bucket, each fed by the BigQuery warehouse. It replaced a fragmented set of one-off views and spreadsheets with one door, one house style, and one access model.

| Metric | Value |
|--------|-------|
| Live production dashboards | 8 |
| Staging-only surfaces awaiting promotion | 3 |
| NL-query assistant (pilot) | 1 |
| Serving footprint | 1 serving service, 1 bucket, 4 refresh jobs |

Everything lives behind Google IAP, restricted to `psyclehealth.com`, and is further gated per-department. Content is de-identified aggregate data -- no patient-level PHI reaches any dashboard.

---

## 2. Problem & Goals

Media buyers, account managers, and leadership were each stitching together Meta Ads Manager, GHL, and per-clinic eval spreadsheets by hand. Numbers disagreed, nobody trusted a single source, and there was no role-appropriate view.

### Goals

- One trusted surface per role, all reading the same canonical warehouse.
- Spend -> lead -> eval -> treatment funnel visible end-to-end, per clinic and per pod.
- Grade performance consistently (shared RAG thresholds), not by gut.
- Fast iteration -- regenerate an HTML file and it's live in seconds.
- Department-scoped access with a self-serve share UI.

### Non-goals

- No patient-level PHI on any dashboard -- aggregate/de-identified only.
- No per-app Cloud Run services -- one serving portal, always.
- No production GHL/Meta writes from a dashboard (read-only surfaces).
- Not a replacement for Looker Studio's exploratory BI -- the two coexist.

---

## 3. Users & Access

A per-user, department-scoped ACL stored as a single `access.json` in the bucket (60s in-memory cache). Identity comes from the IAP header; default access is **none until assigned**; admins see everything. A granted live page implicitly grants its staging twin.

| Department | Default pages granted |
|------------|-----------------------|
| Admins | All pages -- seeded: `sebastian@`, `danny@` |
| Creative | Top Ads |
| Account Managers | Account Manager Cockpit, Client Monthly, Care Coordinator, Analyst |
| Finance | Eval Pricing Calculator, Acquisition Cockpit |
| Media Buying | Media Buyer Command Center (+ staging) |

A Sheets-style `/admin` grid (admin-only) manages the user x department matrix; `/staging` lists the staging twins a user may open. The `psycle-analyst` pilot is additionally gated by an env allow-list on top of the department layer.

---

## 4. Architecture

Serving and refresh are deliberately separated: the portal only serves static HTML from a bucket, and Cloud Run jobs exist solely for unattended refresh. Change a dashboard's content by regenerating its HTML and copying it to the bucket -- live in seconds, no redeploy.

| Component | Detail |
|-----------|--------|
| Portal | `psycle-dashboards` -- Flask on Cloud Run (us-central1, project `ghl-data`), behind IAP + domain lock. Injects the house stylesheet into every page. |
| Content store | `gs://psycle-dashboards-ghl-data` -- private bucket (public access prevented); holds every `*.html` + `access.json`. |
| Generators | `build_*.py` query BigQuery via the `bq` CLI and self-upload the HTML artifact. Run locally or in a job. |
| Shared layer | `common.py` -- canonical thresholds, pace-grading SQL, pod-filter chip UI; injected into generators. |
| Refresh jobs | Cloud Run jobs + schedulers (PT): dashboards-refresh (daily 06:00), topads-refresh (~4h), heatmap-refresh (hourly), mb job. Many jobs -> one bucket -> one portal. |
| House style | Sunset Afterglow -- dark-native token system, `data-theme` toggle, one stylesheet edited once to restyle all views. |

---

## 5. Dashboard Catalog

Eleven distinct surfaces plus the assistant pilot. Live pages carry a `-staging` twin (topads, agency, cc, heatmap, mb) for eyeball-before-promote.

| # | Dashboard | Route | Group | For | Status |
|---|-----------|-------|-------|-----|--------|
| 01 | Top Ads by Geo & Insurance | `/topads` | Acquisition | Creative | Live |
| 02 | Acquisition Cockpit | `/agency` | Acquisition | Finance | Live |
| 03 | Account Manager Cockpit (Heat Map) | `/heatmap` | Delivery | AMs | Live |
| 04 | Care Coordinator Report | `/cc` | Delivery | Ops/AMs | Live |
| 05 | Client Monthly Report | `/client` | Delivery | AMs | Live |
| 06 | Client Performance Report | `/client-report` | Delivery | Client-facing | Live (orphan generator) |
| 07 | Media Buyer Command Center | `/mb` | Media Buying | Media buyers | Live |
| 08 | Eval Pricing Calculator | `/calculator` | Pricing | Finance | Live |
| 09 | Insurance Intelligence | `/insurance-staging` | Delivery | AMs | Staging |
| 10 | Insurance Marketplace | `/insurance-marketplace-staging` | Acquisition | Strategy | Staging |
| 11 | Lead Routing suggestions | `/routing-staging` | Delivery | Ops | Staging |
| 12 | Psycle Analyst (NL query) | `/ask` | Assistant | Pilot users | Pilot |

Adjacent to the portal, a 3-page Looker Studio report reads the same warehouse -- see §7.

---

## 6. Design System & Metric Layer

### Sunset Afterglow (house style)

- Dark-native token system in one stylesheet, injected into every page by the portal.
- `data-theme` on `<html>` (light / dark / default dark) with a no-flash pre-paint script + 3-way toggle.
- Tokens only -- no shared component class names (they collided with view-local classes). Restyle all views by editing one file + one deploy.
- Type: Fraunces (serif display) + DM Sans (body). Legacy views (agency, client, calculator) still carry hardcoded palettes, remapped by `_BRIDGE` -- migration in progress.

### Canonical thresholds (`common.py`)

- **Eval pace RAG** -- green >= 0.90, amber >= 0.75, red below, plus a "ramping" state.
- **CPL** ($/paid lead) -- green <= 20, amber <= 35, red above.
- **CPE** ($/eval) -- green <= 150, amber <= 270, red above.
- Shared pod-filter chips + pod-label map so a metric is defined once and every dashboard inherits it.

---

## 7. Data Foundation

All dashboards read the same layered warehouse; ratios and grades come from the canonical semantic layer, never re-invented per view.

| Layer / concept | Detail |
|-----------------|--------|
| Layering | raw `*_raw` (append-only) -> modeled `ghl` / `meta` / `google` (dedup-to-latest) -> `unified`. |
| Canonical layer | `ghl.contact_funnel` (per-lead funnel, INT64 `is_*` flags) + `ghl.cc_weekly_report` + the metric dictionary. |
| Key rates | lead->eval approx 7.33%, eval->showed approx 54.3%, lead->eval lag approx 17.3 days. Show rate = SHOWED / RESOLVED. Eval/CPE base = `evaluation_submission_date`. |
| Eval truth | `ghl.evaluations` from 30 clinic tracker Sheets (*/15 sync) is the operational source for post-eval outcomes; GHL alone understates evals. |
| Looker Studio | Adjacent 3-page report on `ghl.kpi_monthly` / `kpi_drift` / `kpi_detail` views (monthly KPIs, 12x12 drift + TRUE-CPE, campaign detail). Separate BI surface from the IAP portal. |
| Quality suite | 6 assertions (freshness, completeness, uniqueness, drift, validity, reconciliation) + an hourly anomaly monitor routing to Slack/Asana/email. |

---

## 8. Security & Compliance

- **IAP + domain lock** on the single portal; private bucket with public access prevented. No `--allow-unauthenticated`, ever.
- **Department ACL** -- default none-until-assigned; identity from the IAP header.
- **PHI posture** -- dashboards show de-identified aggregates only; patient name/notes stay locked in `ghl.evaluations` and are never logged or rendered. Age uses Safe-Harbor bands (90+ aggregated).
- **No production writes** -- all GHL writes route through `prod_write_guard` (dry-run default, sandbox-only, fail-closed); dashboards are strictly read-only.
- **Audit** -- data-access audit logging enabled with 6-year retention; scheduler-failure alerting on.
- **Open risk** -- cleartext AWS key + approx 60 GHL tokens in the Integrations sheet (escalated, tracked in remediation).

---

## 9. Known Gaps & Open Items

**Reconcile:** `client_perf_report.html` is served at `/client-report` but has no generator in the repo -- an orphan artifact. Either recover/author its generator or document its external source.

- **Promote staging surfaces** -- Insurance Intelligence, Insurance Marketplace, and Lead Routing ship staging-only, awaiting eyeball -> live routes.
- **Palette migration** -- agency, client, and calculator still use legacy hardcoded palettes (bridged by `_BRIDGE`); `build_mb.py` still carries inline copies of `common.py` helpers pending migration.
- **Source-of-truth alignment** -- MB and Client reports read submission-date sources directly rather than the normalized eval layer; heatmap uses `ad_insights` while others use `ad_performance_daily` -- standardize.
- **/mb polish backlog** -- rejected-ad icon should distinguish "currently rejected" from "rejected-before-but-running"; eval-source section must sum to the client-row eval total; bundled-clinic map recenters to HQ instead of the clinic.
- **Heatmap pod views** -- pod-filtered views can render zero metrics / no ZIPs (pod-label mismatch in the JS filter); color scale needs recalibration to realistic benchmarks.
- **Auto-refresh** -- two promotion-protection crons were paused to protect a live promotion; reconcile job images to canonical and resume.

---

## 10. Roadmap

- **Promote & migrate** -- ship the three staging surfaces; finish the sunset-token migration and retire `_BRIDGE`.
- **Cross-filter depth** -- `/mb` map cross-filter by campaign/adset/ad node; heatmap clinic-table <-> map cross-filter.
- **Routing activation** -- Phase A (suggest) -> B (sandbox via `prod_write_guard`) -> C (gated prod webhook). Never unguarded prod writes.
- **Insurance capture** -- GHL "insurance asked on call" field to fix the blank insurance-category gap feeding the insurance dashboards.
- **Analyst GA** -- graduate the `/ask` pilot once reconciliation + access are proven.
- **Client Performance** -- benchmarked LTV/reimbursement wiring (pending approval) + a missing-fields ingestion/flagging plan.
