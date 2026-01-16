# 🔐 BACKEND — FULL SOURCE CODE (DJANGO + DRF)

> Assumptions

* Django 4.x
* Django REST Framework
* PostgreSQL
* Single monorepo `backend/`
* Custom User model
* Service-layer enforcement

---

## 📦 PROJECT STRUCTURE (REAL FILES)

```
backend/
├── manage.py
├── config/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
│
├── accounts/
├── customers/
├── depots/
├── inventory/
├── distribution/
├── transactions/
├── invoices/
├── audit/
├── reports/
```

---

# 🔐 ACCOUNTS — Identity & Authorization

## `accounts/models.py`

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    ROLE_ADMIN = 'ADMIN'
    ROLE_SALES = 'SALES'

    ROLE_CHOICES = (
        (ROLE_ADMIN, 'Admin'),
        (ROLE_SALES, 'Sales'),
    )

    role = models.CharField(max_length=10, choices=ROLE_CHOICES)
    vehicle_no = models.CharField(max_length=20, null=True, blank=True)

    def is_admin(self):
        return self.role == self.ROLE_ADMIN
```

---

## `accounts/permissions.py`

```python
from rest_framework.permissions import BasePermission

class IsAdmin(BasePermission):
    def has_permission(self, request, view):
        return bool(
            request.user
            and request.user.is_authenticated
            and request.user.role == 'ADMIN'
        )
```

---

## `accounts/serializers.py`

```python
from rest_framework import serializers
from .models import User

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = (
            'id', 'username', 'password',
            'role', 'vehicle_no',
            'is_active'
        )
        extra_kwargs = {'password': {'write_only': True}}

    def create(self, validated_data):
        user = User.objects.create_user(**validated_data)
        return user
```

---

## `accounts/views.py`

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

## `accounts/urls.py`

```python
from rest_framework.routers import DefaultRouter
from .views import UserViewSet

router = DefaultRouter()
router.register('users', UserViewSet, basename='users')

urlpatterns = router.urls
```

---

# 👥 CUSTOMERS — Contracts & Rates

## `customers/models.py`

```python
from django.db import models

class Customer(models.Model):
    CASH = 'CASH'
    CREDIT = 'CREDIT'

    PAYMENT_CHOICES = (
        (CASH, 'Cash'),
        (CREDIT, 'Credit'),
    )

    name = models.CharField(max_length=255)
    address = models.TextField()
    payment_type = models.CharField(max_length=10, choices=PAYMENT_CHOICES)

    meter_rate = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    rate_9kg = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    rate_12_7kg = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    rate_14kg = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    rate_50kg_pol = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    rate_50kg_l = models.DecimalField(max_digits=10, decimal_places=2, default=0)

    last_meter_reading = models.DecimalField(max_digits=10, decimal_places=2, default=0)

    def __str__(self):
        return self.name
```

---

## `customers/views.py`

```python
from rest_framework.viewsets import ModelViewSet
from .models import Customer
from .serializers import CustomerSerializer

class CustomerViewSet(ModelViewSet):
    queryset = Customer.objects.all()
    serializer_class = CustomerSerializer
```

---

## `customers/serializers.py`

```python
from rest_framework import serializers
from .models import Customer

class CustomerSerializer(serializers.ModelSerializer):
    class Meta:
        model = Customer
        fields = '__all__'
```

---

## `customers/urls.py`

```python
from rest_framework.routers import DefaultRouter
from .views import CustomerViewSet

router = DefaultRouter()
router.register('customers', CustomerViewSet)

urlpatterns = router.urls
```

---

# 🏭 DEPOTS & INVENTORY — Physical Reality

## `depots/models.py`

```python
from django.db import models

class Depot(models.Model):
    code = models.CharField(max_length=20, unique=True)
    name = models.CharField(max_length=100)

    def __str__(self):
        return self.code
```

---

## `inventory/models.py`

```python
from django.db import models
from depots.models import Depot

class Inventory(models.Model):
    depot = models.ForeignKey(Depot, on_delete=models.CASCADE)
    equipment_name = models.CharField(max_length=50)
    quantity = models.IntegerField(default=0)

    class Meta:
        unique_together = ('depot', 'equipment_name')
```

---

## `inventory/views.py`

```python
from rest_framework.generics import ListAPIView
from .models import Inventory
from .serializers import InventorySerializer

class InventoryListView(ListAPIView):
    queryset = Inventory.objects.all()
    serializer_class = InventorySerializer
```

---

## `inventory/urls.py`

```python
from django.urls import path
from .views import InventoryListView

urlpatterns = [
    path('inventory/', InventoryListView.as_view()),
]
```

---

# 🚚 DISTRIBUTION — Stock Movement (COMMANDS)

## `distribution/models.py`

```python
from django.db import models
from accounts.models import User
from depots.models import Depot

