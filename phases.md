# Docx_Care — 7-Day Production Sprint Plan

**Goal:** Deliver a fully tested, production-hardened Clinic Management System on StackCP shared hosting within 7 working days.  
**Execution Methodology:** Incremental vertical slices, Native Session Auth, pure Blade + Vanilla CSS (with native Light/Dark modes), and zero third-party framework overhead.

---

## Sprint Overview Matrix

| Day | Focus Area | Core Modules Delivered | Verification Standard |
| :---: | :--- | :--- | :--- |
| **Day 1** | **Foundation & Auth** | MariaDB schema base, Native Session Auth, Vanilla CSS Design System (Light/Dark), Layout shells. | Multi-theme toggle works; Admin/Doctor session login redirects to respective layouts. |
| **Day 2** | **Public Experience** | Public website, Doctor directory, Clinic & Home visit booking engine (`REQ-2026-XXXX`), Doctor registration. | End-to-end booking generates triage record; validation rejects incomplete submissions. |
| **Day 3** | **Admin Operations** | Operational Dashboard (6 KPI cards, P&L chart, latest requests table), Doctor approvals, Appointment triage. | Dashboard metrics populate live from DB; triage updates status to `voting_open`. |
| **Day 4** | **Doctor Portal** | Doctor onboarding, subscription fee proof upload & admin approval, Appointment board with Blind Voting. | Doctor expresses interest; aggregate count increments; peer identities remain 100% hidden. |
| **Day 5** | **Clinical Delivery** | Admin doctor assignment, assigned treatments view, full patient address reveal, clinical session notes. | Doctor logs treatment progress; appointment completes; billing record initializes. |
| **Day 6** | **Financial Engine** | Patient Billing, payment receipts (PKR), Doctor payouts ledger, Clinic P&L and operational expense tracking. | Outstanding balance and liability cards update in real-time; financial transactions are atomic. |
| **Day 7** | **Hardening & Deploy** | Privacy audit, CSRF/Rate-limit verification, local PC asset build, SSH upload & live StackCP configuration. | Smoke tests pass on StackCP; `APP_DEBUG=false`; MariaDB 10.11.18 migrations clean. |

---

## Detailed Daily Implementation Schedule

### Day 1: System Baseline, Native Auth & Vanilla CSS Design System
- **Engineering Tasks:**
  1. Initialize clean Laravel 11/12 project; lock environment to PHP 8.4 and MariaDB 10.11.x in `.env`.
  2. Implement migrations for `users` and `settings`. Seed default administrative superuser.
  3. Build `AuthController` with native session mechanisms (`Auth::attempt()`, session regeneration, logout invalidation).
  4. Build `resources/css/app.css` containing complete design token definitions (`--brand-green: #16a34a`, `--bg-sidebar: #0b0d10`, Light/Dark variables).
  5. Build reusable layout templates:
     - `layouts/admin.blade.php`: Fixed 260px dark sidebar with DOCX brand logo, subtext, active route pill, top bar, and theme toggle.
     - `layouts/doctor.blade.php`: Tailored navigation for verified medical practitioners.
     - `layouts/public.blade.php`: Clean responsive header and footer.
  6. Implement `resources/js/theme.js` for instant dark/light switching without layout shift.
- **Definition of Done (DoD):**
  - Application boots; login screen toggles between Light and Dark mode; valid credentials route to dashboard shells.

---

### Day 2: Public Website & Clinic/Home Visit Booking Engine
- **Engineering Tasks:**
  1. Implement migrations and models for `services` and `appointments`.
  2. Build public landing page displaying physiotherapy & hijama services with pricing and duration.
  3. Build public Doctor Directory displaying approved doctor credentials, specialties, and visit capabilities.
  4. Build Doctor Onboarding application form (`/join-doctor`): collects PMDC/license number, CNIC, and bank details.
  5. Build Appointment Request Form (`/book`):
     - Dynamic visit type selection: **Clinic Visit** vs. **Home Visit**.
     - Conditional DOM rendering: when "Home Visit" is selected, patient address and landmark fields become required.
     - Server-side FormRequest validation enforcing date constraints (future dates only).
     - Unique tracking code generator: format `REQ-YYYY-XXXX` (e.g., `REQ-2026-0041`).
     - Friendly confirmation page with reference tracking.
- **Definition of Done (DoD):**
  - Booking request creates a `triage_pending` record in MariaDB; doctor registration sets account status to `pending`.

---

### Day 3: Admin Operational Dashboard & Triage System
- **Engineering Tasks:**
  1. Build `AdminDashboardController` matching the confirmed UI preview:
     - Metric 1: *New appointment requests* (`count(*) WHERE status = 'triage_pending'`).
     - Metric 2: *Doctors pending approval* (`count(*) WHERE is_approved = 0`).
     - Metric 3: *Subscription proofs to verify* (`count(*) WHERE status = 'pending'`).
     - Metric 4: *Unassigned appointments* (`count(*) WHERE assigned_doctor_id IS NULL AND status != 'cancelled'`).
     - Metric 5: *Outstanding patient balance* (`SUM(balance_amount)` formatted as `PKR 184,500`).
     - Metric 6: *Doctor balances due* (`SUM(net_payout) WHERE status = 'due'` formatted as `PKR 96,200` in dark inverted card).
  2. Implement Latest Appointment Requests Table:
     - Columns: Ref, Patient Name, Category, Visit (Pill badge: `Home` teal or `Clinic` slate), Preferred Date.
     - Quick action button: `Open triage`.
  3. Implement responsive 6-Month Clinic P&L Chart widget using lightweight SVG bars.
  4. Build Triage Queue (`/admin/triage`): Admin reviews request and clicks "Open for Voting", moving status to `voting_open`.
  5. Build Doctor Approvals Screen (`/admin/approvals`): Admin inspects CNIC/license and approves doctor.
