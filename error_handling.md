# Docx_Care — Error Handling & Validation Standards

**Product:** Docx_Care (DOCX CLUB SMC-PVT Ltd.)  
**Integration:** Blade Templates with Vanilla CSS (Theme-aware Light/Dark Error Tokens)  

---

## 1. Core Principles
1. **Zero Information Leakage:** Production users must never see stack traces, SQL error codes, internal server paths, or raw database messages.
2. **Actionable User Guidance:** Validation feedback must highlight specific fields and state clear requirements.
3. **Atomic Rollbacks:** Any operational failure during financial transactions or doctor assignment must completely roll back the database state.
4. **Resilient Logging:** Detailed diagnostic information (exception class, timestamp, actor ID, correlation ID) is recorded to secure local logs.

---

## 2. HTTP Status Code Mapping Matrix

| Status | Category | System Behavior | User Presentation |
| :---: | :--- | :--- | :--- |
| **302** | Validation / Flash | Redirects back with input (`withInput()`) and flash banner. | High-contrast alert banner + red border on invalid input fields. |
| **401** | Unauthenticated | Redirects to `/login` with `intended()` URL saved. | Redirect to login with alert: *"Please log in to continue."* |
| **403** | Unauthorized Role | Denies access at middleware boundary; logs security event. | Custom `errors/403.blade.php`: *"Access denied: You lack permissions for this portal."* |
| **404** | Resource Not Found | Catches `ModelNotFoundException` silently. | Custom `errors/404.blade.php`: *"The requested record was not found."* |
| **409** | Workflow Conflict | Catches concurrent assignment or duplicate vote. | Toast alert: *"This appointment was already updated by another user. Please refresh."* |
| **419** | Session / CSRF Expired | Catches `TokenMismatchException`. | Custom `errors/419.blade.php`: *"Your session expired. Please refresh the page and try again."* |
| **429** | Rate Limited | Throttles abuse on login and public booking. | Toast alert: *"Too many requests. Please wait 60 seconds before trying again."* |
| **500** | Fatal Server Error | Catches unhandled exceptions, scrubs output, logs stack trace. | Custom `errors/500.blade.php`: *"An unexpected error occurred. Please contact clinic support."* |

---

## 3. UI Alert Components (Vanilla CSS for Light & Dark Mode)

### Blade Component Markup (`resources/views/components/alert.blade.php`)
```html
@if(session('success'))
<div class="alert alert-success" role="alert">
    <svg class="alert-icon" viewBox="0 0 20 20" fill="currentColor">
        <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clip-rule="evenodd"/>
    </svg>
    <div class="alert-content">{{ session('success') }}</div>
</div>
@endif

@if(session('error') || $errors->any())
<div class="alert alert-error" role="alert">
    <svg class="alert-icon" viewBox="0 0 20 20" fill="currentColor">
        <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.707 7.293a1 1 0 00-1.414 1.414L8.586 10l-1.293 1.293a1 1 0 101.414 1.414L10 11.414l1.293 1.293a1 1 0 001.414-1.414L11.414 10l1.293-1.293a1 1 0 00-1.414-1.414L10 8.586 8.707 7.293z" clip-rule="evenodd"/>
    </svg>
    <div class="alert-content">
        @if(session('error'))
            <div>{{ session('error') }}</div>
        @else
            <ul class="alert-list">
                @foreach($errors->all() as $error)
                    <li>{{ $error }}</li>
                @endforeach
            </ul>
        @endif
    </div>
</div>
@endif
```

### CSS Token Styling (`resources/css/app.css`)
```css
.alert {
    display: flex;
    align-items: flex-start;
    gap: 0.75rem;
    padding: 0.875rem 1.25rem;
    border-radius: var(--radius-md);
    margin-bottom: 1.25rem;
    font-size: 0.9375rem;
    border: 1px solid transparent;
}
.alert-icon {
    width: 1.25rem;
    height: 1.25rem;
    flex-shrink: 0;
    margin-top: 0.125rem;
}
.alert-success {
    background-color: #f0fdf4;
    color: #166534;
    border-color: #bbf7d0;
}
.alert-error {
    background-color: #fef2f2;
    color: #991b1b;
    border-color: #fecaca;
}
[data-theme="dark"] .alert-success {
    background-color: rgba(22, 163, 74, 0.15);
    color: #86efac;
    border-color: rgba(34, 197, 94, 0.3);
}
[data-theme="dark"] .alert-error {
    background-color: rgba(239, 68, 68, 0.15);
    color: #fca5a5;
    border-color: rgba(239, 68, 68, 0.3);
}
.alert-list {
    margin: 0;
    padding-left: 1.25rem;
}
```

---

## 4. Custom Domain Exceptions

```php
namespace App\Exceptions;

class AppointmentConflictException extends \DomainException {}
class BlindVotingException extends \DomainException {}
class SubscriptionExpiredException extends \DomainException {}
```

### Transaction Handling Pattern
```php
try {
    DB::transaction(function () use ($appointmentId, $doctorId) {
        $appointment = Appointment::lockForUpdate()->findOrFail($appointmentId);

        if (!in_array($appointment->status, ['voting_open', 'triage_pending'])) {
            throw new AppointmentConflictException('This appointment is no longer available for assignment.');
        }

        $appointment->update([
            'assigned_doctor_id' => $doctorId,
            'status' => 'assigned'
        ]);

        PatientBilling::create([
            'appointment_id' => $appointment->id,
            'invoice_number' => 'INV-' . date('Y') . '-' . str_pad($appointment->id, 4, '0', STR_PAD_LEFT),
            'total_amount' => 3000.00,
            'paid_amount' => 0.00,
            'balance_amount' => 3000.00,
            'status' => 'unpaid'
        ]);
    });

    return redirect()->route('admin.assignment.index')->with('success', 'Doctor successfully assigned.');
} catch (AppointmentConflictException $e) {
    return back()->with('error', $e->getMessage());
} catch (\Throwable $e) {
    Log::error('Assignment Failure', ['error' => $e->getMessage(), 'appointment_id' => $appointmentId]);
    return back()->with('error', 'A system error occurred while processing the assignment. Please try again.');
}
```

---

## 5. Production Safe Logging Standards
- **PII Scrubbing:** Never write unencrypted CNICs, bank account numbers, passwords, or complete patient home addresses to `storage/logs/laravel.log`.
- **Structured Context:**
  ```php
  Log::channel('daily')->info('Patient Payment Recorded', [
      'invoice_id' => $invoice->id,
      'amount_pkr' => $amount,
      'method' => $method,
      'admin_user_id' => auth()->id(),
      'ip' => request()->ip()
  ]);
  ```
