# Docx_Care — AI Development Prompts (Production Sprint)

**Product:** Docx_Care  
**Entity:** DOCX CLUB SMC-PVT Ltd.  
**Baseline Stack:** PHP 8.4, Laravel 11/12, MariaDB 10.11.x InnoDB, Blade + Vanilla CSS + Vanilla JS (No Tailwind), Native Session Auth, StackCP.  
**Theme:** Light & Dark mode via CSS variables (`--brand-green: #16a34a`, `--bg-sidebar: #0b0d10`).  
**Currency & Time:** `PKR` formatted as `PKR 184,500` | Stored in `UTC`, displayed in `Asia/Karachi`.

---

## 1. Master System Rules for AI Execution
1. **Zero Tailwind:** Write semantic Vanilla CSS with CSS custom properties for Light/Dark modes in `resources/css/app.css`.
2. **Native Authentication Only:** Use Laravel's native session mechanisms (`Auth::attempt()`, `$request->session()->regenerate()`). Do not install Breeze, Jetstream, or Fortify.
3. **Strict Blind Voting:** Doctors must never receive peer doctor IDs, names, individual choices, or identifiable timestamps in queries, models, or Blade views. Query aggregate counts only (`withCount`).
4. **MariaDB 10.11 Compatibility:** Write strict MariaDB migrations with foreign keys, composite indexes, and utf8mb4 encoding. No SQLite in-memory dialect.
5. **Atomic Transactions:** Wrap multi-record mutations (assignment + billing, payment + ledger entry) inside `DB::transaction()`.
6. **No Node on Production:** Assets are built on local PC via `npm run build` and committed/uploaded to `public/build/`.

---

## 2. Master Senior Engineer Prompt Header

Prepend this instruction when prompting an AI agent to execute a task:

> Act as a senior principal Laravel engineer building **Docx_Care** for **DOCX CLUB SMC-PVT Ltd.**. 
> Adhere strictly to the baseline: PHP 8.4, Laravel 11/12, MariaDB 10.11.x, Blade with Vanilla CSS (Dark/Light mode via CSS variables, strictly NO Tailwind), Vanilla JS, and Native Session Auth. 
> Doctor portal queries must strictly enforce Blind Voting privacy (aggregate counts only). Financial amounts are in PKR. Ensure state transitions are atomic using `DB::transaction()`. Write clean controllers, FormRequests, models with type-hinted relationships, and Blade views. Return production-ready code with no placeholder shortcuts.

---

## 3. Production Sprint Execution Prompts (Day-by-Day)

---

### Day 1: Project Setup, Native Session Auth & Vanilla CSS Design System
```text
Task: Initialize Docx_Care core foundation, Native Session Auth, and the Vanilla CSS Design System with Light/Dark Mode.

Context: DOCX CLUB SMC-PVT Ltd. Clinic Management System. No Tailwind, no Breeze.

Files to Create/Update:
- .env (MariaDB 10.11 configuration, APP_TIMEZONE=UTC, APP_LOCALE=en)
- database/migrations/0001_01_01_000000_create_users_table.php (users table: id, name, email, phone, password, role: admin/staff/doctor, status: active/pending/suspended)
- database/migrations/xxxx_create_settings_table.php (settings table: key, value)
- app/Models/User.php (fillable, casts, role helper methods: isAdmin(), isDoctor())
- app/Http/Controllers/Auth/AuthController.php (showLoginForm, login with rate-limiting, logout with session invalidation)
- app/Http/Middleware/EnsureRole.php (enforce role check on routes)
- resources/css/app.css (Vanilla CSS tokens: brand green #16a34a, dark sidebar #0b0d10, cards #ffffff / #12161c, light/dark variables, buttons, tables, forms, alerts)
- resources/js/theme.js (vanilla script toggling [data-theme="dark"] on <html> and storing in localStorage)
- resources/views/layouts/admin.blade.php (260px dark sidebar with DOCX logo, "Docx_Care / DOCX CLUB SMC-PVT Ltd." subtitle, navigation links with active green pills, header with Admin avatar pill, and theme toggle)
- resources/views/layouts/doctor.blade.php (doctor-specific sidebar navigation and header)
- resources/views/layouts/public.blade.php (clean responsive header with logo, navigation links, and booking CTA)
- resources/views/auth/login.blade.php (centered card login supporting dark/light mode)
- database/seeders/DatabaseSeeder.php (creates initial admin: admin@docxcare.com / password123)

Acceptance Criteria:
1. `php artisan migrate --seed` runs without errors on MariaDB.
2. Visiting `/login` renders a polished form matching the dark/light design system.
3. Submitting valid admin credentials logs the user in, regenerates session, and redirects to `/admin/dashboard`.
4. Role middleware blocks non-admin users from accessing admin routes with 403.
5. Clicking the theme toggle switches between Light and Dark mode without page reload or FOUC.
```

