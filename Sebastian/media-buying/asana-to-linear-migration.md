# PRD -- Asana -> Linear: Media-Buying Ticketing & Reports

**Department:** Media Buying -- tooling migration
**Status:** Draft for review
**Owner:** Sebastian
**Aligns with:** Linear = write-spine initiative
**As of:** 2026-07-13
**Source project:** `ghl-data-repo` (current dependency inventory)

> Migrate every media-buying task -- historical and ongoing, for every team member -- from Asana to Linear, preserve today's automation and dashboards, and make Linear the write-spine that feeds the weekly reports. Built to run alongside Asana with zero disruption until an explicit cutover.

---

## 1. Objective & Principles

Consolidate ticketing on Linear so it's the single system of record for media-buying work -- searchable per member, historically complete, and driving both the dashboards and the weekly reports. This is the natural home of the existing "Linear = write/feature spine, warehouse = read layer" direction.

### Goals

- All media-buying tasks (both Asana projects, all members) live in Linear -- history included.
- Ongoing auto-ticketing (Slack, Fellow, Drive, anomalies) writes to Linear.
- Dashboards + weekly reports read Linear-sourced data with no loss of columns.
- Clinic / pod / bottleneck become structured labels -- not fragile title-parsing.
- One config for all IDs -- no more gids scattered across five files.

### Principles

- **Parallel-run, then cut.** Build Linear beside Asana; cut over per surface only when verified.
- **Preserve the read contract.** Consumers keep reading the same column names via a compatibility view.
- **No production writes without approval** -- same guardrail as today.
- **Reversible.** Asana stays read-only during cutover; nothing deleted until sign-off.

---

## 2. What Exists Today (the footprint to preserve)

Two Asana projects -- **Media Buying** (`1213118591047727`) and **Operations & Data** (`1214582035897832`) -- reached by one PAT (`asana-pat`). Five systems touch them:

| System | Direction | What it does |
|--------|-----------|--------------|
| `asana-sync` | pull -> BQ | Hourly Cloud Run job -> `ghl_raw.asana_tasks` -> view `ghl.asana_tasks` (the read contract). |
| `slack-ticketing` | write | @media_buyers Slack msgs -> Media-Buying tasks + all custom fields + `:memo:` reaction. |
| `fellow-asana-sync` | write | Fellow PR-call action items -> Media-Buying tasks (in-session; pod-gated to Sebastian). |
| `anomaly-monitor` | write | Data anomalies -> Ops & Data tasks; dedup via `ghl.anomaly_state.asana_task_gid`. |
| in-session cron | write (MCP) | `ticketing_cron.txt` creates tasks via Asana MCP (Slack B / Drive D / Fellow E) with the same gids. |

**Consumers of `ghl.asana_tasks`:** the **/mb** and **heatmap** dashboards, the **weekly-meta-report** overview authoring, and dedup in slack-ticketing + anomaly-monitor.

**Load-bearing pieces:** the title convention `[Clinic] | [Type] [Bottleneck]` (via `asana_labeler.py`), the `asana_clinic_crosswalk`, and the columns `clinic_label` / `is_clinic_task` / `status` / `name`/`action` / `notes` / `comments` / `permalink_url` / `due_on` / `modified_at`.

---

## 3. Target Model & the Asana -> Linear Mapping

Linear's native model is teams, workflow states, issues, labels, priority, assignee, projects, cycles. It has no arbitrary per-issue custom fields, so Asana's custom fields become **label groups** (set deterministically at creation). This is an upgrade: clinic/pod/bottleneck stop being parsed from the title and become queryable structured data -- killing the misassignment bug.

| Asana | Linear | Notes |
|-------|--------|-------|
| Project: Media Buying | Team "Media Buying" | All member issues; workflow states below. |
| Project: Operations & Data | Team "Ops & Data" | Anomaly + Drive-mention issues. |
| Section: Tickets | State **Triage** / label | Landing state for auto-created issues. |
| Task | **Issue** | Title keeps `[Clinic] \| [Type] [Bottleneck]`. |
| Custom field **Status** (To Do) | **Workflow State** | Todo -> In Progress -> **Ready for Review** -> Done. Ready-for-Review counts as done in reports. |
| Custom field **Pod** | Label group `Pod/*` | Glutamates, Catalyst, Pea Pod, Heavyweights. |
| Custom field **Clinic** | Label group `Clinic/*` | Deterministic key -> replaces the fuzzy crosswalk. |
| Bottleneck (in title) | Label group `Bottleneck/*` | The 7-value taxonomy becomes structured. |
| Type (Investigate/Optimize) | Label group `Type/*` | -- |
| Custom field **Priority** | Native Priority | Urgent / Medium -> Linear priority levels. |
| Custom field **Media Buyer(s)** | Assignee | + subscribers/label if more than one. |
| Custom field **Dept** | Team (implicit) | Media-Buying team makes Dept redundant. |
| Comments / stories | Issue comments | Ported in backfill. |
| notes / due_on / assignee / completed / permalink | description / due date / assignee / state=Done / issue URL | Direct 1:1. |

