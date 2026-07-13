# Dashboard 11 -- Lead Routing Suggestions

**Route:** `/routing-staging`
**Group:** Delivery
**Users:** Ops
**Status:** Staging -- Phase A (suggest only)
**Refresh:** via the staging build path
**Source project:** Psycle-Data (matcher, Phase A)

---

## 1. Purpose & Users

Phase-A delivery surface for the matcher: one row per recent lead -- where it landed vs. the better-fit clinic + suggested CC. **Read-only suggestions, never auto-assigned.** Gives Ops a review surface for routing quality before any automated routing is switched on.

## 2. Data Sources (BigQuery)

- `ghl.lead_routing_suggestions` (`zip`, `carrier`, current/suggested clinic, `p_show`, `match_score`, `distance`, `network_fit`, `suggested_cc`, `reasoning`)

## 3. Features

- Routing table sorted by mismatch + show probability, with match reasoning.
- Foundation for the future routing webhook (suggest -> sandbox -> gated prod).

## 4. Status & Refresh

- **Status:** Staging, Phase A (suggestions only).
- **Refresh:** Runs through the staging build path.

## 5. Known Gaps

- Read-only by design -- never auto-assigns leads.
- Routing activation roadmap: Phase A (suggest) -> Phase B (sandbox via `prod_write_guard`) -> Phase C (gated prod webhook). Never unguarded prod writes.
