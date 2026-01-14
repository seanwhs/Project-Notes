# 🧪 HSH Sales System — Full Stack Lab Guide

**Objective:** Deploy a **full-stack HSH Sales System** (Django REST Framework backend + React + RR7 frontend), validate workflow correctness, simulate **load and offline scenarios**, and verify **audit integrity**.

---

## **Step 0 — Prerequisites**

* Docker & Docker Compose installed
* Node.js 20+ and npm
* Postman or HTTP client for testing
* Python 3.11+ if running backend outside Docker
* Recommended: VSCode + Docker extension

---

## **Step 1 — Project Structure**

```
hsh_sales_system/
├── backend/
│   ├── accounts/
│   ├── customers/
│   ├── depots/
│   ├── inventory/
│   ├── distribution/
│   ├── transactions/
│   ├── audit/
│   ├── reports/
│   └── config/
├── frontend/
│   ├── src/
│   └── package.json
├── docker-compose.yml
└── README.md
```

**Key principle:** Frontend & backend **isolated but networked via Docker**.

---

## **Step 2 — Docker Compose Setup**

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
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql

  backend:
    build: ./backend
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./backend:/app
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=mysql://hsh:hshpass@db:3306/hsh_sales
    depends_on:
      - db

  frontend:
    build: ./frontend
    command: npm run dev -- --host
    volumes:
      - ./frontend:/app
    ports:
      - "5173:5173"
    depends_on:
      - backend

volumes:
  db_data:
```

---

## **Step 3 — Backend Setup**

**1. Install Python requirements**

```bash
cd backend
python -m venv venv
source venv/bin/activate
pip install django djangorestframework mysqlclient
pip install djangorestframework-simplejwt
```

**2. Configure `.env`**

```
SECRET_KEY=supersecret
DEBUG=1
DB_NAME=hsh_sales
DB_USER=hsh
DB_PASSWORD=hshpass
DB_HOST=db
DB_PORT=3306
```

**3. Migrate & create superuser**

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
```

**4. Verify API**

```
curl http://localhost:8000/api/customers/
```

---

## **Step 4 — Frontend Setup**

**1. Install dependencies**

```bash
cd frontend
npm install
```

**2. Start dev server**

```bash
npm run dev
```

Visit: `http://localhost:5173/`

---

## **Step 5 — Test Core Workflows**

### **5.1 Distribution**

1. Go to `/distribution`
2. Select Depot, Equipment, Quantity, Status → Submit
3. Verify inventory decreases/increases **only via backend service**

### **5.2 Transactions**

1. Go to `/transaction/:id`
2. Fill Meter / Cylinder / Service sections
3. Submit → Transaction record created
4. Check **audit log**: `/audit/`

### **5.3 Reports (Admin)**

1. Visit `/reports`
2. Filter by date / customer
3. Export → Verify totals match transaction table

---

## **Step 6 — Load Test Data**

We can generate **dummy load** via Python management command.

**backend/load_test_data.py**

```python
import os, django, random
from customers.models import Customer
from depots.models import Depot
from inventory.models import Inventory
from django.contrib.auth import get_user_model

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')
django.setup()

User = get_user_model()

# Create depots
for i in range(3):
    Depot.objects.get_or_create(code=f"DPT{i+1}", defaults={'name': f"Depot {i+1}"})

# Create inventory
for depot in Depot.objects.all():
    for eq in ["Cylinder 9kg", "Cylinder 12.7kg", "Meter"]:
        Inventory.objects.get_or_create(depot=depot, equipment_name=eq, defaults={'quantity': 100})

# Create customers
for i in range(20):
    Customer.objects.get_or_create(name=f"Customer {i+1}", address=f"Address {i+1}", payment_type=random.choice(["CASH","CREDIT"]))
```

Run:

```bash
python load_test_data.py
```

---

## **Step 7 — Load Testing Transactions**

**1. Python script for bulk transactions**

```python
import requests, random

API = "http://localhost:8000/api/transactions/create_tx/"
CUSTOMER_IDS = list(range(1, 21))
USER_ID = 1  # Superuser or sales user

for i in range(50):
    customer = random.choice(CUSTOMER_IDS)
    payload = {
        "customer": customer,
        "user": USER_ID,
        "total_amount": random.randint(50,500),
        "is_paid": random.choice([True,False])
    }
    r = requests.post(API, json=payload)
    print(r.status_code, r.json())
```

**2. Validate**

* Check `/transactions/` → 50 new transactions
* Check `/audit/` → all logged

---

## **Step 8 — Offline Queue Simulation**

1. Go offline in browser
2. Submit forms → queued in **localStorage**:

```js
localStorage.setItem("offlineQueue", JSON.stringify([{ action: "distribution", depot:1, quantity:5 }]));
```

3. Reconnect → flush queue to backend via loader/action

✔ Backend validation ensures **inventory safety**

---

## **Step 9 — Audit Verification**

1. Visit `/audit/`
2. Verify:

* Timestamp server-generated
* User attribution present
* Immutable records (try to delete → fail)

---

## **Step 10 — Print / Export Test**

1. Go to `/reports`
2. Export CSV → Compare totals with `/transactions/` table
3. Verify print-friendly layout

---

## ✅ **Lab Completion Checklist**

* [ ] Backend migrated & seeded
* [ ] Frontend connected & routes working
* [ ] Distribution & Transactions workflow tested
* [ ] Load test scripts executed successfully
* [ ] Offline queue simulation tested
* [ ] Audit logs verified for integrity
* [ ] Reports printed & exported successfully
* [ ] Docker Compose full stack validated

---

This completes a **full-stack, enterprise-grade lab for HSH Sales System**.