**Single source of IDs:** Every team / state / label id lives in one versioned `linear_ids.json` that all writers import -- directly retiring the six gid sets duplicated across `slack-ticketing/app.py`, `fellow-asana-sync/config.json`, `ticketing_cron.txt`, and the handoff doc.

---

## 4. Migration Phases

Phases 0-2 build Linear beside a fully-live Asana (no disruption). Phases 3-4 are the cutover, run in dual-write/parallel. Phases 5-6 finish and decommission.

### Phase 0 -- Linear foundation (no disruption)

- **Do:** Create the two teams, workflow states (incl. Ready-for-Review), and label groups (Pod / Clinic / Bottleneck / Type / Source). Provision a Linear API key + OAuth app -> secret `linear-api-key`. Author `linear_ids.json`.
- **Deliverable:** An empty but fully-shaped Linear workspace + the id config.
- **Risk:** Clinic label set must cover every active + historical clinic (see failure #7).

### Phase 1 -- Historical backfill (all members, all history) (no disruption)

- **Do:** Export both Asana projects (tasks, custom fields, comments, assignees, dates, completion). Import to Linear via the official Asana importer where it preserves metadata, else a custom GraphQL importer (`services/asana-to-linear-backfill/`). Map fields per §3; derive Clinic/Pod/Bottleneck/Type labels; match assignees by email.
- **Deliverable:** Every Asana task present in Linear with labels, comments, assignee, completion.
- **Risk:** `createdAt` backdating, comment authorship/timestamps, attachments, departed users (failure #6). Reconcile counts per project/clinic before proceeding.

### Phase 2 -- Warehouse mirror (read contract) (no disruption)

- **Do:** Build `linear-sync` (GraphQL API + webhooks) -> `ghl_raw.linear_issues` -> view `ghl.linear_issues` whose columns match `ghl.asana_tasks` 1:1 (`clinic_label` from the Clinic label, `is_clinic_task` = has Clinic label, `status` from state, `permalink`=url...). Publish compatibility view `ghl.tasks` = UNION(historical asana_tasks, live linear_issues), deduped.
- **Deliverable:** `ghl.tasks` -- a drop-in replacement for `ghl.asana_tasks`.
- **Risk:** The `ghl.asana_tasks` view + `asana_clinic_crosswalk` are defined only in BQ, not the repo -- capture their DDL into the repo first (failure #3).

### Phase 3 -- Ongoing write-path cutover (dual-write)

- **Do:** Rewrite the three writers (slack-ticketing, fellow-sync, anomaly-monitor) to create Linear issues via GraphQL using `linear_ids.json` -- labels + assignee + priority + state. Port `asana_labeler.py` -> `task_labeler.py` (same title convention + label derivation; keep the CI sync check). Switch the in-session `ticketing_cron.txt` from Asana MCP to Linear MCP. Keep the `:memo:` Slack ack + BQ-first dedup (now on `ghl.tasks`).
- **Deliverable:** New tickets land in Linear; optional brief dual-write window to de-risk.
- **Risk:** Headless jobs can't use interactive Linear MCP -- they use the API key; only the in-session cron uses MCP (failure #5). All six id sets must be swapped (failure #1).

### Phase 4 -- Consumer cutover (dashboards + reports) (parallel-run)

- **Do:** Repoint `build_mb.py` + `build_heatmap.py` `SQL_TASKS` and the weekly-report overview query from `ghl.asana_tasks` -> `ghl.tasks` (column names preserved -> one-line diff). Map the Status badge -> Linear state; permalink -> Linear URL. Update the `/generate-weekly-reports` skill + the "Ready-for-Review = done" rule. Repoint anomaly dedup.
- **Deliverable:** Dashboards + reports run off Linear; compare side-by-side against Asana output before switching off the old view.
- **Risk:** Any consumer reading a column not reproduced in the compat view (failure #10).

### Phase 5 -- Reports from Linear (enhancement)

- **Do:** Reports already read Linear via `ghl.tasks` after Phase 4. Optionally add a "Weekly Reports" Linear project that tracks send-status per pod/week, and wire Linear's Slack integration for notifications. Fix the `chat:write` bot gap so sending can run unattended (see §6).
- **Deliverable:** Weekly reports sourced + tracked in Linear; send path hardened.
- **Risk:** Linear can't render the per-account Slack reports itself -- the generator stays ours (§6).

### Phase 6 -- Decommission Asana (after sign-off)

- **Do:** Stop `asana-sync` (freeze `ghl.asana_tasks` as history), archive the Asana projects read-only, revoke `asana-pat`, remove Asana MCP from the cron, retire the fuzzy crosswalk.
- **Deliverable:** Linear is the sole system of record; Asana archived, not deleted.
- **Risk:** Keep historical rows in `ghl.tasks` for reporting continuity.

---

## 5. Failure-Point Register

Grounded in the current dependency inventory. Severity: **high** / **med**.

| # | Failure point | Sev | Mitigation |
|---|---------------|-----|------------|
| 1 | Six project/field/option **gid sets hardcoded in >=5 files** (slack-ticketing, fellow config, cron, handoff). | high | Single `linear_ids.json` imported everywhere; delete the duplicates. |
| 2 | **Title convention + `asana_labeler`** is load-bearing for clinic resolution + every dashboard. | high | Keep the title convention; add label derivation; port to `task_labeler.py`; keep the CI sync check. |
| 3 | `ghl.asana_tasks` view + `asana_clinic_crosswalk` + `anomaly_state` are **defined only in BQ, not the repo**. | high | Capture current DDL into the repo before touching; rebuild as `linear_issues` + compat view. |
| 4 | Shared **single PAT** (`asana-pat`) across three services. | med | Linear API key/OAuth app in `linear-api-key`; mind GraphQL rate limits on backfill. |
| 5 | **Two write paths** (REST services + in-session MCP cron); Linear MCP needs interactive auth. | high | Migrate both; headless jobs use the API key, only the in-session cron uses Linear MCP. |
| 6 | **Historical fidelity**: `createdAt` backdating, comment authorship/timestamps, attachments, departed users. | med | Prefer the official importer; store original dates in a field/description; reconcile counts. |
| 7 | **Clinic-label coverage** for new subaccounts (same class as the `custom_field_map` gap). | med | Seed Clinic labels from `dim_clinic`; add a "no clinic label" alert; validate on create. |
| 8 | **"Ready for Review = done"** semantics must map to a Linear state honored by reports. | med | Dedicated workflow state; encode the done-set in the sync + report logic. |
| 9 | **Dedup drift** during dual-run (Slack `:memo:` ack vs BQ dedup). | med | Keep the reaction; store Slack permalink on the issue; dedup on `ghl.tasks`. |
| 10 | Consumers read specific columns (`clinic_label` / `is_clinic_task` / `status` / `name`/`action` / `notes` / `comments` / `permalink` / `due_on` / `modified_at`). | med | Reproduce every column in the compat view; diff dashboards before cutover. |
| 11 | Linear has **no arbitrary custom fields** + single assignee (vs Media Buyer(s) multi). | med | Label groups for structured fields; assignee + subscribers for multi-buyer. |
| 12 | Anomaly `[ALERT]` tasks are intentionally non-clinic. | med | Preserve `is_clinic_task=false` via absence of a Clinic label. |

---

## 6. Reports from Linear -- the honest scope

"Send reports from Linear" has two readable meanings; only one is Linear's job.

- **Source the report content from Linear** -- delivered by Phase 4. The weekly overviews already read task data via `ghl.tasks`, which becomes Linear-backed. No extra work beyond the cutover.
- **Have Linear broadcast the per-account Slack reports** -- not what Linear does. Linear is issue tracking; it won't render the WoW tables / top-ads / narratives per clinic channel. That generator stays ours (the weekly-meta-report service + the Slack send).

What Linear *can* add: a "Weekly Reports" project with one issue per pod per week to **track send status**, plus Linear's native Slack integration for lightweight notifications. Separately, this migration is the moment to fix the **`chat:write` bot gap** (today reports post only via Claude's Slack MCP because `slack-bot-token` is the read-only ticketing bot) so unattended sending finally works.

---

## 7. What Stays Exactly the Same

- The BigQuery warehouse, all metrics, and the semantic layer -- untouched.
- The weekly-report generator (build -> assemble -> render), the pod-per-thread format, and the Slack send mechanism.
- The `[Clinic] | [Type] [Bottleneck]` title convention + the pod/bottleneck taxonomy.
- Dedup-via-`:memo:`-reaction on the Slack source; the Fellow pod-enablement gate.
- The dashboards' look, routes, and columns (they just read `ghl.tasks`).

---

## 8. Open Decisions for Sebastian

- **Teams vs one team + projects** -- two Linear teams (Media Buying, Ops & Data) or one team with projects? Two teams mirrors Asana cleanly.
- **Backfill vehicle** -- Linear's official Asana importer (fast, best metadata fidelity) vs a custom GraphQL importer (full control of label mapping). Likely importer first, then a label-enrichment pass.
- **Hard cutover vs dual-write window** per writer -- dual-write is safer but doubles the ticket noise briefly.
- **Scope of "all members"** -- confirm which Asana assignees map to which Linear users, incl. anyone who has left.
- **`chat:write` bot** -- approve creating a proper Slack app for unattended report sending as part of this work?
