# Docx_Care — Database Architecture & DDL Schema

**Engine:** MariaDB 10.11.x (Default Engine: `InnoDB`, Charset: `utf8mb4`, Collation: `utf8mb4_unicode_ci`)  
**ORM Standard:** Laravel Eloquent (Strict types, mass-assignment guards)  
**Currency Standard:** Pakistani Rupee (`PKR`) stored as `DECIMAL(12,2)`  
**Time Standard:** Stored in `UTC`; presented in `Asia/Karachi`  

---

## 1. Schema Design Principles
1. **Strict Relational Integrity:** Enforce foreign keys with explicit delete actions (`CASCADE`, `RESTRICT`, or `SET NULL`).
2. **Race-Condition Prevention:** Composite unique constraints prevent duplicate votes (`uq_appointment_doctor_vote`) and duplicate profile links.
3. **Optimized Indexes:** Composite indexes on frequently queried operational filters (`(role, status)`, `(visit_type, preferred_date)`, `(entry_date, type)`).
4. **Auditability:** Core administrative actions and financial modifications are tracked in `audit_logs` and `clinic_finances`.

---

## 2. MariaDB DDL Specifications

```sql
-- 1. Users (Core Authentication & System Roles)
CREATE TABLE users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    phone VARCHAR(50) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    role ENUM('admin', 'staff', 'doctor') NOT NULL DEFAULT 'doctor',
    status ENUM('active', 'pending', 'suspended') NOT NULL DEFAULT 'pending',
    email_verified_at TIMESTAMP NULL,
    remember_token VARCHAR(100) NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    INDEX idx_user_role_status (role, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 2. Doctor Profiles (Clinical Verification, Capabilities & Bank Details)
CREATE TABLE doctor_profiles (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL UNIQUE,
    specialty VARCHAR(150) NOT NULL,
    license_number VARCHAR(100) NOT NULL UNIQUE, -- PMDC / Healthcare Commission Reg No.
    cnic VARCHAR(50) NOT NULL UNIQUE,            -- National Identity Card Number
    bio TEXT NULL,
    photo_path VARCHAR(255) NULL,
    visit_capabilities ENUM('clinic', 'home', 'both') NOT NULL DEFAULT 'both',
    bank_name VARCHAR(100) NULL,
    account_number VARCHAR(100) NULL,
    account_title VARCHAR(150) NULL,
    commission_rate DECIMAL(5,2) NOT NULL DEFAULT 70.00, -- Doctor share percentage (e.g., 70%)
    is_approved BOOLEAN NOT NULL DEFAULT FALSE,
    approved_by BIGINT UNSIGNED NULL,
    approved_at TIMESTAMP NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (approved_by) REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 3. Subscriptions (Doctor Membership Lifecycle & Payment Proof Verification)
CREATE TABLE subscriptions (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    doctor_id BIGINT UNSIGNED NOT NULL,
    plan_name VARCHAR(100) NOT NULL DEFAULT 'Monthly Doctor Membership',
    amount DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    proof_image_path VARCHAR(255) NOT NULL,
    status ENUM('pending', 'active', 'rejected', 'expired') NOT NULL DEFAULT 'pending',
    start_date DATE NULL,
    end_date DATE NULL,
    verified_by BIGINT UNSIGNED NULL,
    verified_at TIMESTAMP NULL,
    admin_notes VARCHAR(255) NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (doctor_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (verified_by) REFERENCES users(id) ON DELETE SET NULL,
    INDEX idx_subscription_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 4. Services (Treatment Catalog)
CREATE TABLE services (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(150) NOT NULL,
    slug VARCHAR(150) NOT NULL UNIQUE,
    category VARCHAR(100) NOT NULL, -- Physiotherapy, Hijama, Psychology, Acupuncture, etc.
    description TEXT NULL,
    duration_minutes INT UNSIGNED NOT NULL DEFAULT 45,
    base_price DECIMAL(10,2) NOT NULL DEFAULT 0.00,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    INDEX idx_service_category_active (category, is_active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 5. Appointments (Booking Engine & Clinical Workflow)
CREATE TABLE appointments (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    reference_no VARCHAR(50) NOT NULL UNIQUE, -- Standard: REQ-YYYY-XXXX (e.g., REQ-2026-0041)
    patient_name VARCHAR(255) NOT NULL,
    patient_phone VARCHAR(50) NOT NULL,
    patient_email VARCHAR(255) NULL,
    patient_address TEXT NULL,                -- Critical for Home Visits
    category VARCHAR(100) NOT NULL,           -- Physiotherapy, Hijama, etc.
    visit_type ENUM('clinic', 'home') NOT NULL DEFAULT 'clinic',
    preferred_date DATE NOT NULL,
    preferred_time VARCHAR(50) NULL,
    clinical_notes TEXT NULL,
    assigned_doctor_id BIGINT UNSIGNED NULL,
    status ENUM(
        'triage_pending',  -- Newly submitted request
        'voting_open',     -- Open on Doctor board for interest
        'assigned',        -- Doctor assigned by Admin
        'in_progress',     -- Treatment in progress
        'completed',       -- Treatment sessions finished
        'cancelled'        -- Cancelled
    ) NOT NULL DEFAULT 'triage_pending',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (assigned_doctor_id) REFERENCES users(id) ON DELETE SET NULL,
    INDEX idx_appointment_status (status),
    INDEX idx_appointment_visit_date (visit_type, preferred_date),
    INDEX idx_appointment_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 6. Appointment Votes (Blind Interest Voting Mechanism)
CREATE TABLE appointment_votes (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    appointment_id BIGINT UNSIGNED NOT NULL,
    doctor_id BIGINT UNSIGNED NOT NULL,
    vote_type ENUM('interested', 'unavailable') NOT NULL DEFAULT 'interested',
    remarks VARCHAR(255) NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    UNIQUE KEY uq_appointment_doctor_vote (appointment_id, doctor_id),
    FOREIGN KEY (appointment_id) REFERENCES appointments(id) ON DELETE CASCADE,
    FOREIGN KEY (doctor_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 7. Session Notes (Treatment Progress Tracking)
CREATE TABLE session_notes (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    appointment_id BIGINT UNSIGNED NOT NULL,
    doctor_id BIGINT UNSIGNED NOT NULL,
    session_number INT UNSIGNED NOT NULL DEFAULT 1,
    pain_scale_before TINYINT UNSIGNED NULL,  -- Scale 1-10
    pain_scale_after TINYINT UNSIGNED NULL,   -- Scale 1-10
    symptoms_observed TEXT NOT NULL,
    treatment_provided TEXT NOT NULL,
    patient_response TEXT NULL,
    next_session_date DATE NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (appointment_id) REFERENCES appointments(id) ON DELETE CASCADE,
    FOREIGN KEY (doctor_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 8. Patient Billings (Invoices & Balance Tracking)
CREATE TABLE patient_billings (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    appointment_id BIGINT UNSIGNED NOT NULL UNIQUE,
    invoice_number VARCHAR(50) NOT NULL UNIQUE, -- Standard: INV-YYYY-XXXX
    total_amount DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    paid_amount DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    balance_amount DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    status ENUM('unpaid', 'partial', 'paid') NOT NULL DEFAULT 'unpaid',
    payment_method VARCHAR(50) NULL,            -- Cash, Bank Transfer, EasyPaisa, JazzCash
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (appointment_id) REFERENCES appointments(id) ON DELETE CASCADE,
    INDEX idx_billing_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 9. Doctor Payouts (Compensation Ledger)
CREATE TABLE doctor_payouts (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    payout_number VARCHAR(50) NOT NULL UNIQUE,  -- Standard: PAY-YYYY-XXXX
    doctor_id BIGINT UNSIGNED NOT NULL,
    appointment_id BIGINT UNSIGNED NULL,
    gross_amount DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    clinic_commission DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    net_payout DECIMAL(12,2) NOT NULL DEFAULT 0.00,
    status ENUM('due', 'paid') NOT NULL DEFAULT 'due',
    payment_method VARCHAR(50) NULL,
    transaction_ref VARCHAR(100) NULL,
    paid_at TIMESTAMP NULL,
    processed_by BIGINT UNSIGNED NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (doctor_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (appointment_id) REFERENCES appointments(id) ON DELETE SET NULL,
    FOREIGN KEY (processed_by) REFERENCES users(id) ON DELETE SET NULL,
    INDEX idx_doctor_payout_status (doctor_id, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 10. Clinic Finances (P&L Ledger & Capital Reserve)
CREATE TABLE clinic_finances (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    type ENUM('revenue', 'doctor_payout', 'expense', 'capital') NOT NULL,
    category VARCHAR(100) NOT NULL, -- Rent, Utilities, Medical Supplies, Patient Fee, Subscription Fee, etc.
    amount DECIMAL(12,2) NOT NULL,
    description TEXT NULL,
    reference_id VARCHAR(100) NULL, -- Links to invoice_number or payout_number
    entry_date DATE NOT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    INDEX idx_pnl_date_type (entry_date, type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 11. Audit Logs (Enterprise Administrative Traceability)
CREATE TABLE audit_logs (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NULL,
    action VARCHAR(100) NOT NULL,     -- e.g. DOCTOR_APPROVED, APPOINTMENT_ASSIGNED, PAYMENT_RECEIVED
    target_type VARCHAR(100) NOT NULL,
    target_id BIGINT UNSIGNED NOT NULL,
    details JSON NULL,
    ip_address VARCHAR(45) NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL,
    INDEX idx_audit_target (target_type, target_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 12. Settings (Key-Value Clinic Operational Parameters)
CREATE TABLE settings (
    `key` VARCHAR(100) PRIMARY KEY,
    `value` TEXT NULL,
    `updated_at` TIMESTAMP NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 3. Eloquent Model Relationships

```php
// User.php
public function doctorProfile(): HasOne { return $this->hasOne(DoctorProfile::class); }
public function subscriptions(): HasMany { return $this->hasMany(Subscription::class, 'doctor_id'); }
public function activeSubscription(): HasOne {
    return $this->hasOne(Subscription::class, 'doctor_id')
        ->where('status', 'active')
        ->where('end_date', '>=', now()->toDateString());
}
public function assignedAppointments(): HasMany { return $this->hasMany(Appointment::class, 'assigned_doctor_id'); }
public function votes(): HasMany { return $this->hasMany(AppointmentVote::class, 'doctor_id'); }
public function payouts(): HasMany { return $this->hasMany(DoctorPayout::class, 'doctor_id'); }

// Appointment.php
public function assignedDoctor(): BelongsTo { return $this->belongsTo(User::class, 'assigned_doctor_id'); }
public function votes(): HasMany { return $this->hasMany(AppointmentVote::class); }
public function sessionNotes(): HasMany { return $this->hasMany(SessionNote::class); }
public function billing(): HasOne { return $this->hasOne(PatientBilling::class); }

// Scope for Doctor Board (Enforcing Blind Privacy):
public function scopeForDoctorBoard($query, $doctorId) {
    return $query->where('status', 'voting_open')
        ->withCount(['votes as total_interested' => function($q) {
            $q->where('vote_type', 'interested');
        }])
        ->withExists(['votes as has_voted' => function($q) use ($doctorId) {
            $q->where('doctor_id', $doctorId);
        }]);
}
```
