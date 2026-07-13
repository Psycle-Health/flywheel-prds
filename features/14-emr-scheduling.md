# Feature 14 -- EMR & Scheduling

**Dependencies:** 02-clinics-orgs-pods, 03-ghl-sync
**Source:** Flywheel (P1 scheduler, P2 CH->EMR integration) -- already in progress under Allen
**Users:** Admin, CC (booking), Clinic Admin (view appointments)

This feature is ALREADY BEING BUILT in the Flywheel repo. This PRD documents it for completeness and to show how it connects to the other features. Refer to `services/core/PRD.md` and `apps/ops/PRD.md` in the Flywheel repo for the authoritative specs.

---

## P1 -- Scheduling (Coordinator Calendar)

**What:** Coordinator-driven calendar that books evaluations directly into the clinic's EMR. No patient-facing surface.

**EMR Adapter Pattern:**
- `Emr::BaseAdapter` interface -- IntakeQ is impl #1 of ~18
- Capability spectrum: `:native` (real open slots via EMR API) vs `:best_effort` (configured hours minus synced appointments)
- IntakeQ has native availability via its public booking-widget API

**Booking Flow:**
1. CC selects patient + clinic + provider + time slot
2. Flywheel writes appointment to EMR via adapter
3. Pre-write re-query confirms slot is still open
4. Confirmation-call backstop for dumb-write scenarios (race condition)
5. IntakeQ webhook syncs appointment status back to Flywheel
6. Flywheel creates `evaluations` record + optional GHL writeback

**Key models:** Clinic, Provider, EmrConnection, Patient, Appointment

## P2 -- Connective Health Integration

**What:** Replace Keragon automation. CH delivers FHIR R4 / US Core 6.1.0 bundles -> Flywheel ingests -> maps to EMR fields -> writes to IntakeQ.

**Delivery model:** Request-then-receive (not passive S3 drop):
1. On eval submission, Flywheel POSTs processing request to CH API (per-practice OID + API key)
2. CH builds bundle + PDF asynchronously (prod latency ~12+ hours)
3. CH delivers to Psycle-owned S3 bucket + fires `document.deliveryComplete` webhook
4. Flywheel ingests bundle -> `ch_payloads` (encrypted, replay-safe)
5. Mapping engine: `(ch_fhir | ch_pdf | ghl_crm) -> {EMR, provider, field}`
6. Write mapped data to IntakeQ (Client API + Files API for PDF)

**Qualification:** >= 2 antidepressant trials detected from `MedicationRequest` resources -> mark qualified

**Feature-flagged:** Ships behind a flip. Keragon stays as fallback until proven.

## Connection to Other Features

| This Feature | Connects To | How |
|-------------|-------------|-----|
| EMR appointment write | Feature 05 (evaluations table) | Creates evaluation record on booking |
| EMR appointment status | Feature 06 (patient funnel pages) | Status updates flow to eval table -> UI |
| CH medication data | Feature 05 (mv_contact_funnel) | Qualification flag updated |
| CH clinical data | Feature 06 (contact detail view) | Shows in patient timeline |
| Provider/clinic models | Feature 02 (clinics-orgs-pods) | Shared domain objects |
| GHL writeback | Feature 03 (ghl-sync) | Optional: eval status writes back to GHL contact |

## What's Already Built (in Flywheel repo)

- Rails 8 API scaffolded with ActiveRecord
- Postgres RLS with `flywheel_app` role (proven by isolation tests)
- Sidekiq + Redis for background jobs
- IntakeQ adapter (booking, sync, availability)
- CH webhook receiver + S3 ingest
- Local mocks for CH, GHL, IntakeQ, MinIO
- CI/CD: GitHub Actions with path-filtered deploys to AWS ECS
- Docker Compose dev environment (`just dev`)