---

### Day 2: Public Website & Clinic/Home Visit Booking Engine
```text
Task: Build the Public Website, Clinic Services Showcase, Doctor Directory, and the Clinic & Home Visit Booking Engine.

Files to Create/Update:
- database/migrations/xxxx_create_services_table.php (id, title, slug, category, description, duration_minutes, base_price, is_active)
- database/migrations/xxxx_create_appointments_table.php (id, reference_no, patient_name, patient_phone, patient_email, patient_address, category, visit_type: clinic/home, preferred_date, preferred_time, clinical_notes, assigned_doctor_id, status: triage_pending/voting_open/assigned/in_progress/completed/cancelled)
- app/Models/Service.php & app/Models/Appointment.php
- app/Http/Requests/StoreAppointmentRequest.php (validation: patient_name required, patient_phone required, category required, visit_type in:clinic,home; patient_address required if visit_type == home; preferred_date future date)
- app/Http/Controllers/Public/HomeController.php (landing page with services, trust points)
- app/Http/Controllers/Public/DoctorDirectoryController.php (lists approved doctors with specialty and visit capabilities)
- app/Http/Controllers/Public/DoctorSignupController.php (handles doctor registration: PMDC license, CNIC, specialty, bank details; sets status to pending)
- app/Http/Controllers/Public/BookingController.php (create, store, and confirmation methods)
- resources/views/public/home.blade.php, /services.blade.php, /doctors.blade.php, /join-doctor.blade.php
- resources/views/public/book.blade.php (booking form with dynamic JS: selecting 'Home Visit' reveals address and landmark inputs)
- resources/views/public/confirmation.blade.php (success screen showing tracking reference REQ-YYYY-XXXX)

Acceptance Criteria:
1. Booking a "Clinic Visit" creates an appointment with `visit_type = clinic` and auto-generated `reference_no` (e.g., `REQ-2026-0041`).
2. Booking a "Home Visit" validates that `patient_address` is provided.
3. Doctor signup stores pending user and doctor profile awaiting admin verification.
4. All forms feature responsive Vanilla CSS styling, inline error states, and CSRF protection.
```

---

