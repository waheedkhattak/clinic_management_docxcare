# Docx_Care — Security & Privacy Specification

**Product:** Docx_Care (DOCX CLUB SMC-PVT Ltd.)  
**Standards:** Defense-in-Depth, Least Privilege, Zero Data Leakage  
**Platform Target:** StackCP Linux Shared Hosting with MariaDB 10.11.x  

---

## 1. Native Authentication Hardening
- **Framework Native Session:** Authenticates exclusively via Laravel's native session engine (`Auth::attempt()`). No third-party packages with bloated attack surfaces.
- **Session Fixation Defense:** Immediately regenerate session identifier upon successful login:
  ```php
  $request->session()->regenerate();
  ```
- **Cookie Security Configuration (`config/session.php`):**
  - `secure => env('APP_ENV') === 'production'` (Transmitted only over HTTPS).
  - `http_only => true` (Inaccessible to browser JavaScript/XSS).
  - `same_site => 'lax'` (Protection against Cross-Site Request Forgery).
- **Brute-Force Rate Limiting:**
  - Login endpoint protected via `RateLimiter::attempt()`: Maximum 5 failed attempts per minute per IP + email combination before a 60-second cooldown is enforced.

---

## 2. Role-Based Access Control (RBAC) & Middleware Pipeline
Access must be denied by default. Middleware enforces boundaries server-side:

```text
Request ──> [Authenticate] ──> [EnsureRole:admin|doctor|staff] ──> [EnsureActiveSubscription] ──> Controller
```

| Portal Scope | Allowed Roles | Enforcement Mechanism | Failure Response |
| :--- | :--- | :--- | :--- |
| **`/admin/*`** | `admin` | `EnsureRole:admin` middleware | `403 Forbidden` |
| **`/doctor/*`** | `doctor` (Approved) | `EnsureRole:doctor` + `EnsureApprovedDoctor` | `403 Forbidden` / Redirect to `/doctor/onboarding` |
| **Doctor Board** | `doctor` (Subscribed) | `EnsureActiveSubscription` middleware | Redirect to `/doctor/subscription` |
| **Public Routes** | Anyone | Standard CSRF verification on state-changing requests | `419 Page Expired` |

---

## 3. Blind Voting Mathematical Privacy Architecture
The Blind Voting mechanism is a core operational requirement. Doctors must never be able to discover which peer doctors have bid on an appointment.

1. **Query-Level Data Isolation:**
   - The query used by doctors to load open appointments uses aggregate count subqueries only:
     ```php
     // Safe: Retrieves only numeric aggregate count
     $appointments = Appointment::where('status', 'voting_open')
         ->withCount(['votes as interested_count' => function($q) {
             $q->where('vote_type', 'interested');
         }])
         ->get();
     ```
   - **Prohibited Code:** Eager loading `votes.doctor` or returning raw `AppointmentVote` collections to doctor-facing Blade templates or JSON responses is strictly forbidden.
2. **Database Constraint Integrity:**
   - Composite unique index `uq_appointment_doctor_vote (appointment_id, doctor_id)` prevents a doctor from submitting multiple bids to artificially inflate interest counts.
3. **Audit Trail Isolation:**
   - Only administrative users with verified `role == 'admin'` have authorization to view the list of doctors who expressed interest.

---

## 4. Patient PII & Home Visit Privacy
- **Address Masking:**
  - Home visit appointments contain sensitive home addresses. On the general Doctor Appointment Board (`voting_open`), only the general city/area is displayed (*e.g., "DHA Phase 5, Lahore"*).
  - The complete street address, house number, and patient phone number remain masked until the Admin officially executes the assignment.
- **Scoped Record Authorization:**
  - In `/doctor/treatments/{id}`, the controller strictly validates that `appointment->assigned_doctor_id === auth()->id()`.

---

## 5. File Upload Security Standards
Doctors upload credential documents (PMDC/license, CNIC) and monthly subscription bank slips.
- **Strict MIME Type Whitelist:** `image/jpeg`, `image/png`, `application/pdf` exclusively. Extension spoofing is checked via PHP Fileinfo (`$file->getMimeType()`).
- **File Size Cap:** Strictly limited to 5,120 KB (5 MB).
- **Storage Sanitization:**
  - Uploaded files are renamed using cryptographically secure UUIDs:
    ```php
    $filename = Str::uuid() . '.' . $file->getClientOriginalExtension();
    $path = $file->storeAs('proofs/' . date('Y/m'), $filename, 'public');
    ```
- **Execution Prevention:** The `.htaccess` in the storage directory explicitly disables PHP script execution (`php_flag engine off` or `RemoveHandler .php`).

---

## 6. Financial Integrity & Transaction Safety
- **Atomic Operations:** All billing modifications, doctor disbursements, and triage status updates must execute inside a database transaction:
  ```php
  DB::transaction(function () use (...) {
      // 1. Lock appointment row
      $appointment = Appointment::lockForUpdate()->findOrFail($id);
      // 2. Perform state transition
      // 3. Post double-entry to clinic_finances
  });
  ```
- **Pessimistic Locking:** `lockForUpdate()` is utilized during doctor assignment to prevent race conditions when two administrators attempt simultaneous assignments.

---

## 7. StackCP Shared Hosting Security Baseline
- **Document Root Isolation:** Web server document root must point strictly to Laravel's `/public` folder. The parent directory containing `.env`, `composer.json`, and source code must never be publicly accessible.
- **Environment Settings:**
  - `APP_ENV=production`
  - `APP_DEBUG=false` (Stack traces and SQL strings are never exposed to browser).
- **Database Account Principle of Least Privilege:**
  - The MariaDB user configured in `.env` should possess permissions only for `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `ALTER`, `INDEX`, `DROP` on the `docxcare` database.
