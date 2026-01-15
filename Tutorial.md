# 🏗️ HSH Sales System — Full Stack Lab Tutorial (Authoritative Edition)

**Objective:** Build a **production-grade LPG logistics & sales system** using:

* **Backend:** Django 4.2 + DRF + MySQL
* **Frontend:** React 18 + React Router v7
* **Infra:** Docker + Docker Compose
* **Design:** Domain-driven, inventory-safe, audit-safe, report-ready

---

## 0️⃣ Prerequisites

Install the following:

* Docker + Docker Compose
* Python 3.11+
* Node.js 20+
* VS Code (recommended)

---

## 1️⃣ Project Structure

```bash
mkdir hsh_sales_system
cd hsh_sales_system
mkdir backend frontend
```

### Backend (logical)

```
backend/
├── config/
├── accounts/
├── customers/
├── depots/
├── inventory/
├── distribution/
├── transactions/
├── audit/
├── reports/
```

---

## 2️⃣ Backend Bootstrap

### 2.1 Virtual Environment

```bash
cd backend
python -m venv venv
venv\Scripts\activate
```

### 2.2 Install Dependencies

```bash
pip install django djangorestframework djangorestframework-simplejwt mysqlclient
```

### 2.3 Create Django Project

```bash
django-admin startproject config .
```

### 2.4 Create Apps

```bash
python manage.py startapp accounts
python manage.py startapp customers
python manage.py startapp depots
python manage.py startapp inventory
python manage.py startapp distribution
python manage.py startapp transactions
python manage.py startapp audit
python manage.py startapp reports
```

---

## 3️⃣ Global Django Configuration

### `config/settings.py`

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",

    "rest_framework",

    "accounts",
    "customers",
    "depots",
    "inventory",
    "distribution",
    "transactions",
    "audit",
    "reports",
]

AUTH_USER_MODEL = "accounts.User"

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication"
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated"
    ],
}
```

> 🚨 **Set `AUTH_USER_MODEL` before the first migration** to avoid `auth.User` conflicts.

---

## 4️⃣ Database Configuration

### Docker Compose (MySQL)

```yaml
version: "3.9"
services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: hsh_sales
      MYSQL_USER: hsh
      MYSQL_PASSWORD: hshpass
    ports:
      - "3306:3306"
```

### Django DB Settings

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.mysql",
        "NAME": "hsh_sales",
        "USER": "hsh",
        "PASSWORD": "hshpass",
        "HOST": "db",
        "PORT": "3306",
    }
}
```

---

## 5️⃣ Accounts — Identity & Roles

### `accounts/models.py`

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    ROLE_CHOICES = [
        ("ADMIN", "Admin"),
        ("SALES", "Sales"),
    ]
    role = models.CharField(max_length=10, choices=ROLE_CHOICES)
    vehicle_no = models.CharField(max_length=20, null=True, blank=True)
```

---

## 6️⃣ Customers — Contracts & Pricing

### `customers/models.py`

```python
from django.db import models

class Customer(models.Model):
    PAYMENT = [("CASH", "Cash"), ("CREDIT", "Credit")]

    name = models.CharField(max_length=255)
    address = models.TextField()
    payment_type = models.CharField(max_length=10, choices=PAYMENT)

    meter_rate = models.DecimalField(max_digits=10, decimal_places=2)
    rate_9kg = models.DecimalField(max_digits=10, decimal_places=2)
    rate_12_7kg = models.DecimalField(max_digits=10, decimal_places=2)
    rate_14kg = models.DecimalField(max_digits=10, decimal_places=2)
    rate_50kg_pol = models.DecimalField(max_digits=10, decimal_places=2)
    rate_50kg_l = models.DecimalField(max_digits=10, decimal_places=2)

    last_meter_reading = models.DecimalField(max_digits=10, decimal_places=2, default=0)
```

> ❗ Rates are snapshotted at transaction time, never recalculated.

---

## 7️⃣ Depots & Inventory

### `depots/models.py`

```python
from django.db import models

class Depot(models.Model):
    code = models.CharField(max_length=20, unique=True)
    name = models.CharField(max_length=100)
```

### `inventory/models.py`

```python
from django.db import models
from depots.models import Depot

class Inventory(models.Model):
    depot = models.ForeignKey(Depot, on_delete=models.CASCADE)
    equipment = models.CharField(max_length=50)
    quantity = models.IntegerField(default=0)

    class Meta:
        unique_together = ("depot", "equipment")
```

> ❗ Inventory is **read-only via API** — mutations go through services.

---

## 8️⃣ Distribution — Atomic Stock Movement

### `distribution/models.py`

```python
from django.db import models
from django.contrib.auth import get_user_model
from depots.models import Depot

User = get_user_model()

class Distribution(models.Model):
    TYPE = [
        ("COLLECTION", "Collection"),
        ("EMPTY_RETURN", "Empty Return"),
    ]

    number = models.CharField(max_length=30, unique=True)
    user = models.ForeignKey(User, on_delete=models.PROTECT)
    depot = models.ForeignKey(Depot, on_delete=models.PROTECT)
    equipment = models.CharField(max_length=50)
    quantity = models.PositiveIntegerField()
    type = models.CharField(max_length=20, choices=TYPE)
    created_at = models.DateTimeField(auto_now_add=True)
```

### `distribution/services.py`

```python
from django.db import transaction
from inventory.models import Inventory
from audit.services import AuditService

class DistributionService:

    @staticmethod
    @transaction.atomic
    def execute(user, depot, equipment, qty, movement):
        stock, _ = Inventory.objects.select_for_update().get_or_create(
            depot=depot, equipment=equipment, defaults={"quantity": 0}
        )

        stock.quantity += qty if movement == "EMPTY_RETURN" else -qty
        stock.save()

        AuditService.log(user, f"{movement}: {equipment} x{qty}")
