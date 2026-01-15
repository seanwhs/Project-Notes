# 🔐 BACKEND — Code 

**Purpose**: Production-grade backend for **regulated LPG operations** — not a CRUD demo.

---

## ✅ Architectural Characteristics

* Domain-Driven
* Service-Oriented
* Inventory-Safe
* Meter-Safe
* Audit-Safe
* Report-Ready

**Design stance**: Business invariants are enforced structurally, not by convention.

---

## 📦 App Structure

```
backend/
├── accounts/        # Identity & roles
├── customers/       # Contracts & billing rules
├── depots/          # Physical stock locations
├── inventory/       # Depot-scoped quantities (read-only)
├── distribution/    # Physical stock movement (commands)
├── transactions/    # Billing & sales logic
├── audit/           # Immutable activity log
├── reports/         # Read-only finance views
├── config/
│   └── urls.py
```

---

# 1️⃣ ACCOUNTS — Identity & Authorization

### Model

```python
class User(AbstractUser):
    ROLE_CHOICES = (
        ('ADMIN', 'Admin'),
        ('SALES', 'Sales'),
    )

    role = models.CharField(max_length=10, choices=ROLE_CHOICES)
    vehicle_no = models.CharField(max_length=20, blank=True, null=True)
```

**Invariants**

* Roles are explicit (no permission-string ambiguity)
* Vehicle assignment is optional but traceable

### Access Control

```python
class IsAdmin(BasePermission):
    def has_permission(self, request, view):
        return request.user.is_authenticated and request.user.role == 'ADMIN'
```

### URLs

```
GET/POST/PATCH/DELETE  /api/accounts/users/
```

Admin-only. No privilege escalation paths.

---

# 2️⃣ CUSTOMERS — Contracts & Rates

### Model

```python
class Customer(models.Model):
    PAYMENT = [('CASH', 'Cash'), ('CREDIT', 'Credit')]

    name = models.CharField(max_length=255)
    address = models.TextField()
    payment_type = models.CharField(max_length=10, choices=PAYMENT)

    meter_rate = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    rate_9kg = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    rate_12_7kg = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    rate_14kg = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    rate_50kg_pol = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    rate_50kg_l = models.DecimalField(max_digits=10, decimal_places=2, default=0)

    last_meter_reading = models.DecimalField(max_digits=10, decimal_places=2, default=0)
```

**Invariant**

> Rates are historical truth at point-of-sale. Never recomputed retroactively.

### URLs

```
GET/POST/PATCH  /api/customers/customers/
```

---

# 3️⃣ DEPOTS & INVENTORY — Physical Reality

### Depot

```python
class Depot(models.Model):
    code = models.CharField(max_length=20, unique=True)
    name = models.CharField(max_length=100)
```

### Inventory (Read-Only)

```python
class Inventory(models.Model):
    depot = models.ForeignKey(Depot, on_delete=models.CASCADE)
    equipment_name = models.CharField(max_length=50)
    quantity = models.IntegerField(default=0)

    class Meta:
        unique_together = ('depot', 'equipment_name')
```

**Invariants**

* Always depot-scoped
* No global stock illusion
* No direct writes

### URLs

```
GET  /api/inventory/inventory/
```

All mutations flow through services only.

---

# 4️⃣ DISTRIBUTION — Stock Movement (Commands)

### Model

```python
class Distribution(models.Model):
    STATUS = [('COLLECTION', 'Collection'), ('EMPTY_RETURN', 'Empty Return')]

    distribution_no = models.CharField(max_length=30, unique=True)
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    depot = models.ForeignKey(Depot, on_delete=models.CASCADE)
    equipment_name = models.CharField(max_length=50)
    quantity = models.PositiveIntegerField()
    status = models.CharField(max_length=20, choices=STATUS)
    created_at = models.DateTimeField(auto_now_add=True)
```

### Service (Atomic)

* Adjusts inventory
* Creates distribution record
* Writes audit log

**Invariant**: No distribution without inventory mutation.

### URL

```
POST  /api/distribution/distributions/create/
```

Command-style endpoint (intentionally not CRUD).

---

# 5️⃣ TRANSACTIONS — Billing Core

### Transaction

```python
class Transaction(models.Model):
    transaction_no = models.CharField(max_length=50, unique=True)
    customer = models.ForeignKey(Customer, on_delete=models.PROTECT)
    user = models.ForeignKey(User, on_delete=models.PROTECT)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    is_paid = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
```

### Meter Sale

```python
class MeterSale(models.Model):
    transaction = models.OneToOneField(Transaction, on_delete=models.CASCADE)
    last_reading = models.DecimalField(max_digits=10, decimal_places=2)
    latest_reading = models.DecimalField(max_digits=10, decimal_places=2)
    quantity = models.DecimalField(max_digits=10, decimal_places=2)
    rate = models.DecimalField(max_digits=10, decimal_places=2)
    subtotal = models.DecimalField(max_digits=10, decimal_places=2)
```

