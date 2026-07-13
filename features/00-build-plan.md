# Flywheel Build Plan -- Feature-Based PRDs

**For:** Allen Lee (CTO)
**Deadline:** PRDs due July 16, 2026. Target: consolidated platform by end of Q3 2026.
**Method:** Each feature is a self-contained buildable unit. Build in order -- each depends on prior features.

---

## Build Order

| # | Feature | Dependencies | Est. Complexity | Source Projects |
|---|---------|-------------|----------------|----------------|
| 01 | [Auth & Users](./01-auth-users.md) | None (foundation) | Medium | Dashboard prod, Internal app |
| 02 | [Clinics, Orgs & Pods](./02-clinics-orgs-pods.md) | 01 | Medium | Dashboard prod, Internal app, Psycle-Data (clinic_lookup) |
| 03 | [GHL Sync Pipeline](./03-ghl-sync.md) | 01, 02 | High | Dashboard prod (webhook+reconcile), Psycle-Data (9 BQ sync jobs) |
| 04 | [Meta Sync & Ads Dashboard](./04-meta-ads.md) | 01, 02 | High | Dashboard prod (drill-down+winning ads), Psycle-Data (meta-insights-sync) |
| 05 | [Contact Funnel & Analytics Views](./05-contact-funnel.md) | 03, 04 | High | Psycle-Data (contact_funnel, evaluations_norm, cc_daily, METRICS_DICTIONARY) |
| 06 | [Patient Funnel Pages](./06-patient-funnel.md) | 03, 05 | Medium | Dashboard prod (7 data pages), Psycle-Data (evaluations_norm) |
| 07 | [CC Dashboard](./07-cc-dashboard.md) | 03, 05 | Medium | Psycle-Data (/cc, cc_daily), Jul 6 "App Overview" meeting |
| 08 | [Media Buyer Command Center](./08-media-buyer.md) | 04, 05 | High | Psycle-Data (/mb, /agency, /topads, /heatmap, calculator) |
| 09 | [Client Reports & Leaderboard](./09-client-reports.md) | 05, 06 | Medium | Psycle-Data (/client-report), Dashboard prod (leaderboard), Internal (retainers) |
| 10 | [Clinic Settings & Accounts](./10-clinic-settings.md) | 02 | Medium | Dashboard prod (settings), prototype (accounts matcher mockup) |
| 11 | [Financial Ops](./11-financial-ops.md) | 02, 09 | Medium | Internal app (revenue/expenses/contractors/efficiency) |
| 12 | [Onboarding & Intake](./12-onboarding.md) | 02, 10 | Medium | Prototype (onboarding.html), Jun 26 "Clinics Tables" meeting |
| 13 | [Chat & AI](./13-chat-ai.md) | 01, 02 | Low | Dashboard prod (Hope AI + Slack) |
| 14 | [EMR & Scheduling](./14-emr-scheduling.md) | 02, 03 | High | Flywheel (P1 scheduler, P2 CH integration) -- already in progress |

---

## Psycle Internal User Types

| Type | Count | Access Scope |
|------|-------|-------------|
| `admin` | ~7 | Full access to everything |
| `growth` | 3 | Media buying, Meta ads, spend data, pod-scoped clinics |
| `am` | 4 | Client reports, leaderboard, patient funnel, clinic settings, pod-scoped |
| `cc` | ~15 | CC dashboard, calls, consults, leads, eval booking, pod-scoped |
| `content` | ~8 | Future scope, read-only dashboards via permission overrides |

## External (Clinic) User Types

| Type | Role Level | Access |
|------|-----------|--------|
| `provider` | clinic_admin or clinic_team | Own clinic, may require NPI + provider_type |
| `billing` | clinic_admin or clinic_team | Own clinic |
| `manager` | clinic_admin or clinic_team | Own clinic |
| `owner` | clinic_admin | Own clinic |

## 4 Pods

| Pod | AM | CCs |
|-----|-----|-----|
| Glutamates | Jake Inzalaca (AM Lead) | Zohreh, Anne Marie, Mia |
| Heavyweights | Kyla Davidson | Jamie, Aliah |
| Catalyst | Cody Shimokochi | Christine, David, Miguel, John |
| Pea Pod | Kerrie Clark | Francis, Jessica, Daisy |

Cross-pod Ops: Rachel Rubenstein (CC Manager), Leila Rishwain (Ops)

## Source-of-Truth Architecture

```
GHL owns (syncs TO Flywheel):        Flywheel owns (SOT, optional writeback TO GHL):
─────────────────────────────         ──────────────────────────────────────────────
Lead creation (contacts)              Qualification status
Insurance submission                  PA status, submission, approval, sessions
In-network determination              Treatment status, modality, sessions
Consult scheduling & show             Custom clinic fields (never in GHL)
Eval status, date, time, scheduled_by Anything imported from Google Sheets
Call/conversation data                 All Connective Health / EMR data
```

## Non-Negotiable Rules

1. Leads from `actions` where `action_type === "lead"`. Never `onsite_conversion.lead_grouped`.
2. CPL from Meta's `cost_per_action_type`. Never `spend / leads`.
3. All daily bucketing in `America/Los_Angeles` (PT).
4. Paid-only for media metrics (`ad_id IS NOT NULL`) except lead-source panel + ZIP map.
5. Eval counts include ALL evaluations with an evaluation submission date (including CX BY PATIENT / CX BY CLINIC). Cancellations also kept in show-rate denominators.
6. CC attribution = actor (caller), never `assigned_to`.
7. GHL location ID = clinic PK. Never generate separate IDs.
8. No PHI in logs. Encrypt at rest with AES-256-GCM.
9. All secrets via AWS Secrets Manager. Never plaintext env vars.
10. `clinic_id` separated from `org_id` (multi-location support).
11. Insurances are provider-scoped, not clinic-scoped.
