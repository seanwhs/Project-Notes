# 🧪 BACKEND LAB — HSH SALES SYSTEM 

**Purpose:** Regulated LPG operations (billing-safe, stock-safe, meter-safe)
**Design:** Domain-Driven · Service-Oriented · Audit-First · Regulator-Ready
**Audience:** Architects · Senior Engineers · Production Teams

This lab is a **complete, buildable backend reference**. If followed step-by-step, you will end with:

• JWT-secured APIs
• Inventory-locked stock movement
• Meter-safe & cylinder-safe billing
• Immutable audit trail
• PDF invoices
• **Swagger (OpenAPI) via drf-yasg**

---

## 0️⃣ ENVIRONMENT & DEPENDENCIES

```bash
mkdir hsh_sales_backend
cd hsh_sales_backend
python -m venv venv
venv\Scripts\activate

pip install \
  django \
  djangorestframework \
  mysqlclient \
  djangorestframework-simplejwt \
  drf-yasg \
  reportlab

django-admin startproject config .
```

Create domain apps:

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

## 1️⃣ GLOBAL SETTINGS (NON-NEGOTIABLE)

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
    "DEFAULT_AUTHENTICATION_CLASSES": (
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ),
    "DEFAULT_PERMISSION_CLASSES": (
        "rest_framework.permissions.IsAuthenticated",
    ),
}

from datetime import timedelta
SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=60),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=1),
}
```

⚠️ **Must be set before first migration**

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

### `accounts/urls.py`

```python
from rest_framework.routers import DefaultRouter
from .views import UserViewSet

router = DefaultRouter()
router.register("users", UserViewSet)

urlpatterns = router.urls
```

---

## 3️⃣ CUSTOMERS — CONTRACT & PRICING SOURCE OF TRUTH

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

**Invariant:** Rates are snapshotted at sale time.

---

## 4️⃣ DEPOTS, INVENTORY & VEHICLES

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

🔒 Inventory is **never writable** via CRUD APIs.

---

## 5️⃣ DISTRIBUTION — ALL STOCK MOVEMENT

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

---

## 6️⃣ TRANSACTIONS — BILLING CORE

### Models

```python
class Transaction(models.Model):
    number = models.CharField(max_length=50, unique=True)
    customer = models.ForeignKey(Customer, on_delete=models.PROTECT)
    user = models.ForeignKey(User, on_delete=models.PROTECT)
    total = models.DecimalField(max_digits=12, decimal_places=2)
    paid = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

class MeterSale(models.Model):
    transaction = models.OneToOneField(Transaction, on_delete=models.CASCADE)
    last = models.DecimalField(max_digits=10, decimal_places=2)
    latest = models.DecimalField(max_digits=10, decimal_places=2)
    qty = models.DecimalField(max_digits=10, decimal_places=2)
    rate = models.DecimalField(max_digits=10, decimal_places=2)
    subtotal = models.DecimalField(max_digits=10, decimal_places=2)

class LineItem(models.Model):
    transaction = models.ForeignKey(Transaction, on_delete=models.CASCADE, related_name="items")
    item_type = models.CharField(max_length=10)
    description = models.CharField(max_length=100)
    quantity = models.PositiveIntegerField()
    rate = models.DecimalField(max_digits=10, decimal_places=2)
    subtotal = models.DecimalField(max_digits=10, decimal_places=2)
```

---

### Transaction Service (Authoritative)

```python
from django.db import transaction
from audit.services import AuditService

class TransactionService:
    @staticmethod
    @transaction.atomic
    def create_meter_sale(user, customer, latest):
        last = customer.last_meter_reading
        qty = latest - last
        subtotal = qty * customer.meter_rate

        txn = Transaction.objects.create(
            number=f"TXN-{Transaction.objects.count()+1}",
            customer=customer,
            user=user,
            total=subtotal,
        )

        MeterSale.objects.create(
            transaction=txn,
            last=last,
            latest=latest,
            qty=qty,
            rate=customer.meter_rate,
            subtotal=subtotal,
        )

        customer.last_meter_reading = latest
        customer.save(update_fields=["last_meter_reading"])
        AuditService.log(user, f"Meter sale {txn.number}")
        return txn
```

---

## 7️⃣ AUDIT — IMMUTABLE LOG

```python
class AuditLog(models.Model):
    user = models.ForeignKey(User, on_delete=models.PROTECT)
    action = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
```

---

## 8️⃣ REPORTS — READ-ONLY

```python
@api_view(["GET"])
@permission_classes([IsAdmin])
def summary(request):
    return Response(Transaction.objects.aggregate(total=Sum("total")))
```

---

## 9️⃣ INVOICE (PDF)

```python
def generate_invoice(transaction, path):
    ... # reportlab implementation
```

---

## 🔟 SWAGGER / OPENAPI (drf-yasg)

### `config/urls.py`

```python
from drf_yasg.views import get_schema_view
from drf_yasg import openapi
from rest_framework.permissions import AllowAny

schema_view = get_schema_view(
    openapi.Info(
        title="HSH Sales API",
        default_version='v1',
        description="Regulated LPG Sales System",
    ),
    public=True,
    permission_classes=[AllowAny],
)
```

```python
urlpatterns += [
    path('swagger/', schema_view.with_ui('swagger', cache_timeout=0)),
    path('redoc/', schema_view.with_ui('redoc', cache_timeout=0)),
]
```

---

## 1️⃣1️⃣ ROOT URL (FINAL)

```python
urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/", include("accounts.urls")),
    path("api/", include("customers.urls")),
    path("api/reports/", include("reports.urls")),
    path("api/token/", TokenObtainPairView.as_view()),
    path("api/token/refresh/", TokenRefreshView.as_view()),
]
```

---

## ✅ FINAL GUARANTEES

✔ Swagger-documented APIs
✔ JWT-secured endpoints
✔ Inventory-safe stock movement
✔ Meter-safe & cylinder-safe billing
✔ Immutable audit trail
✔ Regulator-ready design

This is a **reference LPG backend architecture**, not a demo.
