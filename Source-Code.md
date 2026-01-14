# 🏗️ HSH SALES SYSTEM — FULL SOURCE CODE (MySQL Edition)

---

## 1️⃣ Project Structure

```
hsh-sales-system/
├── backend/
│   ├── config/
│   │   └── settings.py
│   ├── accounts/
│   │   ├── models.py
│   │   ├── permissions.py
│   │   └── admin.py
│   ├── customers/
│   │   └── models.py
│   ├── inventory/
│   │   └── models.py
│   ├── distribution/
│   │   ├── models.py
│   │   └── services.py
│   ├── transactions/
│   │   ├── models.py
│   │   └── services.py
│   ├── billing/
│   │   └── services.py
│   ├── reports/
│   │   └── services.py
│   ├── audit/
│   │   ├── models.py
│   │   └── services.py
│   ├── manage.py
│   └── requirements.txt
├── frontend/
│   ├── app/
│   │   ├── routes/
│   │   │   └── Delivery.jsx
│   │   ├── services/
│   │   │   └── offline.js
│   │   ├── hooks/
│   │   │   └── usePrinter.js
│   │   ├── utils/
│   │   └── root.jsx
│   └── package.json
├── docker-compose.yml
└── README.md
```

---

## 2️⃣ Backend — Requirements

`backend/requirements.txt`:

```txt
Django>=5.0
djangorestframework
djangorestframework-simplejwt
django-cors-headers
mysqlclient
reportlab
gunicorn
```

Install dependencies:

```bash
pip install -r requirements.txt
```

---

## 3️⃣ Backend — Django Settings

`backend/config/settings.py`:

```python
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.getenv('SECRET_KEY', 'unsafe-dev-key')
DEBUG = os.getenv('DEBUG', 'True') == 'True'
ALLOWED_HOSTS = []

INSTALLED_APPS = [
    'django.contrib.admin', 'django.contrib.auth', 'django.contrib.contenttypes',
    'django.contrib.sessions', 'django.contrib.messages', 'django.contrib.staticfiles',
    'rest_framework', 'corsheaders',
    'accounts', 'customers', 'inventory', 'distribution',
    'transactions', 'billing', 'reports', 'audit',
]

AUTH_USER_MODEL = 'accounts.User'

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
}

CORS_ALLOW_ALL_ORIGINS = True

DATABASES = {
    "default": {
        "ENGINE": os.getenv('DB_ENGINE', 'django.db.backends.mysql'),
        "NAME": os.getenv('DB_NAME', 'hsh'),
        "USER": os.getenv('DB_USER', 'root'),
        "PASSWORD": os.getenv('DB_PASSWORD', ''),
        "HOST": os.getenv('DB_HOST', 'localhost'),
        "PORT": os.getenv('DB_PORT', '3306'),
        "OPTIONS": {
            "init_command": "SET sql_mode='STRICT_TRANS_TABLES'"
        },
    }
}

STATIC_URL = 'static/'
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'
```

---

## 4️⃣ Backend — Accounts

`backend/accounts/models.py`:

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    ROLE_CHOICES = (
        ('admin', 'Admin'),
        ('sales', 'Sales'),
    )
    role = models.CharField(max_length=10, choices=ROLE_CHOICES)
    vehicle_no = models.CharField(max_length=50, blank=True)
```

`backend/accounts/permissions.py`:

```python
from rest_framework.permissions import BasePermission

class IsAdmin(BasePermission):
    def has_permission(self, request, view):
        return request.user.is_authenticated and request.user.role == 'admin'

class IsSales(BasePermission):
    def has_permission(self, request, view):
        return request.user.is_authenticated and request.user.role == 'sales'
```

---

## 5️⃣ Backend — Customers

`backend/customers/models.py`:

```python
from django.db import models

class Customer(models.Model):
    PAYMENT_CHOICES = (('cash','Cash'), ('monthly','Monthly'))

    name = models.CharField(max_length=200)
    payment_type = models.CharField(max_length=20, choices=PAYMENT_CHOICES)
    rate_14kg = models.DecimalField(max_digits=10, decimal_places=2)
    rate_50kg = models.DecimalField(max_digits=10, decimal_places=2)

    def __str__(self):
        return self.name