### Day 3: Admin Operational Dashboard & Triage System
```text
Task: Build the Admin Operational Dashboard matching the confirmed UI preview, Doctor Approvals, and Appointment Triage.

Files to Create/Update:
- app/Http/Controllers/Admin/AdminDashboardController.php
- app/Http/Controllers/Admin/TriageController.php
- app/Http/Controllers/Admin/DoctorApprovalController.php
- resources/views/admin/dashboard.blade.php (Exact UI preview replica):
  * Header: "Operational Dashboard" with subtitle "Sample data for design preview" and Admin avatar pill.
  * 6 KPI Summary Cards:
    - Card 1: "New appointment requests" (Count of status = triage_pending)
    - Card 2: "Doctors pending approval" (Count of doctor_profiles where is_approved = 0)
    - Card 3: "Subscription proofs to verify" (Count of subscriptions where status = pending)
    - Card 4: "Unassigned appointments" (Count where assigned_doctor_id IS NULL and status != cancelled)
    - Card 5: "Outstanding patient balance" (Formatted: PKR 184,500)
    - Card 6: "Doctor balances due" (Dark card styling, Formatted: PKR 96,200)
  * Latest Appointment Requests Table:
    - Columns: Ref, Patient, Category, Visit (Badge: 'Home' teal or 'Clinic' dark slate), Preferred Date.
    - Quick action button: "Open triage" leading to triage queue.
  * Clinic P&L Widget (Last 6 Months):
    - Lightweight inline responsive SVG bar chart.
    - Summary metrics: Revenue, Doctor earnings, Expenses, Net profit (all in PKR).
- resources/views/admin/triage/index.blade.php (triage queue: Admin can review requests and click "Open for Voting" -> status becomes 'voting_open')
- resources/views/admin/doctors/approvals.blade.php (table of pending doctors with one-click "Approve" button)

Acceptance Criteria:
1. Dashboard queries live database counts and sums; zero hardcoded mockup figures in production mode.
2. Clicking "Open for Voting" in Triage updates appointment status to `voting_open`.
3. Approving a doctor updates `is_approved = true` and `users.status = 'active'`.
4. Layout matches the dark navigation sidebar and clean card hierarchy of the preview screenshot.
```

---

### Day 4: Doctor Portal, Subscriptions & Blind Voting Board
```text
Task: Implement the Doctor Portal, Subscription Proof Verification, and the Blind Interest Voting Board.

Files to Create/Update:
- database/migrations/xxxx_create_doctor_profiles_table.php (user_id, specialty, license_number, cnic, bio, photo_path, visit_capabilities, bank details, is_approved)
- database/migrations/xxxx_create_subscriptions_table.php (doctor_id, plan_name, amount, proof_image_path, status: pending/active/rejected/expired, start_date, end_date, verified_by)
- database/migrations/xxxx_create_appointment_votes_table.php (appointment_id, doctor_id, vote_type: interested/unavailable, remarks, UNIQUE(appointment_id, doctor_id))
- app/Models/DoctorProfile.php, Subscription.php, AppointmentVote.php
- app/Http/Middleware/EnsureActiveSubscription.php (checks if doctor subscription is active before accessing board)
- app/Http/Controllers/Doctor/DoctorDashboardController.php
- app/Http/Controllers/Doctor/SubscriptionController.php (upload payment slip: max 5MB PDF/JPG/PNG; store in storage/app/public/proofs/)
- app/Http/Controllers/Doctor/AppointmentBoardController.php (lists appointments where status = 'voting_open')
- app/Http/Controllers/Admin/SubscriptionAdminController.php (admin verifies proof and activates subscription)
- resources/views/doctor/dashboard.blade.php, /subscription.blade.php, /board.blade.php
- resources/views/admin/subscriptions/index.blade.php

Acceptance Criteria:
1. Doctor uploads monthly subscription fee receipt; Admin can verify and activate membership.
2. On `/doctor/board`, doctor sees open appointments with category, visit type, and general area.
3. Blind Voting Privacy Rule: The query MUST use `withCount('votes as total_interested')`. Doctor views and JSON responses must NEVER load other doctors' IDs, names, or individual votes.
4. Doctor clicks "Show Interest": creates record in `appointment_votes`. Unique constraint prevents duplicate bids.
```

---

### Day 5: Doctor Assignment Workflow & Clinical Session Notes
```text
Task: Build Admin Doctor Assignment and the Doctor Session Progress Notes Tracking Workflow.

Files to Create/Update:
- database/migrations/xxxx_create_session_notes_table.php (appointment_id, doctor_id, session_number, pain_scale_before, pain_scale_after, symptoms_observed, treatment_provided, patient_response, next_session_date)
- app/Models/SessionNote.php
- app/Http/Controllers/Admin/AssignmentController.php:
  * index: lists appointments with status 'voting_open' and interested doctor counts.
  * show: displays details and lists all doctors who expressed interest under blind voting.
  * assign: wraps in DB::transaction -> updates assigned_doctor_id, changes status to 'assigned', and initializes patient_billings record.
- app/Http/Controllers/Doctor/TreatmentController.php:
  * index: lists appointments assigned to logged-in doctor.
  * show: displays complete patient details and unmasked home visit address.
  * addNote: records clinical progress (symptoms, treatment applied, pain scale 1-10).
  * complete: marks treatment completed (status = 'completed').
- resources/views/admin/assignment/index.blade.php, /show.blade.php
- resources/views/doctor/treatments/index.blade.php, /show.blade.php

Acceptance Criteria:
1. Admin selects a doctor from the interested pool and executes assignment.
2. In `DB::transaction`, appointment status becomes `assigned`, and `patient_billings` invoice is created.
3. Only the officially assigned doctor can access patient contact details and home visit address.
4. Doctor can log session progress notes and mark the treatment completed.
```

