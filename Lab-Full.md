# 🧪 HSH Sales System — Full Stack Lab Guide (FINAL)

**Objective:** Deploy and validate a **production-grade full-stack HSH Sales System**
(**Django REST Framework backend + React 18 + React Router v7 frontend**), then verify:

* Inventory safety
* Meter correctness
* Offline tolerance
* Audit immutability
* Report accuracy under load

---

## 0️⃣ Prerequisites

* Docker & Docker Compose
* Node.js 20+
* Python 3.11+
* Postman / curl
* Recommended: VSCode + Docker extension

---

## 1️⃣ Project Structure (Canonical)

```
hsh_sales_system/
├── backend/
│   ├── accounts/
│   ├── customers/
│   ├── depots/
│   ├── inventory/
│   ├── distribution/
│   ├── transactions/
│   ├── invoices/
│   ├── audit/
│   ├── reports/
│   └── config/
├── frontend/
│   ├── src/
│   ├── index.html
│   └── package.json
├── docker-compose.yml
└── README.md
```

**Rule:**
Frontend and backend are **isolated**, communicating only over `/api`.

---

## 2️⃣ Docker Compose (Aligned with Backend)

### `docker-compose.yml`

```yaml
version: "3.9"

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: hsh_sales
      POSTGRES_USER: hsh
      POSTGRES_PASSWORD: hshpass
    volumes:
      - db_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  backend:
    build: ./backend
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./backend:/app
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgres://hsh:hshpass@db:5432/hsh_sales
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

✔ Matches backend assumptions
✔ Transaction-safe DB
✔ Network-isolated services

---

## 3️⃣ Backend Setup & Validation

### 3.1 Install & migrate

```bash
cd backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

python manage.py migrate
python manage.py createsuperuser
```

### 3.2 Verify core endpoints

```bash
curl http://localhost:8000/api/customers/customers/
curl http://localhost:8000/swagger/
```

✔ Swagger available
✔ API reachable

---

## 4️⃣ Frontend Setup

```bash
cd frontend
npm install
npm run dev
```

Visit:
👉 `http://localhost:5173`

---

## 5️⃣ Core Workflow Validation

### 5.1 Authentication

1. Login via `/login`
2. JWT stored in `localStorage`
3. Protected routes load via RR7 loader

✔ Frontend guarded
✔ Backend authoritative

---

### 5.2 Distribution (Stock Movement)

1. Go to `/distribution`
2. Submit:

   * depot
   * equipment_name
   * quantity
   * status (`COLLECTION` / `EMPTY_RETURN`)
3. Backend endpoint hit:

```
POST /api/distribution/distributions/create/
```

4. Verify:

   * Inventory adjusted atomically
   * Audit log written

✔ No direct inventory mutation
✔ Service-enforced invariants

---

### 5.3 Transactions (Billing Core)

1. Go to `/transaction`
2. Enter:

   * Meter readings
   * Cylinder quantities
   * Service items
3. Submit → backend command:

```
POST /api/transactions/transactions/create/
```

4. Verify:

   * Transaction committed
   * Meter reading updated
   * Audit entry written
   * Invoice eligible for issuance

✔ Meter-safe
✔ Snapshot pricing
✔ No frontend authority

---

### 5.4 Invoice Issuance

```
POST /api/transactions/{transaction_no}/issue-invoice/
GET  /api/invoices/{invoice_no}/
```

Verify:

* Sequential invoice numbers
* GST preserved
* Immutable totals

---

## 6️⃣ Seed & Load Test Data

### 6.1 Seed script (one-time)

**backend/scripts/seed_data.py**

```python
from customers.models import Customer
from depots.models import Depot
from inventory.models import Inventory
import random

for i in range(3):
    depot, _ = Depot.objects.get_or_create(code=f"DPT{i+1}", name=f"Depot {i+1}")
    for eq in ["9kg", "12.7kg", "14kg", "50kg_pol", "50kg_l"]:
        Inventory.objects.get_or_create(
            depot=depot,
            equipment_name=eq,
            defaults={"quantity": 200}
        )

for i in range(20):
    Customer.objects.get_or_create(
        name=f"Customer {i+1}",
        address="Test Address",
        payment_type=random.choice(["CASH", "CREDIT"])
    )
```

Run:

```bash
python manage.py shell < scripts/seed_data.py
```

---

## 7️⃣ Transaction Load Test

```python
import requests, random

API = "http://localhost:8000/api/transactions/transactions/create/"
TOKEN = "JWT_TOKEN_HERE"

headers = {"Authorization": f"Bearer {TOKEN}"}

for i in range(50):
    payload = {
        "customer": random.randint(1,20),
        "total_amount": random.randint(100,500)
    }
    r = requests.post(API, json=payload, headers=headers)
    print(r.status_code)
```

Verify:

* Transactions created
* Audit log populated
* No partial commits

---

## 8️⃣ Offline Queue Simulation

1. Disconnect network
2. Submit transaction or distribution
3. Confirm saved to `localStorage`
4. Reconnect → auto flush

✔ Backend rejects invalid state
✔ No silent data loss

---

## 9️⃣ Audit Integrity Verification

Visit:

```
GET /api/audit/audit-logs/
```

Confirm:

* Append-only
* Server timestamps
* User attribution
* No delete/edit routes

---

## 🔟 Reports & Print Validation

Endpoints:

```
GET /api/reports/sales/?from=&to=
GET /api/reports/inventory/?depot=
```

Validate:

* Totals match transactions
* Print CSS applies cleanly
* CSV/PDF export consistent

---

## ✅ FINAL LAB CHECKLIST

* [ ] Docker stack running
* [ ] Swagger reachable
* [ ] JWT auth enforced
* [ ] Distribution inventory-safe
* [ ] Transactions meter-safe
* [ ] Invoices GST-correct
* [ ] Offline queue tested
* [ ] Audit immutable
* [ ] Reports reconcile totals

---

## 🎯 Result

You now have a **fully validated, regulator-ready full-stack LPG sales system** with:

* Explicit command boundaries
* Structural safety (not conventions)
* Offline tolerance
* Audit-grade traceability
* Frontend & backend in strict alignment