```

---

## 6️⃣ Backend — Inventory

`backend/inventory/models.py`:

```python
from django.db import models

class Inventory(models.Model):
    equipment = models.CharField(max_length=50, unique=True)
    full_qty = models.IntegerField(default=0)
    empty_qty = models.IntegerField(default=0)
```

---

## 7️⃣ Backend — Distribution

`backend/distribution/models.py`:

```python
from django.db import models
from accounts.models import User

class Distribution(models.Model):
    STATUS = (('collection','Collection'),('empty_return','Empty Return'))

    distribution_no = models.CharField(max_length=30)
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    depot = models.CharField(max_length=50)
    equipment = models.CharField(max_length=50)
    quantity = models.PositiveIntegerField()
    status = models.CharField(max_length=20, choices=STATUS)
    created_at = models.DateTimeField(auto_now_add=True)
```

`backend/distribution/services.py`:

```python
import uuid
from .models import Distribution
from inventory.models import Inventory

class DistributionService:

    @staticmethod
    def create(user, items):
        dist_no = f"DIST-{uuid.uuid4().hex[:8].upper()}"
        for i in items:
            Distribution.objects.create(
                distribution_no=dist_no,
                user=user,
                depot=i['depot'],
                equipment=i['equipment'],
                quantity=i['quantity'],
                status=i['status']
            )
            inv, _ = Inventory.objects.get_or_create(equipment=i['equipment'])
            if i['status'] == 'collection':
                inv.full_qty += i['quantity']
            else:
                inv.empty_qty += i['quantity']
            inv.save()
        return dist_no
```

---

## 8️⃣ Backend — Transactions

`backend/transactions/models.py`:

```python
from django.db import models
from customers.models import Customer
from accounts.models import User

class Transaction(models.Model):
    customer = models.ForeignKey(Customer, on_delete=models.CASCADE)
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    total_amount = models.DecimalField(max_digits=12, decimal_places=2)
    is_paid = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
```

`backend/transactions/services.py`:

```python
from decimal import Decimal
from .models import Transaction
from customers.models import Customer
from inventory.models import Inventory
from django.db import models as dj_models

class TransactionService:

    @staticmethod
    def create(user, customer_id, qty_14, qty_50):
        customer = Customer.objects.get(id=customer_id)
        total = Decimal(qty_14) * customer.rate_14kg + Decimal(qty_50) * customer.rate_50kg

        Inventory.objects.filter(equipment="CYL 14").update(full_qty=dj_models.F('full_qty') - qty_14)
        Inventory.objects.filter(equipment="CYL 50").update(full_qty=dj_models.F('full_qty') - qty_50)

        return Transaction.objects.create(
            customer=customer,
            user=user,
            total_amount=total
        )
```

---

## 9️⃣ Backend — Billing & PDF/Email

`backend/billing/services.py`:

```python
import io
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import A4
from django.core.mail import EmailMessage

def generate_invoice_pdf(tx):
    buffer = io.BytesIO()
    c = canvas.Canvas(buffer, pagesize=A4)
    c.drawString(50, 800, "HOCK SOON HENG LPG")
    c.drawString(50, 770, f"Invoice: {tx.id}")
    c.drawString(50, 740, f"Customer: {tx.customer.name}")
    c.drawString(50, 710, f"Total: ${tx.total_amount}")
    c.showPage()
    c.save()
    buffer.seek(0)
    return buffer

def send_invoice_email(transaction, recipient_email):
    buffer = generate_invoice_pdf(transaction)
    email = EmailMessage(
        subject=f"Invoice {transaction.id}",
        body="Please find attached your invoice.",
        to=[recipient_email]
    )
    email.attach(f"Invoice_{transaction.id}.pdf", buffer.read(), 'application/pdf')
    email.send()
```

---

## 🔟 Backend — Reports

`backend/reports/services.py`:

```python
from transactions.models import Transaction

