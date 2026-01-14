# 🏗️ HSH Sales System — Full Stack Lab Tutorial

**Objective:** Build a **production-grade LPG logistics and sales system** using **Django REST Framework** (backend) and **React + React Router v7** (frontend), fully Dockerized, offline-aware, and audit-ready.

---

## **Step 0 — Prerequisites**

Make sure you have:

* **Docker & Docker Compose**
* **Python 3.11+**
* **Node.js 20+ & npm**
* **Postman** (optional, for API testing)
* **VSCode** (recommended)

---

## **Step 1 — Create Project Structure**

Create the main folders:

```bash
mkdir hsh_sales_system
cd hsh_sales_system
mkdir backend frontend
```

Inside **backend**:

```
backend/
├── accounts/
├── customers/
├── depots/
├── inventory/
├── distribution/
├── transactions/
├── audit/
├── reports/
└── config/
```

Inside **frontend**:

```
frontend/
├── src/
│   ├── layouts/
│   ├── routes/
│   ├── components/
│   └── index.css
├── package.json
└── main.jsx
```

---

## **Step 2 — Initialize Backend (Django + DRF)**

### 2.1 Create a virtual environment

```bash
cd backend
python -m venv venv
source venv/bin/activate
```

### 2.2 Install packages

```bash
pip install django djangorestframework mysqlclient djangorestframework-simplejwt
```

### 2.3 Create Django project

```bash
django-admin startproject config .
```

### 2.4 Create apps

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

## **Step 3 — Configure Database**

### 3.1 MySQL Setup (local or Docker)

**Docker MySQL**:

```yaml
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

### 3.2 Django settings (`config/settings.py`)

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'hsh_sales',
        'USER': 'hsh',
        'PASSWORD': 'hshpass',
        'HOST': 'db',
        'PORT': '3306',
    }
}
```

---

## **Step 4 — Implement Backend Models**

### 4.1 Accounts (`accounts/models.py`)

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

class User(AbstractUser):
    ROLE_CHOICES = (('ADMIN','Admin'),('SALES','Sales'))
    role = models.CharField(max_length=10, choices=ROLE_CHOICES)
    vehicle_no = models.CharField(max_length=20, blank=True, null=True)
```

### 4.2 Customers (`customers/models.py`)

```python
from django.db import models

class Customer(models.Model):
    PAYMENT = [('CASH','Cash'),('CREDIT','Credit')]
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

### 4.3 Depots & Inventory

```python
# depots/models.py
class Depot(models.Model):
    code = models.CharField(max_length=20, unique=True)
    name = models.CharField(max_length=100)
```

```python
# inventory/models.py
from django.db import models
from depots.models import Depot

class Inventory(models.Model):
    depot = models.ForeignKey(Depot, on_delete=models.CASCADE)
    equipment_name = models.CharField(max_length=50)
    quantity = models.IntegerField(default=0)

    class Meta:
        unique_together = ('depot','equipment_name')
```

---

## **Step 5 — Implement Services & Atomic Operations**

Example: **Distribution Service (`distribution/services.py`)**

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
            depot=depot, equipment_name=equipment, defaults={'quantity':0}
        )

        if status == 'COLLECTION':
            inventory.quantity -= quantity
        else:
            inventory.quantity += quantity
        inventory.save()

        dist = Distribution.objects.create(
            distribution_no=f"DIST-{uuid.uuid4().hex[:8]}",
            user=user, depot=depot, equipment_name=equipment,
            quantity=quantity, status=status
        )

        AuditService.log(user,f"Distribution {dist.distribution_no}")
        return dist
```

✔ Atomic, audit-safe, inventory-safe.

---

## **Step 6 — Setup REST API**

### 6.1 Serializers (`accounts/serializers.py`)

```python
from rest_framework import serializers
from .models import User

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id','username','role','vehicle_no']
```

### 6.2 ViewSets (`accounts/views.py`)

```python
from rest_framework.viewsets import ModelViewSet
from rest_framework.permissions import IsAuthenticated
from .models import User
from .serializers import UserSerializer

class UserViewSet(ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = [IsAuthenticated]
```

### 6.3 URLs (`config/urls.py`)

```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from accounts.views import UserViewSet

router = DefaultRouter()
router.register(r'users', UserViewSet)

urlpatterns = [
    path('api/', include(router.urls)),
]
```

---

## **Step 7 — Frontend Setup (React + RR7)**

```bash
cd ../frontend
npm create vite@latest
# Choose React + JS
npm install react-router-dom@7 tailwindcss
npx tailwindcss init -p
```

**Directory Structure:**

```
src/
├── main.jsx
├── router.jsx
├── api.js
├── layouts/DashboardLayout.jsx
├── routes/login.jsx
├── routes/dashboard.jsx
```

---

### 7.1 Main Entry (`main.jsx`)

```jsx
import React from "react";
import ReactDOM from "react-dom/client";
import { RouterProvider } from "react-router-dom";
import router from "./router";
import "./index.css";

ReactDOM.createRoot(document.getElementById("root")).render(
  <React.StrictMode>
    <RouterProvider router={router} />
  </React.StrictMode>
);
```

---

### 7.2 Router (`router.jsx`)

```jsx
import { createBrowserRouter, redirect } from "react-router-dom";
import DashboardLayout from "./layouts/DashboardLayout";
import Login from "./routes/login";
import Dashboard from "./routes/dashboard";

export default createBrowserRouter([
  { path: "/login", element: <Login /> },
  {
    element: <DashboardLayout />,
    children: [
      { path: "/", element: <Dashboard /> },
    ]
  }
]);
```

---

### 7.3 API Helper (`api.js`)

```js
export async function api(url, options = {}) {
  return fetch(`/api${url}`, {
    credentials: "include",
    headers: { "Content-Type": "application/json" },
    ...options,
  });
}
```

---

## **Step 8 — Dockerize Full Stack**

**docker-compose.yml**

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
    ports: ["3306:3306"]

  backend:
    build: ./backend
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./backend:/app
    ports: ["8000:8000"]
    depends_on: ["db"]

  frontend:
    build: ./frontend
    command: npm run dev -- --host
    volumes:
      - ./frontend:/app
    ports: ["5173:5173"]
    depends_on: ["backend"]
```

Run:

```bash
docker-compose up --build
```

---

## **Step 9 — Test Workflows**

1. Go to `http://localhost:5173` → Login
2. Test `/distribution` → create stock movements
3. Test `/transaction/:id` → create transaction with meter/cylinder
4. Check `/inventory` → depot-safe stock
5. Check `/reports` → verify totals
6. Test audit log `/audit` → immutable

---

## ✅ **Step 10 — Optional Enhancements**

* **Offline queue** with LocalStorage
* **Thermal printing** with ESC/POS
* **PDF invoices** via WeasyPrint
* **QuickBooks sync** with Celery


