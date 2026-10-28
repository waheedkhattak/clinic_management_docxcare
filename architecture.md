# Docx_Care — Architecture & Technical Specifications

**Product:** Docx_Care  
**Entity:** DOCX CLUB SMC-PVT Ltd.  
**Domain:** Multi-Specialty Clinic Management System (Specializing in Physiotherapy, Hijama & Home Healthcare)  
**Architecture Pattern:** High-Performance Modular Laravel Monolith (Zero microservice overhead)  
**Execution Target:** Production-Grade 7-Day Sprint Delivery  

---

## 1. Executive System Topology

Docx_Care is architected as an integrated three-tier portal system running on a unified Laravel core and a single MariaDB database. Domain isolation is strictly enforced via Route Groups, Controllers, and Middleware.

```text
                                  +------------------------------------+
                                  |             Docx_Care              |
                                  |      (DOCX CLUB SMC-PVT Ltd.)      |
                                  +------------------------------------+
                                       |            |            |
                     +-----------------+            |            +-----------------+
                     v                              v                              v
        +-------------------------+   +-------------------------+   +-------------------------+
        |   1. Public Website     |   |    2. Doctor Portal     |   |    3. Admin Portal      |
        +-------------------------+   +-------------------------+   +-------------------------+
        | • Clinic Services       |   | • Doctor Onboarding     |   | • Operational Dashboard |
        | • Doctor Directory      |   | • Subscription Status   |   | • Doctor Approvals      |
        | • Appointment Booking   |   | • Appointment Board     |   | • Appointment Triage    |
        |   (Clinic & Home Visits)|   | • Blind Interest/Voting |   | • Doctor Assignment     |
        | • Doctor Registration   |   | • Assigned Treatments   |   | • Patient Billing       |
        |   & Subscription Info   |   | • Session Notes/Progress|   | • Doctor Payouts        |
        | • Contact & About       |   | • Earnings & Payouts    |   | • Clinic P&L & Capital  |
        +-------------------------+   +-------------------------+   +-------------------------+
                     |                              |                              |
                     +------------------------------+------------------------------+
                                                    |
                                                    v
                               +----------------------------------------+
                               |     Core Routing, Auth & Middleware    |
                               +----------------------------------------+
                                                    |
                                                    v
                               +----------------------------------------+
                               |    MariaDB 10.11.x (InnoDB Engine)     |
                               +----------------------------------------+
```

---

## 2. Technical Baseline & Hosting Environment

Every component is locked to ensure strict compatibility with the production hosting environment:

| Layer | Technology | Specification & Implementation Rules |
| :--- | :--- | :--- |
| **PHP Runtime** | **PHP 8.4.x** | Same minor version for Web (FPM/Apache) and CLI. Strict types enabled (`declare(strict_types=1);`). |
| **Core Framework**| **Laravel 11.x / 12.x** | Latest LTS/stable monolith. Lean architecture without third-party boilerplate packages. |
| **Database** | **MariaDB 10.11.x** | Production version: **10.11.18 InnoDB**. Strict SQL mode, foreign keys, utf8mb4 encoding. |
| **Frontend UI** | **Blade + Vanilla CSS** | Custom CSS design token architecture. **Tailwind CSS is strictly excluded**. Built-in native Light and Dark theme switching. |
| **Interactivity** | **Vanilla JavaScript** | Modern ES6 modules for modals, responsive navigation, SVG bar chart rendering, and theme management. |
| **Asset Pipeline**| **Vite (Local Only)** | Assets are compiled on developer PC (`npm run build`). Pre-compiled `/public/build` assets are pushed via Git/SSH. **No Node.js runtime on production**. |
| **Authentication**| **Native Session Auth** | Standard Laravel Session authentication (`Auth::attempt()`). Zero dependency on Breeze or Fortify. |
| **Test Engine** | **PHPUnit / Pest** | Executed directly against MariaDB (in-memory SQLite is prohibited to preserve MariaDB dialect fidelity). |
| **Temporal Standard**| **UTC + PKT** | All timestamps stored in **UTC**; displayed in **`Asia/Karachi`** (UTC+5). |
| **Currency** | **PKR** | Pakistani Rupee stored as `DECIMAL(12,2)`, rendered as `PKR 184,500`. |
| **Production Host**| **StackCP "Linux Unlimited"**| Linux shared environment. SSH deployment. Document root strictly pointed to `/public`. |

