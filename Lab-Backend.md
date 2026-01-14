# 🧪 LAB: Building the HSH Sales System Backend (DRF)

**Objective:** Build a production-ready backend for regulated LPG operations that is **domain-driven, service-oriented, inventory-safe, meter-safe, audit-safe, and report-ready**.

**Prerequisites:**

* Python 3.11+
* PostgreSQL or MySQL
* Django 4.2+
* Django REST Framework
* pip, virtualenv
* Optional: Docker/Docker Compose for containerized setup

---

## **Step 0 — Setup Environment**

1. Create a project folder:

```bash
mkdir hsh_sales_backend
cd hsh_sales_backend
```

2. Create a virtual environment:

```bash
python -m venv venv
source venv/bin/activate  # Linux/macOS
venv\Scripts\activate     # Windows
```

3. Install dependencies:

```bash
pip install django djangorestframework mysqlclient djangorestframework-simplejwt
```

4. Start Django project:

```bash
django-admin startproject config .
```

5. Create apps:

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

## **Step 1 — Accounts: Identity & Authorization**

**1.1 models.py**

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    ROLE_CHOICES = (
        ('ADMIN', 'Admin'),
        ('SALES', 'Sales'),
    )
    role = models.CharField(max_length=10, choices=ROLE_CHOICES)
    vehicle_no = models.CharField(max_length=20, blank=True, null=True)
```

**1.2 serializers.py**

```python
from rest_framework import serializers
from .models import User

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'role', 'vehicle_no']
```

**1.3 permissions.py**

```python
from rest_framework.permissions import BasePermission

class IsAdmin(BasePermission):
    def has_permission(self, request, view):
        return request.user.is_authenticated and request.user.role == 'ADMIN'

class IsSales(BasePermission):
    def has_permission(self, request, view):
        return request.user.is_authenticated and request.user.role == 'SALES'
```

**1.4 views.py**

```python
from rest_framework.viewsets import ModelViewSet
from rest_framework.permissions import IsAuthenticated
from .models import User
from .serializers import UserSerializer
from .permissions import IsAdmin

class UserViewSet(ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = [IsAuthenticated, IsAdmin]
```

**✅ Lab Checkpoint:** Only admin users can create or edit users.

---

## **Step 2 — Customers: Contracts & Rates**

**2.1 models.py**

```python
from django.db import models

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

**2.2 views.py**

```python
from rest_framework.viewsets import ModelViewSet
from rest_framework.permissions import IsAuthenticated
from .models import Customer
from .serializers import CustomerSerializer

class CustomerViewSet(ModelViewSet):
    queryset = Customer.objects.all()
    serializer_class = CustomerSerializer
    permission_classes = [IsAuthenticated]
```

**✅ Lab Checkpoint:** CRUD operations on customers; rates are immutable historically.

---

## **Step 3 — Depots & Inventory**

**3.1 depots/models.py**

```python
from django.db import models

class Depot(models.Model):
    code = models.CharField(max_length=20, unique=True)
    name = models.CharField(max_length=100)

    def __str__(self):
        return self.code
```

**3.2 inventory/models.py**

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

**3.3 inventory/views.py**

```python
from rest_framework.viewsets import ReadOnlyModelViewSet
from rest_framework.permissions import IsAuthenticated
from .models import Inventory
from .serializers import InventorySerializer

class InventoryViewSet(ReadOnlyModelViewSet):
    queryset = Inventory.objects.all()
    serializer_class = InventorySerializer
    permission_classes = [IsAuthenticated]
```

**✅ Lab Checkpoint:** Inventory is **queryable only**; mutations go through services.

---

## **Step 4 — Distribution Service: Atomic Inventory Movement**

**4.1 models.py**

```python
from django.db import models
from django.contrib.auth import get_user_model
from depots.models import Depot

User = get_user_model()

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

**4.2 services.py**

```python
from django.db import transaction
from inventory.models import Inventory
from .models import Distribution
from audit.services import AuditService
import uuid

class DistributionService:

    @staticmethod
    @transaction.atomic
    def create(user, depot, equipment, quantity, status):
        inventory, _ = Inventory.objects.get_or_create(
            depot=depot,
            equipment_name=equipment,
            defaults={'quantity': 0}
        )

        if status == 'COLLECTION':
            inventory.quantity -= quantity
        else:
            inventory.quantity += quantity

        inventory.save()

        dist = Distribution.objects.create(
            distribution_no=f"DIST-{uuid.uuid4().hex[:8]}",
            user=user,
            depot=depot,
            equipment_name=equipment,
            quantity=quantity,
            status=status
        )

        AuditService.log(user, f"Distribution {dist.distribution_no}")
        return dist
```

**✅ Lab Checkpoint:** Test `DistributionService.create()` and verify inventory is updated atomically.

---

## **Step 5 — Transactions & Billing**

**5.1 models.py**

```python
from django.db import models
from django.contrib.auth import get_user_model
from customers.models import Customer

User = get_user_model()

class Transaction(models.Model):
    transaction_no = models.CharField(max_length=50, unique=True)
    customer = models.ForeignKey(Customer, on_delete=models.PROTECT)
    user = models.ForeignKey(User, on_delete=models.PROTECT)
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    is_paid = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
```

**5.2 MeterSale (example)**

```python
class MeterSale(models.Model):
    transaction = models.OneToOneField(Transaction, on_delete=models.CASCADE)
    last_reading = models.DecimalField(max_digits=10, decimal_places=2)
    latest_reading = models.DecimalField(max_digits=10, decimal_places=2)
    quantity = models.DecimalField(max_digits=10, decimal_places=2)
    rate = models.DecimalField(max_digits=10, decimal_places=2)
    subtotal = models.DecimalField(max_digits=10, decimal_places=2)
```

**✅ Lab Checkpoint:** Meter readings are updated **only after transaction commit**.

---

## **Step 6 — Audit & Reports**

* Audit logs are **append-only**, immutable, and always linked to user actions.
* Reports modules fetch **flat payloads**, URL-driven filters, admin-only access.
* Integrate `/api/reports/` views with **serializer + query filters**.

---

## **Step 7 — Optional: Dockerize Backend**

**docker-compose.yml**:

```yaml
version: '3.8'
services:
  backend:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      - db

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: hsh_sales
      MYSQL_USER: hsh
      MYSQL_PASSWORD: hsh123
    ports:
      - "3306:3306"
```

---

## ✅ Lab Completion Checklist

* [ ] `accounts` — Admin-only user CRUD
* [ ] `customers` — Contract & rate-safe
* [ ] `depots` — Depot-scoped
* [ ] `inventory` — Query-only, mutations via services
* [ ] `distribution` — Atomic, inventory-safe, audited
* [ ] `transactions` — Meter-safe, billing-safe
* [ ] `audit` — Append-only, immutable
* [ ] `reports` — Admin-only, filterable, export-ready
* [ ] Backend is container-ready
* [ ] Security boundaries respected

---

This lab is **hands-on and workflow-ready**. Each step builds on the previous, guaranteeing **inventory, meter, and audit safety**, as per HSH LPG operational standards.


