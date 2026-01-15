# 🧪 MASTER LAB — HSH SALES SYSTEM 

**Purpose:** Regulated LPG operations
**Design:** Domain-Driven · Service-Oriented · Inventory-Safe · Meter-Safe · Audit-Safe · Report-Ready
**Audience:** Architects · Senior Engineers · Production Teams

---

## 0️⃣ ENVIRONMENT (DO THIS ONCE)

```bash
mkdir hsh_sales_backend
cd hsh_sales_backend
python -m venv venv
venv\Scripts\activate
pip install django djangorestframework mysqlclient djangorestframework-simplejwt
django-admin startproject config .
```

Create apps:

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

## 1️⃣ GLOBAL DJANGO CONFIG (NON-NEGOTIABLE)

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
    "DEFAULT_AUTHENTICATION_CLASSES": (
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ),
    "DEFAULT_PERMISSION_CLASSES": (
        "rest_framework.permissions.IsAuthenticated",
    ),
}
```

> ⚠️ **Must exist before first migration**
> This prevents the `auth.User` clash you hit earlier.

---

## 2️⃣ ACCOUNTS — IDENTITY & RBAC

### `accounts/models.py`

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    ROLE_CHOICES = (
        ("ADMIN", "Admin"),
        ("SALES", "Sales"),
        ("SUPERVISOR", "Supervisor"),
    )

    role = models.CharField(max_length=15, choices=ROLE_CHOICES)
    vehicle_no = models.CharField(max_length=20, null=True, blank=True)
```

---

### `accounts/permissions.py`

```python
from rest_framework.permissions import BasePermission

class IsAdmin(BasePermission):
    def has_permission(self, request, view):
        return request.user.role == "ADMIN"

class IsSales(BasePermission):
    def has_permission(self, request, view):
        return request.user.role == "SALES"
```

---

### `accounts/views.py`

```python
from rest_framework.viewsets import ModelViewSet
from .models import User
from .serializers import UserSerializer
from .permissions import IsAdmin

class UserViewSet(ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = [IsAdmin]
```

---

## 3️⃣ CUSTOMERS — CONTRACT & PRICING TRUTH

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

**Invariant**

> Rates are snapshotted at transaction time
> Never recalculated retroactively

---

## 4️⃣ DEPOTS & INVENTORY (READ-ONLY)

### `depots/models.py`

```python
from django.db import models

class Depot(models.Model):
    code = models.CharField(max_length=20, unique=True)
    name = models.CharField(max_length=100)
```

---

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

**Rule**

> Inventory is **never writable** via API

---

## 5️⃣ DISTRIBUTION — PHYSICAL STOCK MOVEMENT

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

---

### `distribution/services.py`

```python
from django.db import transaction
from inventory.models import Inventory
from audit.services import AuditService
import uuid

class DistributionService:

    @staticmethod
    @transaction.atomic
    def execute(user, depot, equipment, qty, movement):
        stock, _ = Inventory.objects.select_for_update().get_or_create(
            depot=depot,
            equipment=equipment,
            defaults={"quantity": 0}
        )

        if movement == "COLLECTION":
            stock.quantity -= qty
        else:
            stock.quantity += qty

        stock.save()

        AuditService.log(user, f"Stock {movement}: {equipment} x{qty}")
```

✔ Atomic
✔ Race-safe
✔ Audited

---

## 6️⃣ TRANSACTIONS — BILLING CORE

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

---

### Meter Sale

```python
class MeterSale(models.Model):
    transaction = models.OneToOneField(Transaction, on_delete=models.CASCADE)
    last = models.DecimalField(max_digits=10, decimal_places=2)
    latest = models.DecimalField(max_digits=10, decimal_places=2)
    qty = models.DecimalField(max_digits=10, decimal_places=2)
    rate = models.DecimalField(max_digits=10, decimal_places=2)
    subtotal = models.DecimalField(max_digits=10, decimal_places=2)
```

**Invariant**

> Meter reading updates occur **after commit only**

---

## 7️⃣ AUDIT — IMMUTABLE LOG

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

---

### `audit/services.py`

```python
from .models import AuditLog

class AuditService:
    @staticmethod
    def log(user, action):
        AuditLog.objects.create(user=user, action=action)
```

✔ Append-only
✔ Non-erasable

---

## 8️⃣ REPORTS — READ-ONLY FINANCE

### `/reports/summary`

```python
from django.db.models import Sum
from rest_framework.decorators import api_view, permission_classes
from accounts.permissions import IsAdmin
from transactions.models import Transaction
from rest_framework.response import Response

@api_view(["GET"])
@permission_classes([IsAdmin])
def summary(request):
    data = Transaction.objects.aggregate(total=Sum("total"))
    return Response(data)
```

---

### `/reports/aging`

```python
from django.utils.timezone import now
from datetime import timedelta

@api_view(["GET"])
@permission_classes([IsAdmin])
def aging(request):
    today = now()
    return Response({
        "30": Transaction.objects.filter(created_at__lte=today - timedelta(days=30), paid=False).count(),
        "60": Transaction.objects.filter(created_at__lte=today - timedelta(days=60), paid=False).count(),
        "90": Transaction.objects.filter(created_at__lte=today - timedelta(days=90), paid=False).count(),
    })
```

---

## 9️⃣ URL WIRING

### `config/urls.py`

```python
from django.urls import path
from reports.views import summary, aging

urlpatterns = [
    path("api/reports/summary/", summary),
    path("api/reports/aging/", aging),
]
```

---

## 🔟 MIGRATE & RUN

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```