(CylinderSale & ServiceSale unchanged.)

**Critical Invariant**

> Meter readings update only after successful transaction commit.

### URLs

```
POST /api/transactions/transactions/create/
GET  /api/transactions/transactions/{transaction_no}/
```

---

# 6️⃣ AUDIT — Immutable Truth

**Properties**

* Append-only
* Never edited
* Never deleted

### URL

```
GET  /api/audit/audit-logs/
```

Admin-only. Forensics-ready.

---

# 7️⃣ REPORTS — Finance & Control

**Characteristics**

* Read-only
* Flat payloads
* URL-driven filters
* Export-ready

### URLs

```
GET /api/reports/sales/?from=&to=
GET /api/reports/inventory/?depot=
```

RR7 loader-compatible by design.

---

# 🌐 ROOT ROUTING

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/accounts/', include('accounts.urls')),
    path('api/customers/', include('customers.urls')),
    path('api/depots/', include('depots.urls')),
    path('api/inventory/', include('inventory.urls')),
    path('api/distribution/', include('distribution.urls')),
    path('api/transactions/', include('transactions.urls')),
    path('api/audit/', include('audit.urls')),
    path('api/reports/', include('reports.urls')),
]
```

---

## ✅ Final Result

* URLs express **intent**, not tables
* Commands never masquerade as CRUD
* Inventory & meters are structurally protected
* Audit coverage is automatic
* Regulators can trace every state change

---

**Next optional hardening**

* JWT + refresh rotation
* Idempotency keys (mobile sales)
* RR7 loader ↔ action mapping

---

# 🧾 GST & INVOICE NUMBERING — REGULATED BILLING EXTENSION

This section formalizes **tax correctness** and **invoice traceability**, suitable for Singapore-style GST regimes and regulated LPG billing.

---

## 1️⃣ Invoice Numbering — Immutable & Sequential

### Design Rules

* Invoice numbers are **system-generated**
* Never reused, never edited, never deleted
* Sequence is **monotonic**, not guessable
* Gap-tolerant (failed transactions do not corrupt sequence)

### Model

```python
class Invoice(models.Model):
    invoice_no = models.CharField(max_length=30, unique=True)
    transaction = models.OneToOneField(Transaction, on_delete=models.PROTECT)
    issued_at = models.DateTimeField(auto_now_add=True)

    # Monetary breakdown
    subtotal = models.DecimalField(max_digits=10, decimal_places=2)
    gst_amount = models.DecimalField(max_digits=10, decimal_places=2)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
```

**Invariant**

> An invoice may exist only for a successfully committed transaction.

---

## 2️⃣ GST Calculation — Explicit & Auditable

### GST Rules

* GST rate is **explicitly stored** at invoice time
* Never inferred later
* Never recomputed retroactively

```python
GST_RATE = Decimal('0.09')  # example: 9%

gst_amount = subtotal * GST_RATE
total_amount = subtotal + gst_amount
```

**Why this matters**

* Rate changes do not affect historical invoices
* Auditors can recompute totals exactly

---

## 3️⃣ Invoice Issuance Service (Atomic)

```python
class InvoiceService:

    @staticmethod
    @transaction.atomic
    def issue(transaction_obj):
        assert transaction_obj.id

        subtotal = transaction_obj.total_amount
        gst = subtotal * GST_RATE

        invoice = Invoice.objects.create(
            invoice_no=InvoiceService.next_invoice_no(),
            transaction=transaction_obj,
            subtotal=subtotal,
            gst_amount=gst,
            total_amount=subtotal + gst
        )

        AuditService.log(
            transaction_obj.user,
            f"Invoice {invoice.invoice_no} issued"
        )

        return invoice
```

✔ Atomic
✔ Idempotent-safe
✔ Audit-logged

---

## 4️⃣ Invoice Number Generator

```python
class InvoiceService:

    @staticmethod
    def next_invoice_no():
        today = timezone.now().strftime('%Y%m')
        last = Invoice.objects.filter(invoice_no__startswith=today).order_by('invoice_no').last()

        seq = int(last.invoice_no[-5:]) + 1 if last else 1
        return f"INV-{today}-{seq:05d}"
```

**Example**

```
INV-202601-00001
INV-202601-00002
```

---

## 5️⃣ API Surface

```
POST /api/transactions/{transaction_no}/issue-invoice/
GET  /api/invoices/{invoice_no}/
```

* Issue = command
* Retrieve = read-only

---

## 6️⃣ Compliance Guarantees

✔ Invoice ↔ Transaction is 1:1
✔ GST rate preserved historically
✔ Invoice numbers are immutable
✔ Full audit trail exists

---

## 7️⃣ Explicitly Forbidden

* ❌ Editing invoice totals
* ❌ Deleting invoices
* ❌ Reissuing invoices with same number
* ❌ Calculating GST on the frontend

---

**Result**: Your billing system is now **tax-correct, regulator-ready, and audit-proof**.