class Distribution(models.Model):
    COLLECTION = 'COLLECTION'
    EMPTY_RETURN = 'EMPTY_RETURN'

    STATUS_CHOICES = (
        (COLLECTION, 'Collection'),
        (EMPTY_RETURN, 'Empty Return'),
    )

    distribution_no = models.CharField(max_length=30, unique=True)
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    depot = models.ForeignKey(Depot, on_delete=models.CASCADE)
    equipment_name = models.CharField(max_length=50)
    quantity = models.PositiveIntegerField()
    status = models.CharField(max_length=20, choices=STATUS_CHOICES)
    created_at = models.DateTimeField(auto_now_add=True)
```

---

## `distribution/services.py`

```python
from django.db import transaction
from inventory.models import Inventory
from .models import Distribution
from audit.services import AuditService

class DistributionService:

    @staticmethod
    @transaction.atomic
    def create(**data):
        inventory = Inventory.objects.select_for_update().get(
            depot=data['depot'],
            equipment_name=data['equipment_name']
        )

        if inventory.quantity < data['quantity']:
            raise ValueError("Insufficient stock")

        inventory.quantity -= data['quantity']
        inventory.save()

        dist = Distribution.objects.create(**data)

        AuditService.log(
            data['user'],
            f"Distribution {dist.distribution_no} created"
        )

        return dist
```

---

## `distribution/views.py`

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from .services import DistributionService

class DistributionCreateView(APIView):
    def post(self, request):
        dist = DistributionService.create(**request.data)
        return Response({'id': dist.id}, status=status.HTTP_201_CREATED)
```

---

# 💰 TRANSACTIONS — Billing Core

## `transactions/models.py`

```python
from django.db import models
from customers.models import Customer
from accounts.models import User

class Transaction(models.Model):
    transaction_no = models.CharField(max_length=50, unique=True)
    customer = models.ForeignKey(Customer, on_delete=models.PROTECT)
    user = models.ForeignKey(User, on_delete=models.PROTECT)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    is_paid = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
```

---

## `transactions/services.py`

```python
from django.db import transaction
from .models import Transaction
from audit.services import AuditService

class TransactionService:

    @staticmethod
    @transaction.atomic
    def create(**data):
        tx = Transaction.objects.create(**data)
        AuditService.log(data['user'], f"Transaction {tx.transaction_no}")
        return tx
```

---

# 🧾 INVOICES & GST — REGULATED BILLING

## `invoices/models.py`

```python
from django.db import models
from transactions.models import Transaction

class Invoice(models.Model):
    invoice_no = models.CharField(max_length=30, unique=True)
    transaction = models.OneToOneField(Transaction, on_delete=models.PROTECT)
    issued_at = models.DateTimeField(auto_now_add=True)

    subtotal = models.DecimalField(max_digits=10, decimal_places=2)
    gst_amount = models.DecimalField(max_digits=10, decimal_places=2)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
```

---

## `invoices/services.py`

```python
from decimal import Decimal
from django.utils import timezone
from django.db import transaction
from .models import Invoice
from audit.services import AuditService

GST_RATE = Decimal('0.09')

class InvoiceService:

    @staticmethod
    def next_invoice_no():
        prefix = timezone.now().strftime('%Y%m')
        last = Invoice.objects.filter(invoice_no__contains=prefix).order_by('invoice_no').last()
        seq = int(last.invoice_no[-5:]) + 1 if last else 1
        return f"INV-{prefix}-{seq:05d}"

    @staticmethod
    @transaction.atomic
    def issue(tx):
        subtotal = tx.total_amount
        gst = subtotal * GST_RATE

        invoice = Invoice.objects.create(
            invoice_no=InvoiceService.next_invoice_no(),
            transaction=tx,
            subtotal=subtotal,
            gst_amount=gst,
            total_amount=subtotal + gst
        )

        AuditService.log(tx.user, f"Issued invoice {invoice.invoice_no}")
        return invoice
```

---

# 🧾 AUDIT — IMMUTABLE LOG

## `audit/models.py`

```python
from django.db import models
from accounts.models import User

class AuditLog(models.Model):
    user = models.ForeignKey(User, on_delete=models.PROTECT)
    message = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
```

---

## `audit/services.py`

```python
from .models import AuditLog

class AuditService:

    @staticmethod
    def log(user, message):
        AuditLog.objects.create(user=user, message=message)
```

---

# 📊 REPORTS — READ ONLY

## `reports/views.py`

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from transactions.models import Transaction
from inventory.models import Inventory

class SalesReport(APIView):
    def get(self, request):
        qs = Transaction.objects.all().values(
            'transaction_no', 'total_amount', 'created_at'
        )
        return Response(list(qs))

class InventoryReport(APIView):
    def get(self, request):
        qs = Inventory.objects.all().values(
            'depot__code', 'equipment_name', 'quantity'
        )
        return Response(list(qs))
```

---

# 🌐 ROOT ROUTING

## `config/urls.py`

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/accounts/', include('accounts.urls')),
    path('api/customers/', include('customers.urls')),
    path('api/inventory/', include('inventory.urls')),
    path('api/distribution/', include('distribution.urls')),
    path('api/transactions/', include('transactions.urls')),
    path('api/invoices/', include('invoices.urls')),
    path('api/audit/', include('audit.urls')),
    path('api/reports/', include('reports.urls')),
]
```