---

## 3. Vanilla CSS Design Token Architecture

The UI design is modeled after the confirmed Docx_Care Admin UI preview. It uses CSS custom properties (`:root` and `[data-theme="dark"]`) to deliver instant, flicker-free theme switching.

### 3.1 Design Tokens (`resources/css/app.css`)
```css
:root {
    /* Brand Accent Colors */
    --brand-green: #16a34a;
    --brand-green-hover: #15803d;
    --brand-green-accent: #22c55e;
    --brand-green-subtle: #f0fdf4;

    /* Light Theme Palette */
    --bg-canvas: #f8fafc;
    --bg-surface: #ffffff;
    --bg-sidebar: #0b0d10;
    --border-color: #e2e8f0;
    --border-subtle: #f1f5f9;

    /* Text Colors */
    --text-primary: #0f172a;
    --text-secondary: #475569;
    --text-muted: #64748b;
    --text-inverse: #ffffff;

    /* Status Badges */
    --badge-home-bg: #ccfbf1;
    --badge-home-text: #0f766e;
    --badge-clinic-bg: #0f172a;
    --badge-clinic-text: #ffffff;
    --badge-pending-bg: #fef3c7;
    --badge-pending-text: #92400e;

    /* Metrics & Geometry */
    --sidebar-width: 260px;
    --header-height: 70px;
    --radius-sm: 6px;
    --radius-md: 10px;
    --radius-lg: 14px;
    --shadow-card: 0 1px 3px 0 rgb(0 0 0 / 0.1), 0 1px 2px -1px rgb(0 0 0 / 0.1);
}

[data-theme="dark"] {
    --bg-canvas: #090b0e;
    --bg-surface: #12161c;
    --bg-sidebar: #07090b;
    --border-color: #1f2937;
    --border-subtle: #19212c;

    --text-primary: #f8fafc;
    --text-secondary: #cbd5e1;
    --text-muted: #94a3b8;
    --text-inverse: #ffffff;

    --brand-green-subtle: rgba(22, 163, 74, 0.15);
    --badge-home-bg: rgba(20, 184, 166, 0.2);
    --badge-home-text: #2dd4bf;
    --badge-clinic-bg: #1e293b;
    --badge-clinic-text: #f8fafc;
    --shadow-card: 0 4px 6px -1px rgb(0 0 0 / 0.3), 0 2px 4px -2px rgb(0 0 0 / 0.3);
}
```

### 3.2 Key UI Components
1. **Sidebar Navigation:** Fixed `260px` dark bar (`--bg-sidebar: #0b0d10`) with top logo pill ("DOCX / Docx_Care / DOCX CLUB SMC-PVT Ltd."). Active navigation items render with solid brand green background (`#16a34a`) and crisp white typography.
2. **KPI Metric Cards:** High-contrast summary cards with bold values (`font-size: 1.875rem; font-weight: 700;`). Includes dedicated inverted dark cards for liability tracking (*Doctor balances due*).
3. **Data Tables:** Clean rows with padding `12px 16px`, subtle border dividers, and distinct pill chips for visit types (`Home` in teal/green, `Clinic` in dark slate).
4. **Clinic P&L Chart Widget:** Lightweight inline SVG bar chart dynamically generated via Blade/JavaScript without requiring heavyweight chart libraries.

---

## 4. Complete Application Route Architecture

### 4.1 Public Website Routes (`routes/web.php` - Public Group)
```text
GET  /                             HomeController@index          Public landing & services showcase
GET  /services                     ServiceController@index       All clinic services
GET  /services/{slug}              ServiceController@show        Detailed treatment info (Physio/Hijama)
GET  /doctors                      DoctorDirectoryController     Approved public doctor profiles
GET  /book                         AppointmentController@create  Public booking form (Clinic & Home)
POST /book                         AppointmentController@store   Validate & store booking request
GET  /book/confirmation/{ref}      AppointmentController@confirm Booking success & tracking code
GET  /join-doctor                  DoctorSignupController@create Doctor onboarding & subscription info
POST /join-doctor                  DoctorSignupController@store  Submit doctor application & credentials
GET  /login                        AuthController@showLogin      Portal login form
POST /login                        AuthController@login          Native session authentication
POST /logout                       AuthController@logout         Invalidate session & redirect
```