---

### Day 6: Financial Engine (Billing, Doctor Payouts & Clinic P&L)
```text
Task: Implement Patient Invoicing, Doctor Payout Disbursements, and the Clinic P&L Financial Ledger in PKR.

Files to Create/Update:
- database/migrations/xxxx_create_patient_billings_table.php (appointment_id, invoice_number, total_amount, paid_amount, balance_amount, status: unpaid/partial/paid, payment_method)
- database/migrations/xxxx_create_doctor_payouts_table.php (payout_number, doctor_id, appointment_id, gross_amount, clinic_commission, net_payout, status: due/paid, transaction_ref, paid_at, processed_by)
- database/migrations/xxxx_create_clinic_finances_table.php (type: revenue/doctor_payout/expense/capital, category, amount, description, reference_id, entry_date)
- app/Models/PatientBilling.php, DoctorPayout.php, ClinicFinance.php
- app/Http/Controllers/Admin/BillingController.php (lists invoices, records patient payments, updates balance in PKR, posts revenue credit to clinic_finances)
- app/Http/Controllers/Admin/PayoutController.php (tracks doctor earnings due, disburses payouts with bank transaction ref, posts debit to clinic_finances)
- app/Http/Controllers/Admin/PnLController.php (monthly ledger summary, log clinic overhead expenses like rent and utilities)
- resources/views/admin/billing/index.blade.php
- resources/views/admin/payouts/index.blade.php
- resources/views/admin/pnl/index.blade.php

Acceptance Criteria:
1. Recording a patient payment updates `paid_amount` and recalculates `balance_amount = total_amount - paid_amount` server-side.
2. System logs revenue credit to `clinic_finances`.
3. Disbursing a doctor payout logs debit in `clinic_finances` and updates payout status to `paid`.
4. Admin Dashboard KPI cards ("Outstanding patient balance" and "Doctor balances due") update live in PKR.
```

---

### Day 7: Testing, Optimization & StackCP Deployment
```text
Task: Execute Security Audits, Verify Responsive Light/Dark Themes, Build Assets Locally, and Deploy to StackCP.

Files to Create/Update:
- tests/Feature/BlindVotingPrivacyTest.php (tests that doctor endpoint returns count only, never voter IDs)
- tests/Feature/RoleAccessTest.php (tests that doctor cannot access /admin/* and guest cannot access portals)
- public/.htaccess (Apache security: rewrite engine, block access to .env, .git, and disable PHP execution in uploads)

Execution Commands:
1. Run automated test suite: `php artisan test`
2. Compile frontend assets locally: `npm run build`
3. Verify asset manifest: check `public/build/manifest.json`
4. Prepare deployment payload (exclude node_modules, tests, .git)
5. SSH to StackCP:
   - Configure .env (APP_ENV=production, APP_DEBUG=false, DB credentials)
   - Set document root to /public
   - Run: `php artisan migrate --force`
   - Run: `php artisan config:cache && php artisan route:cache && php artisan view:cache`
   - Run: `php artisan storage:link`

Acceptance Criteria:
1. Automated tests pass with 0 failures on MariaDB.
2. Blind voting privacy mathematically and architecturally validated.
3. Production site live on StackCP over HTTPS with Light and Dark modes functioning smoothly.
```