- **Definition of Done (DoD):**
  - Dashboard renders all 6 metrics accurately from database; triage updates appointment status to `voting_open`.

---

### Day 4: Doctor Portal, Subscriptions & Blind Voting Board
- **Engineering Tasks:**
  1. Implement migrations and models for `doctor_profiles`, `subscriptions`, and `appointment_votes`.
  2. Build Doctor Dashboard with subscription status banner (Active vs. Grace Period vs. Expired).
  3. Build Doctor Subscription Lifecycle:
     - Payment details card showing clinic bank IBAN for monthly dues.
     - Proof upload form: accepts PDF/JPG/PNG up to 5MB; stores file with UUID in `storage/app/public/proofs/`.
     - Admin verification interface to validate receipt and set `start_date` and `end_date`.
  4. Build Doctor Appointment Board (`/doctor/board`):
     - Displays `voting_open` appointments showing Category, Visit Type, and Area/City (Home visit exact address remains hidden).
     - Doctors can express interest ("Interested") or decline ("Pass").
     - **Blind Voting Security Enforcement:** Query uses `withCount('votes')`. Doctor view **never** loads voter identities or peer records.
     - Enforce `uq_appointment_doctor_vote` unique key to prevent duplicate bids.
- **Definition of Done (DoD):**
  - Doctor can submit interest; aggregate counter updates; SQL query audit confirms zero leakage of peer doctor IDs.

---

### Day 5: Doctor Assignment & Clinical Session Notes
- **Engineering Tasks:**
  1. Implement migration and model for `session_notes`.
  2. Build Admin Doctor Assignment Interface (`/admin/assignment/{id}`):
     - Admin views appointment and the list of doctors who expressed interest during the blind voting stage.
     - Admin assigns the chosen doctor.
     - Wrapped in `DB::transaction`: status changes to `assigned`, `assigned_doctor_id` is set, and a `patient_billings` record is initialized.
  3. Build Doctor Assigned Treatments View (`/doctor/treatments`):
     - Only the assigned doctor can now view the patient's full phone number and complete home visit address.
     - Status updates to `in_progress`.
  4. Build Clinical Session Progress Notes Interface:
     - Doctor logs session details: Session number, Pain scale before/after (1-10), treatment administered, next session date.
     - Option to mark treatment as `completed`.
- **Definition of Done (DoD):**
  - Complete workflow passes from doctor assignment to session logging and final clinical completion.

---

### Day 6: Financial Engine (Billing, Doctor Payouts & P&L Ledger)
- **Engineering Tasks:**
  1. Implement migrations and models for `patient_billings`, `doctor_payouts`, and `clinic_finances`.
  2. Build Patient Billing Management (`/admin/billing`):
     - Generates invoice `INV-YYYY-XXXX` with total, paid, and balance in PKR.
     - "Record Payment" modal: accepts partial or full payment with method (Cash, Bank Transfer, EasyPaisa, JazzCash).
     - Automatically logs a `revenue` credit entry into `clinic_finances`.
  3. Build Doctor Payouts Engine (`/admin/payouts`):
     - Calculates doctor's share based on doctor's agreed commission rate (e.g., 70%).
     - Admin records disbursement with bank reference; updates status to `paid` and logs `doctor_payout` debit in `clinic_finances`.
  4. Build Clinic P&L & Capital Screen (`/admin/pnl`):
     - Ledger summarizing monthly Revenue, Doctor Disbursements, Operating Expenses, and Net Profit.
     - Form to record operational overheads (Clinic rent, utilities, clinical supplies).
- **Definition of Done (DoD):**
  - Financial balances reconcile accurately; all numbers update seamlessly on the Admin Dashboard KPI cards.

---

### Day 7: Security Audit, Local Asset Build & StackCP Deployment
- **Engineering Tasks:**
  1. **Security & Privacy Audit:**
     - Execute test suite verifying role boundaries (Doctor cannot open Admin routes; Guest cannot open protected portals).
     - Verify blind voting queries never expose peer identifiers.
  2. **Asset Build on Development PC:**
     - Execute `npm run build` locally. Confirm `public/build/manifest.json` and compiled assets.
  3. **StackCP Deployment via SSH:**
     - Deploy code repository to StackCP Linux hosting.
     - Ensure document root points strictly to `/public`.
     - Configure `.env`: set `APP_ENV=production`, `APP_DEBUG=false`, MariaDB credentials.
     - Run migrations: `php artisan migrate --force`.
     - Execute `php artisan config:cache`, `php artisan route:cache`, `php artisan view:cache`.
     - Run `php artisan storage:link`.
  4. **Production Smoke Testing:**
     - Verify login, booking submission, blind interest submission, and admin dashboard rendering on live URL.
- **Definition of Done (DoD):**
  - System is live, fully operational, secure, and ready for clinic use on StackCP.