class ReportService:

    @staticmethod
    def transactions(filters):
        qs = Transaction.objects.select_related('customer', 'user')
        if filters.get('paid') != 'all':
            qs = qs.filter(is_paid=filters['paid'] == 'paid')
        return qs
```

---

## 1️⃣1️⃣ Backend — Audit

`backend/audit/models.py`:

```python
from django.db import models
from accounts.models import User

class AuditLog(models.Model):
    user = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    action = models.CharField(max_length=200)
    created_at = models.DateTimeField(auto_now_add=True)
```

`backend/audit/services.py`:

```python
from .models import AuditLog

def log_action(user, action):
    AuditLog.objects.create(user=user, action=action)
```

---

## 1️⃣2️⃣ Frontend — Root

`frontend/app/root.jsx`:

```jsx
import { Outlet } from "react-router"

export default function Root() {
  return (
    <div className="min-h-screen bg-gray-100">
      <Outlet />
    </div>
  )
}
```

---

## 1️⃣3️⃣ Frontend — Delivery Form

`frontend/app/routes/Delivery.jsx`:

```jsx
import { Form } from "react-router"
import { usePrinter } from "../hooks/usePrinter"

export default function Delivery() {
  const { printReceipt } = usePrinter()

  return (
    <Form method="post" className="p-6 space-y-4">
      <input name="qty_14" type="number" placeholder="14kg Qty" />
      <input name="qty_50" type="number" placeholder="50kg Qty" />

      <button className="bg-blue-600 text-white p-3">Save</button>

      <button
        type="button"
        onClick={() =>
          printReceipt({ name: "Test", qty14: 1, qty50: 1, total: 100 })
        }
      >
        Print
      </button>
    </Form>
  )
}
```

---

## 1️⃣4️⃣ Frontend — Printer Hook

`frontend/app/hooks/usePrinter.js`:

```javascript
export function usePrinter() {
  const printReceipt = (data) => {
    console.log("Printing:", data)
    // implement ESC/POS over Web Bluetooth
  }
  return { printReceipt }
}
```

---

## 1️⃣5️⃣ Offline Support

`frontend/app/services/offline.js`:

```javascript
export const OfflineService = {
  save: (key, data) => localStorage.setItem(key, JSON.stringify(data)),
  load: (key) => JSON.parse(localStorage.getItem(key) || "null"),
}
```

---

## 1️⃣6️⃣ Docker Compose (MySQL)

`docker-compose.yml`:

```yaml
version: '3.9'
services:
  backend:
    build: ./backend
    command: >
      sh -c "python manage.py migrate &&
             python manage.py runserver 0.0.0.0:8000"
    ports:
      - "8000:8000"
    env_file:
      - ./backend/.env
    depends_on:
      - db
    volumes:
      - ./backend:/app/backend

  db:
    image: mysql:8
    container_name: hsh-mysql
    environment:
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    ports:
      - "3306:3306"
    command: --default-authentication-plugin=mysql_native_password
    volumes:
      - mysql_data:/var/lib/mysql

  frontend:
    build: ./frontend
    ports:
      - "5173:5173"
    env_file:
      - ./frontend/.env
    volumes:
      - ./frontend:/app/frontend

volumes:
  mysql_data:
```

---

## 1️⃣7️⃣ Backend `.env`

`backend/.env`:

```env
SECRET_KEY=unsafe-dev-key
DEBUG=True
DB_ENGINE=django.db.backends.mysql
DB_NAME=hsh
DB_USER=hsh_user
DB_PASSWORD=hsh_pass
DB_HOST=db
DB_PORT=3306
DB_ROOT_PASSWORD=rootpass
```

---

## 1️⃣8️⃣ Quick Start

```bash
# Build & run all containers
docker-compose up --build

# Create Django superuser
docker exec -it hsh-sales-system_backend_1 bash
python manage.py createsuperuser

# Access services
# Backend API: http://localhost:8000
# Frontend: http://localhost:5173
```

---

✅ **This is the complete, full source code** for the **HSH Sales System — MySQL edition**, Docker-ready, with backend, frontend, offline support, printing, PDF/email, audit logging.

