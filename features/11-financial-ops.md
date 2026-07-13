# Feature 11 -- Financial Ops

**Dependencies:** 02-clinics-orgs-pods, 09-client-reports
**Source:** Internal app (~/Psycle-Internal -- revenue/expenses/contractors/efficiency/CSV upload)
**Users:** Admin only (Danny, Elijah). AM sees pod totals, not clinic-level breakdown for other pods.

---

## Schema

```sql
CREATE TABLE internal_revenues (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  clinic_id TEXT NOT NULL REFERENCES clinics(id),
  period_start DATE NOT NULL,
  period_end DATE NOT NULL,
  mrr NUMERIC(12,2),
  contract_value NUMERIC(12,2),
  payment_status TEXT DEFAULT 'current',  -- current, overdue, cancelled, paused
  payment_date DATE,
  source TEXT DEFAULT 'csv',              -- csv, stripe, quickbooks
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE internal_upsells (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  clinic_id TEXT NOT NULL REFERENCES clinics(id),
  upsell_type TEXT NOT NULL,              -- additional_location, premium_tier, add_on_service
  value NUMERIC(12,2) NOT NULL,
  status TEXT DEFAULT 'proposed',         -- proposed, accepted, declined, active, churned
  proposed_date DATE,
  closed_date DATE,
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE internal_expenses (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  clinic_id TEXT REFERENCES clinics(id),  -- NULL = company-wide
  category TEXT NOT NULL,                 -- payroll, software, advertising, office, travel, other
  vendor TEXT,
  description TEXT,
  amount NUMERIC(12,2) NOT NULL,
  expense_date DATE NOT NULL,
  is_recurring BOOLEAN DEFAULT false,
  recurrence_period TEXT,                 -- monthly, quarterly, annual
  source TEXT DEFAULT 'csv',
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE internal_contractors (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  name TEXT NOT NULL,
  email TEXT,
  role TEXT,                              -- designer, developer, consultant
  rate_type TEXT DEFAULT 'hourly',        -- hourly, monthly, project
  rate NUMERIC(10,2),
  hours_per_month NUMERIC(6,1),
  is_active BOOLEAN DEFAULT true,
  start_date DATE,
  end_date DATE,
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE internal_contractor_assignments (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  contractor_id TEXT NOT NULL REFERENCES internal_contractors(id) ON DELETE CASCADE,
  clinic_id TEXT REFERENCES clinics(id),  -- NULL = company-wide
  pod_id TEXT REFERENCES pods(id),
  monthly_cost NUMERIC(10,2),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE internal_csv_uploads (
  id TEXT PRIMARY KEY DEFAULT gen_random_uuid()::text,
  uploaded_by TEXT NOT NULL REFERENCES users(id),
  target_table TEXT NOT NULL,             -- revenues, expenses, upsells, contractors
  file_name TEXT NOT NULL,
  rows_imported INTEGER DEFAULT 0,
  rows_failed INTEGER DEFAULT 0,
  error_log JSONB DEFAULT '[]',
  status TEXT DEFAULT 'processing',       -- processing, completed, failed, partial
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## Dashboard Pages

### Revenue
- **KPI Cards:** Total MRR, MRR Growth %, Avg Revenue/Clinic, Total Active Clients
- **Table:** Clinic name, pod, MRR, contract value, payment status, start date, contract type
- **Access:** Admin = all clinics. AM = own pod detail + other pod totals (not clinic-level).

### Upsells
- **KPI Cards:** Total Upsell Revenue, Conversion Rate, Avg Value, Pipeline Value
- **Table:** Clinic, pod, upsell type, value, status, proposed/closed dates

### Expenses
- **KPI Cards:** Total Monthly, Category Breakdown, Burn Rate, Net Margin
- **Table:** Category, vendor, description, amount, date, recurring flag
- **Chart:** Category breakdown donut

### Contractors
- **KPI Cards:** Total Spend, Active Count, Avg Rate, Utilization
- **Table:** Name, role, rate, hours/month, total cost, assigned clinics/pods

### Efficiency (Derived -- no direct input)
- **KPI Cards:** Net Margin %, Revenue/Employee, CAC, LTV:CAC Ratio
- **Table:** Per-clinic: revenue, ad spend, total cost, margin, ROI
- **Source:** Joins internal_revenues + internal_expenses + mv_clinic_spend_daily

## CSV Upload Pipeline

1. User selects target (Revenues, Expenses, Upsells, Contractors)
2. Uploads CSV file
3. Server validates headers match expected schema (Zod)
4. Maps `clinic_name` -> `clinic_id` (fuzzy match with confirmation UI)
5. Inserts rows, tracking per-row successes and failures
6. Creates `internal_csv_uploads` log entry
7. UI shows: X imported, Y failed, with per-row error details

**Expected CSV formats:**
```
Revenues:  clinic_name, period_start, period_end, mrr, contract_value, payment_status, payment_date, notes
Expenses:  clinic_name, category, vendor, description, amount, expense_date, is_recurring, recurrence_period, notes
Upsells:   clinic_name, upsell_type, value, status, proposed_date, closed_date, notes
Contractors: name, email, role, rate_type, rate, hours_per_month, start_date, notes
```

## API Endpoints

```
-- Revenue
GET    /api/v1/finance/revenues?pod_ids[]&date_start&date_end
POST   /api/v1/finance/revenues/upload     -- CSV upload

-- Upsells
GET    /api/v1/finance/upsells?pod_ids[]
POST   /api/v1/finance/upsells/upload

-- Expenses
GET    /api/v1/finance/expenses?pod_ids[]&category
POST   /api/v1/finance/expenses/upload

-- Contractors
GET    /api/v1/finance/contractors?active_only
POST   /api/v1/finance/contractors/upload

-- Efficiency
GET    /api/v1/finance/efficiency?pod_ids[]&date_start&date_end

-- Upload logs
GET    /api/v1/finance/uploads              -- Upload history
```

## Workflows

- **Payment overdue alert:** Revenue payment_status -> 'overdue' triggers Slack notification (30/60/90 day escalation)
- **Contract renewal tracking:** clinic_retainers approaching end -> in-app notification + Slack to AM
