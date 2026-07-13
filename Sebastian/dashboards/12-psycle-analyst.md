# Dashboard 12 -- Psycle Analyst (NL Query)

**Route:** `/ask`
**Group:** Assistant
**Users:** Pilot users (gated by an env allow-list on top of the department ACL)
**Status:** Pilot
**Refresh:** n/a -- live query against the warehouse
**Source project:** separate `psycle-analyst` Cloud Run service (behind the same IAP)

---

## 1. Purpose & Users

Ask the warehouse in plain English and draft client replies -- read-only, reconciles to the dashboards. A pilot assistant for a small allow-listed set of users.

## 2. Data Sources (BigQuery)

- Proxied to a separate `psycle-analyst` Cloud Run service (behind the same IAP) that queries the warehouse. Reconciles to the same canonical layer the dashboards read.

## 3. Features

- Natural-language query over the warehouse.
- Client-reply drafting.
- Gated by an env allow-list on top of the department ACL.

## 4. Status & Refresh

- **Status:** Pilot.
- **Refresh:** No batch refresh -- answers are generated live against the warehouse at query time.

## 5. Known Gaps

- Pilot-only, doubly gated (department ACL + env allow-list).
- GA is blocked on proving reconciliation + access controls (suite roadmap: "graduate the `/ask` pilot once reconciliation + access are proven").