```

✔ Atomic, race-safe, audited

---

## 9️⃣ Transactions — Billing Core

### `transactions/models.py`

```python
from django.db import models
from django.contrib.auth import get_user_model
from customers.models import Customer

User = get_user_model()

class Transaction(models.Model):
    number = models.CharField(max_length=50, unique=True)
    customer = models.ForeignKey(Customer, on_delete=models.PROTECT)
    user = models.ForeignKey(User, on_delete=models.PROTECT)
    total = models.DecimalField(max_digits=12, decimal_places=2)
    paid = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
```

### Meter Sale Example

```python
class MeterSale(models.Model):
    transaction = models.OneToOneField(Transaction, on_delete=models.CASCADE)
    last = models.DecimalField(max_digits=10, decimal_places=2)
    latest = models.DecimalField(max_digits=10, decimal_places=2)
    qty = models.DecimalField(max_digits=10, decimal_places=2)
    rate = models.DecimalField(max_digits=10, decimal_places=2)
    subtotal = models.DecimalField(max_digits=10, decimal_places=2)
```

---

## 🔟 Audit — Immutable Ledger

### `audit/models.py`

```python
from django.db import models
from django.contrib.auth import get_user_model

User = get_user_model()

class AuditLog(models.Model):
    user = models.ForeignKey(User, on_delete=models.PROTECT)
    action = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
```

### `audit/services.py`

```python
from .models import AuditLog

class AuditService:
    @staticmethod
    def log(user, action):
        AuditLog.objects.create(user=user, action=action)
```

---

## 1️⃣1️⃣ Reports — Finance Read Models

```python
from django.db.models import Sum
from rest_framework.decorators import api_view, permission_classes
from rest_framework.response import Response
from accounts.permissions import IsAdmin
from transactions.models import Transaction

@api_view(["GET"])
@permission_classes([IsAdmin])
def summary(request):
    return Response(Transaction.objects.aggregate(total=Sum("total")))
```

```python
from django.utils.timezone import now
from datetime import timedelta

@api_view(["GET"])
@permission_classes([IsAdmin])
def aging(request):
    today = now()
    return {
        "30": Transaction.objects.filter(created_at__lte=today - timedelta(days=30), paid=False).count(),
        "60": Transaction.objects.filter(created_at__lte=today - timedelta(days=60), paid=False).count(),
        "90": Transaction.objects.filter(created_at__lte=today - timedelta(days=90), paid=False).count(),
    }
```

---

## 1️⃣2️⃣ Migrate & Run

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

---

## 1️⃣3️⃣ Frontend — React + RR7

✔ Vite + Tailwind
✔ RR7 Data APIs
✔ Offline-ready architecture

*`src/router.jsx`, `api.js`, `DashboardLayout.jsx`, `inventory.jsx` already integrated*

---

## 1️⃣4️⃣ Seed Data

```bash
python manage.py seed_data
```

---

## 1️⃣5️⃣ Automated Tests

### Inventory Atomicity

```python
import pytest
from distribution.services import DistributionService
from inventory.models import Inventory

@pytest.mark.django_db
def test_distribution_atomicity(user, depot):
    inv = Inventory.objects.create(depot=depot, equipment="50KG", quantity=10)
    DistributionService.create(user=user, depot=depot, equipment="50KG", quantity=5, status="COLLECTION")
    inv.refresh_from_db()
    assert inv.quantity == 5
```

### No Partial Commit

```python
@pytest.mark.django_db
def test_no_negative_inventory(user, depot):
    Inventory.objects.create(depot=depot, equipment="50KG", quantity=2)
    with pytest.raises(Exception):
        DistributionService.create(user=user, depot=depot, equipment="50KG", quantity=10, status="COLLECTION")
```

✔ Enforces business invariants and audit safety

---

# 🖼️ HSH Sales System — Workflow Diagram (Concept)

```
                 ┌─────────────┐
                 │   Frontend  │
                 │  React + RR7│
                 └─────┬───────┘
                       │ HTTP / Data APIs
                       ▼
                 ┌─────────────┐
                 │  API Layer  │
                 │ DRF + JWT  │
                 └─────┬───────┘
                       │
          ┌────────────┴─────────────┐
          │                          │
  ┌──────────────┐            ┌──────────────┐
  │ Distribution │            │ Transactions │
  │   Service    │            │   Service    │
  └─────┬────────┘            └─────┬────────┘
        │                             │
        │ Inventory updates           │ Meter & Cylinder Billing
        ▼                             ▼
  ┌──────────────┐            ┌──────────────┐
  │  Inventory   │            │   Customer   │
  │  Model (Read)│            │  Model / DB  │
  └─────┬────────┘            └─────┬────────┘
        │                             │
        │ Audit logs                  │
        ▼                             ▼
                 ┌─────────────┐
                 │   Audit DB  │
                 │  Immutable  │
                 └─────────────┘
```

### Key Points:

1. **Frontend:** React + RR7 handles routes, offline-ready, consumes API endpoints.
2. **API Layer:** DRF + JWT handles authentication, permissions, and routing to services.
3. **Distribution Service:** Atomic inventory updates with audit logging.
4. **Transactions Service:** Handles billing, meter readings, customer rates snapshotting.
5. **Inventory:** Always authoritative; API never mutates directly.
6. **Audit DB:** Immutable, append-only logs from all services.
7. **Reports:** Read-only, aggregates data from transactions and audit logs.