### 4.2 Doctor Portal Routes (`routes/web.php` - Doctor Group, Middleware: `auth`, `role:doctor`)
```text
GET  /doctor/dashboard             DoctorDashboardController     Doctor metrics & alert banners
GET  /doctor/subscription          SubscriptionController@index  Subscription status & payment instructions
POST /doctor/subscription/proof    SubscriptionController@upload Submit monthly bank payment slip
GET  /doctor/board                 AppointmentBoardController    Open appointments (Blind voting board)
POST /doctor/board/{id}/interest   AppointmentBoardController    Express interest (Blind vote)
POST /doctor/board/{id}/pass       AppointmentBoardController    Decline / pass appointment
GET  /doctor/treatments            TreatmentController@index     Assigned patient appointments
GET  /doctor/treatments/{id}       TreatmentController@show      Patient details & home address
POST /doctor/treatments/{id}/notes TreatmentController@addNote   Log clinical session progress note
POST /doctor/treatments/{id}/done  TreatmentController@complete  Mark treatment as completed
GET  /doctor/earnings              EarningsController@index      Earnings breakdown & payout history
```

### 4.3 Admin Portal Routes (`routes/web.php` - Admin Group, Middleware: `auth`, `role:admin`)
```text
GET  /admin/dashboard              AdminDashboardController      Main operational dashboard (UI match)
GET  /admin/triage                 TriageController@index        Incoming appointment triage queue
POST /admin/triage/{id}/open-vote  TriageController@openVoting   Release appointment to doctor board
GET  /admin/assignment             AssignmentController@index    Appointments ready for doctor assignment
GET  /admin/assignment/{id}        AssignmentController@show     View interested doctors & assign
POST /admin/assignment/{id}/assign AssignmentController@assign   Assign doctor & generate billing record
GET  /admin/approvals              DoctorApprovalController      Pending doctor credentials review
POST /admin/approvals/{id}/action  DoctorApprovalController      Approve or reject doctor account
GET  /admin/subscriptions          SubscriptionAdminController   Verify doctor subscription slips
POST /admin/subscriptions/{id}/act SubscriptionAdminController   Activate or reject subscription
GET  /admin/billing                BillingController@index       Patient billing & outstanding balances
POST /admin/billing/{id}/pay       BillingController@recordPay   Record patient payment in PKR
GET  /admin/payouts                PayoutController@index        Doctor earnings ledger & payout actions
POST /admin/payouts/{id}/disburse  PayoutController@disburse     Disburse payout & log transaction ref
GET  /admin/pnl                    PnLController@index           Clinic P&L ledger & expense tracking
POST /admin/pnl/expense            PnLController@addExpense      Log operational clinic expense
GET  /admin/staff                  StaffController@index         Staff access & permission management
GET  /admin/settings               SettingsController@index      Clinic configurations & pricing
POST /admin/settings               SettingsController@update     Update settings
```

---

## 5. Deployment Architecture (StackCP Shared Hosting)

1. **Local Build & Packaging:**
   ```bash
   # Run Vite build locally on development PC
   npm run build
   # Verify compiled assets in public/build/
   ```
2. **Production Filesystem Setup:**
   * `/home/user/public_html/` $\to$ Symlinked to or configured as the Laravel `public/` directory.
   * Core Laravel files (`app/`, `bootstrap/`, `config/`, `database/`, `resources/`, `routes/`, `storage/`, `vendor/`) reside safely **one level above** the web root.
3. **Permissions Standard:**
   ```bash
   chmod -R 755 .
   chmod -R 775 storage bootstrap/cache
   ```
4. **Apache VirtualHost / `.htaccess` Directive:**
   Ensure `public/.htaccess` routes all traffic to `index.php` and blocks access to `.env`, `.git`, and private directories.
