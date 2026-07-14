# Flywheel PRDs — ARCHIVED / MOVED

> **⛔ This repository is ARCHIVED (2026-07-14). The canonical PRD home has moved into the Flywheel repo.**
>
> - **Product PRDs** → `Psycle-Health/flywheel` : `docs/prd/features/` (feature-sliced)
> - **Engineering specs** → `docs/spec/{core,ops}/` (per-feature)
> - **Governance + the PRD → spec → ticket → implementation chain** → `docs/prd/README.md`
>
> The 14 feature PRDs here were migrated, are CI-enforced for traceability, and this repo is read-only.
> **Do not edit here.** The content below is the prior staging version, retained for history only.

---

# Flywheel PRDs

**Deadline:** July 16, 2026
**For:** Allen Lee (CTO / back-end lead)

## Feature-Based PRDs (for engineering -- BUILD FROM THESE)

See **[`features/00-build-plan.md`](./features/00-build-plan.md)** for the master index and build order.

14 self-contained, buildable features in dependency order. No duplication. Each has schema, API endpoints, UI components, and workflows.

## Department PRDs (for stakeholders -- REFERENCE ONLY)

The files in this directory (`01-media-buying.md` through `06-platform-infrastructure.md`) are organized by department/user type. They're useful for Jake, Rachel, Sebastian to find "their stuff" but contain cross-feature duplication. Allen should build from the feature-based PRDs.

---

## Original Department Structure (reference)

## PRD Index

| # | PRD | Department Lead | Users |
|---|-----|----------------|-------|
| 1 | [Media Buying](./01-media-buying.md) | Sebastian Roman | Growth: Gene Alder, Sebastian Ramon, Rodrigo Briao |
| 2 | [Care Coordination](./02-care-coordination.md) | Aliyah Taylor / Jessica Alcaraz | CC Manager (Rachel), CCs across all 4 pods |
| 3 | [Account Management](./03-account-management.md) | Jake Inzalaca (AM Lead) | 3 AMs: Kyla (Heavyweights), Cody (Catalyst), Kerrie (Pea Pod) |
| 4 | [Leadership & Finance](./04-leadership-finance.md) | Danny Costa | Founders (Danny, Elijah), Ops (Rachel, Leila) |
| 5 | [Clinic-Facing](./05-clinic-facing.md) | Catherine Jee | Clinic staff, providers, billing, owners (~51 clinics) |
| 6 | [Platform & Infrastructure](./06-platform-infrastructure.md) | Allen Lee | Dev: Allen, Maryam, Catherine, Nick Arora |

## Psycle User Types (Internal)

5 internal user types. These control sidebar visibility, data scoping, and feature access.

| User Type | Access Scope | Who |
|-----------|-------------|-----|
| **Admin** | Full access to everything -- all departments, all clinics, user management, settings | Danny, Elijah, Catherine, Allen, Maryam, Nick, Eleeza |
| **Growth** | Media buying dashboards, Meta ads, creative intelligence, spend data, assigned pod clinics | Gene, Sebastian R., Rodrigo |
| **AM** | Client reports, leaderboard, patient funnel pages, clinic settings -- scoped to assigned pod's clinics | Jake (lead), Kyla, Cody, Kerrie |
| **CC** | CC dashboard, calls, consults, lead follow-up, eval booking -- scoped to assigned pod's clinics | Rachel (manager), Aliyah (lead), Jessica (lead), Zohreh, Francis, Christine, Jamie, Anne Marie, Mia, David, Miguel, John, Daisy, + others |
| **Content** | Content creation tools (future scope), read-only on dashboards | Gabrys, Connor, Camila, Valeria, Sheena, Zara, Juan, Rosalind |

## Pod Structure

4 pods. Each pod has 1 Account Manager and multiple Care Coordinators.

| Pod | AM | CCs |
|-----|-----|-----|
| **Glutamates** | Jake Inzalaca (also AM Lead) | Zohreh Shayan, Anne Marie Spralja, Mia Guadarrama |
| **Heavyweights** (KEDA) | Kyla Davidson | Jamie Lei, Aliah Taylor |
| **Catalyst** | Cody Shimokochi | Christine Nguyen, David Kranker, Miguel Garcia, John Rieger |
| **Pea Pod** | Kerrie Clark | Francis Golez, Jessica Alcaraz, Daisy Garcia |

**Ops leadership (cross-pod):** Rachel Rubenstein (CC Manager), Leila Rishwain (Ops)

## Source Projects Consolidated

All features are drawn from 5 existing projects being unified into Flywheel:
- **Dashboard production** (Psycle-Health/dashboard) -- Maryam's React app, live
- **Psycle-Data** (Psycle-Health/Psycle-Data) -- Sebastian's BQ intelligence layer
- **Dashboard prototype** (dannyxcosta/Psycle-Dashboard) -- Costa's mockups + onboarding
- **Internal app** (Costa's private repo) -- Financial ops
- **Flywheel** (Psycle-Health/flywheel) -- Allen's Rails platform (the target)

## Key Architectural Decisions

- **Flywheel is the central source of truth.** GHL, CH, and third-party systems are satellites.
- **Postgres (RDS) for everything.** No BigQuery -- materialized views for analytics.
- **GHL is SOT for leads through consults.** Flywheel is SOT for evals onward.
- **Clinics can add custom fields** to their post-eval data without touching GHL.
- **Client ID must be separated from Location ID** in the schema (multi-location support).
- **Insurances are provider-scoped**, not clinic-scoped.
- **All timestamps in America/Los_Angeles (PT)** for daily bucketing.
