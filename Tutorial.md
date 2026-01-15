# 🏗️ HSH Sales System — Full Stack Lab Tutorial 

**Objective:** Build a **production-grade LPG logistics & sales system** with **explicit contracts, audit safety, and regulator-ready documentation**.

**Stack:**

* **Backend:** Django 4.2 · Django REST Framework · MySQL · drf-yasg (Swagger)
* **Frontend:** React 18 · React Router v7 (Data APIs)
* **Infra:** Docker · Docker Compose
* **Design:** Domain-driven · inventory-safe · audit-safe · report-ready

---

## 0️⃣ Prerequisites

Install:

* Docker + Docker Compose
* Python 3.11+
* Node.js 20+
* VS Code (recommended)

---

## 1️⃣ Project Structure

```bash
hsh_sales_system/
├── backend/
│   ├── config/
│   ├── accounts/
│   ├── customers/
│   ├── depots/
│   ├── inventory/
│   ├── distribution/
│   ├── transactions/
│   ├── audit/
│   └── reports/
└── frontend/
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
pip install \
  django==4.2 \
  djangorestframework \
  djangorestframework-simplejwt \
  drf-yasg \
  mysqlclient
```

### 2.3 Create Django Project & Apps

```bash
django-admin startproject config .

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
    "drf_yasg",

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
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
    "DEFAULT_PERMISSION_CLASSES": [
        "rest_framework.permissions.IsAuthenticated",
    ],
}
```

> 🚨 `AUTH_USER_MODEL` **must be set before first migration**.

---

## 4️⃣ Database — Dockerized MySQL

### `docker-compose.yml`

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

```python
class User(AbstractUser):
    ROLE_CHOICES = [
        ("ADMIN", "Admin"),
        ("SALES", "Sales"),
    ]
    role = models.CharField(max_length=10, choices=ROLE_CHOICES)
    vehicle_no = models.CharField(max_length=20, null=True, blank=True)
```

Roles are enforced **at service & report boundaries**, not in serializers.

---

## 6️⃣ Customers — Contracts & Pricing

```python
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

✔ Rates are **snapshotted at transaction time**.

---

## 7️⃣ Depots & Inventory (Read-Only API)

```python
class Inventory(models.Model):
    depot = models.ForeignKey(Depot, on_delete=models.CASCADE)
    equipment = models.CharField(max_length=50)
    quantity = models.IntegerField(default=0)

    class Meta:
        unique_together = ("depot", "equipment")
```

❗ Inventory **cannot** be mutated via serializers or views.

---

## 8️⃣ Distribution — Atomic Stock Movement

```python
class Distribution(models.Model):
    TYPE = [("COLLECTION", "Collection"), ("EMPTY_RETURN", "Empty Return")]

    number = models.CharField(max_length=30, unique=True)
    user = models.ForeignKey(User, on_delete=models.PROTECT)
    depot = models.ForeignKey(Depot, on_delete=models.PROTECT)
    equipment = models.CharField(max_length=50)
    quantity = models.PositiveIntegerField()
    type = models.CharField(max_length=20, choices=TYPE)
    created_at = models.DateTimeField(auto_now_add=True)
```

```python
class DistributionService:
    @staticmethod
    @transaction.atomic
    def execute(user, depot, equipment, qty, movement):
        stock = Inventory.objects.select_for_update().get(
            depot=depot, equipment=equipment
        )
        if movement == "COLLECTION" and stock.quantity < qty:
            raise ValueError("Insufficient stock")

        stock.quantity += qty if movement == "EMPTY_RETURN" else -qty
        stock.save()
        AuditService.log(user, f"{movement}: {equipment} x{qty}")
```

✔ Race-safe · rollback-safe · audited

---

## 9️⃣ Transactions — Billing Core

Supports:

* Meter billing
* Cylinder item sales
* Service item sales

```python
class Transaction(models.Model):
    number = models.CharField(max_length=50, unique=True)
    customer = models.ForeignKey(Customer, on_delete=models.PROTECT)
    user = models.ForeignKey(User, on_delete=models.PROTECT)
    total = models.DecimalField(max_digits=12, decimal_places=2)
    paid = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
```

---

## 🔟 Audit — Immutable Ledger

```python
class AuditLog(models.Model):
    user = models.ForeignKey(User, on_delete=models.PROTECT)
    action = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
```

Append-only. No updates. No deletes.

---

## 1️⃣1️⃣ Reports — Read Models

```python
@api_view(["GET"])
@permission_classes([IsAdmin])
def summary(request):
    return Response(Transaction.objects.aggregate(total=Sum("total")))
```

---

## 1️⃣2️⃣ Swagger / OpenAPI (drf-yasg)

### `config/urls.py`

```python
from drf_yasg.views import get_schema_view
from drf_yasg import openapi

schema_view = get_schema_view(
    openapi.Info(
        title="HSH Sales API",
        default_version="v1",
        description="LPG Sales & Inventory System",
    ),
    public=True,
)

urlpatterns = [
    path("swagger/", schema_view.with_ui("swagger")),
    path("redoc/", schema_view.with_ui("redoc")),
]
```

✔ `/swagger/` — interactive dev UI
✔ `/redoc/` — auditor & regulator-friendly

Swagger **documents contracts only** — services still enforce rules.

---

## 1️⃣3️⃣ Migrate & Run

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

---

## 1️⃣4️⃣ Frontend — React + RR7

* RR7 loaders/actions map **1:1 to documented APIs**
* Offline-first capability
* No hidden endpoints — Swagger is source of truth

---

## 🧠 System Guarantees

* ❌ No negative inventory
* ❌ No silent stock mutation
* ❌ No rate recalculation
* ❌ No audit deletion
* ✔ Deterministic billing
* ✔ Regulator-ready documentation

---

**This document is a lab manual, not a demo.**
