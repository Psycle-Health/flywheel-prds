# Feature 12 -- Onboarding & Intake

**Dependencies:** 02-clinics-orgs-pods, 10-clinic-settings
**Source:** Dashboard prototype (`onboarding.html`), Jun 26 "Clinics Tables" meeting
**Users:** Admin (creates onboarding links), new clinic contacts (fill the form)

---

## Onboarding Wizard (5-Step Form)

Public-facing (no auth required -- accessed via unique link sent to new clinic contact).

### Step 1 -- General Info

**Primary Contact:**
- Full Name*, Title*, Email*, Phone*

**Organization:**
- Clinic Name, Clinic Phone*, Clinic Email*, Website*

**Locations** (primary + add more):
- Address*, Hours*, Email
- Services Offered multi-select per location (SPV, TMS, KET, IM Ketamine, Med Mgmt, Therapy, Accelerated TMS, IOP, PHP) -- color-coded pills
- Per-modality capacity: # of Chairs, Sessions/Day, Days/Week (for Spravato + TMS)
- `+ Add Location` button for multi-site

### Step 2 -- Business & Compliance

**Business Registration:**
- Business Name (declared in EIN)*, Address (as indicated in EIN)*, Tax ID/EIN*, Registration Type* (Sole Proprietorship, Partnership, Corporation, Co-Operative, LLC, Non-Profit)
- CP 575 upload (IRS confirmation of EIN) -- file upload (PDF, PNG, DOCX, DOC)

**BAA (Business Associate Agreement):**
- Scrollable legal text (full BAA content embedded)
- Practice/Organization Name*, Authorized Signer Name*, Title*, Date
- Signature pad (click/draw to sign)
- Checkbox: "I acknowledge that I have read and agree to the terms"*

### Step 3 -- Staff & Providers

**Staff Members:**
- Name, Email, Phone, Role dropdown (Office Manager, Front Desk, Billing Admin, PA Admin, Clinic Admin)
- First staff member pre-populated from Step 1 primary contact
- `+ Add Staff Member`

**Evaluating Providers:**
- Provider Name*, Email*, NPI Number* (10-digit), Provider Type dropdown (Psychiatrist MD/DO, PMHNP, NP, CRNA, PA, Other)
- Bio* (textarea), Availability* (e.g., "M-W: 9-3, F: 11-3")
- `+ Add Provider`

### Step 4 -- EMR, Policies & Costs

**EMR & Operations:**
- EMR dropdown* (Osmind, Tebra, Athena, DrChrono, Advanced MD, IntakeQ, Other)
- EMR Appointment Reminders? (Yes SMS, Yes Email, Yes Both, No)
- Evaluation Locations (in-person, telehealth, etc.)
- Intake Paperwork Method (EMR, PandaDocs, DocuSign, etc.)

**Costs & Policies:**
- Eval Cost*
- Cash-Pay Financing (HSA/FSA, Care Credit, Advance Care Card, In-House, Other, None)
- Cash-Pay Pricing Per Series (textarea)
- Deposit Credited Toward Treatment? (Yes/No radio)
- Deposit Refundable? (Yes/No radio)
- Current Monthly Evals
- Cancellation/No-Show Policy (textarea)

**Intake Forms:**
- Clinic Paperwork upload (PDF, DOCX, max 10 files)
- Clinic Consents upload (PDF, DOCX, max 10 files)

### Step 5 -- Treatment-Specific

**Spravato:**
- Buy & Bill or Pharmacy?* (Buy & Bill + Pharmacy, Pharmacy Only, Buy & Bill Only, No, Coming Soon)
- Avg Reimbursement/Session
- Avg Treatment Duration (<2mo, 2-3mo, 3-6mo, 6-12mo, 12+mo)
- Time to Start After Eval (<=1wk, 1-2wk, 2-4wk, 4+wk)
- Use Spravato with Me? (Yes/No/Not Sure)
- Janssen Savings Program? (Yes/No/Not Sure)

**TMS:**
- Offer TMS?* (Yes, No, Coming Soon)
- Completion Rate (90%+, 75-90%, 50-75%, <50%)
- Avg Reimbursement/Session
- Time for TMS to Start (<=1wk, 1-2wk, 2-4wk, 4+wk)

**Submit button** (green gradient: `linear-gradient(135deg, #4ade80, #609FFC)`)

### Progress Bar
- Top of form: thin gradient fill bar (orange->lavender) showing completion %
- Step tabs: clickable numbered tabs (1-5) with active state highlighting

## On Submit

1. Create `organization` record (from Business Registration fields)
2. Create `clinic` record(s) per location (GHL location ID assigned later via Accounts Matcher)
3. Create `provider` records with NPI + bio + availability
4. Create `user` records for staff + providers (with invitation emails)
5. Store BAA signature + timestamp
6. Upload files to S3 (`{clinicId}/{fieldKey}/{filename}`)
7. Log in `clinic_activity_log` (onboarding_completed)
8. Notify assigned pod lead via Slack
9. Create follow-up tasks: GHL subaccount mapping, Meta ad account setup

## File Upload

- S3 storage (replacing GCS from prototype)
- 10 MB max per file; PDF, PNG, JPG, DOC, DOCX
- Pre-signed URLs (15-min expiration) for downloads
- Key format: `{clinicId}/{fieldKey}/{filename}`

## API Endpoints

```
POST   /api/v1/onboarding                -- Submit full form (creates org + clinics + providers + users)
GET    /api/v1/onboarding/:token          -- Load form for a specific invite link
POST   /api/v1/upload                     -- Upload file to S3
GET    /api/v1/upload/signed-url?key=...  -- Get signed download URL
DELETE /api/v1/upload?key=...             -- Delete file from S3
```
