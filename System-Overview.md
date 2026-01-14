# 🏗️ HSH SALES SYSTEM — CHEAT SHEET (MySQL Edition)

## 1️⃣ Project Structure (Essential)

```
hsh-sales-system/
├── backend/       # Django REST API + business logic
│   ├── accounts/
│   ├── customers/
│   ├── inventory/
│   ├── distribution/
│   ├── transactions/
│   ├── billing/
│   ├── reports/
│   ├── audit/
│   └── manage.py
├── frontend/      # React SPA (Vite)
│   ├── routes/
│   ├── hooks/
│   ├── services/
│   └── root.jsx
├── docker-compose.yml
└── README.md
```

---

## 2️⃣ Module Responsibilities

| Module           | Responsibilities                                  |
| ---------------- | ------------------------------------------------- |
| **Accounts**     | JWT Auth, RBAC, custom user model                 |
| **Customers**    | Customer info, payment types, pricing tiers       |
| **Inventory**    | Cylinder stock tracking (full/empty)              |
| **Distribution** | Collection & empty return logs, inventory updates |
| **Transactions** | Process sales, calculate totals, deduct inventory |
| **Billing**      | PDF invoice generation, email dispatch            |
| **Reports**      | Transaction history, filters (admin-only)         |
| **Audit**        | Logs user actions for compliance                  |
| **Frontend**     | Mobile-first UI, offline queue, thermal printing  |

---

## 3️⃣ System Architecture — ASCII Diagram

```
           ┌─────────────────────┐
           │     Frontend        │
           │  React SPA (Vite)   │
           │  Mobile-first UI    │
           │  Offline Queue      │
           │  Bluetooth Printing │
           └─────────┬──────────┘
                     │ HTTPS / REST API
                     ▼
           ┌─────────────────────┐
           │      Backend        │
           │ Django REST Framework│
           │ Business Logic      │
           │ JWT Auth / RBAC     │
           │ PDF / Email Service │
           │ Audit Logging       │
           └─────────┬──────────┘
                     │ SQL Queries
                     ▼
           ┌─────────────────────┐
           │       MySQL         │
           │  ACID-compliant DB  │
           │  Inventory / Users  │
           │  Transactions       │
           └─────────────────────┘
```

---

## 4️⃣ Database ERD — ASCII

```
User 1 ─── * Distribution
User 1 ─── * Transaction
Customer 1 ─── * Transaction
Inventory → Tracks full_qty / empty_qty
AuditLog → Logs actions by User
```

---

## 5️⃣ Data Flow — One Transaction

```
User (Sales)
   │
   ▼
Frontend SPA
   │ Save offline if no network
   ▼
LocalStorage Queue
   │ Network available? → Sync
   ▼
Backend API
   │ Validate user & permissions
   │ Update Inventory
   │ Create Transaction
   │ Generate PDF Invoice
   │ Send Email
   │ Log Audit Entry
   ▼
MySQL DB
   │ Commit transaction → Inventory updated → Audit logged
   ▼
Frontend SPA
   │ Display confirmation / Print receipt
```

---

## 6️⃣ Docker Deployment Overview

```
┌─────────┐   ┌─────────┐   ┌─────────┐
│frontend │   │backend  │   │db       │
└─────────┘   └─────────┘   └─────────┘
Ports: 5173   Ports: 8000   Ports: 3306
Volumes:
./frontend    ./backend     mysql_data
:/app/frontend:/app/backend /var/lib/mysql
```

---

## 7️⃣ Quick Commands

```bash
# Build & run containers
docker-compose up --build

# Create Django superuser
docker exec -it hsh-sales-system_backend_1 bash
python manage.py createsuperuser

# Backend API: http://localhost:8000
# Frontend: http://localhost:5173
```

---

## 8️⃣ Key Features — At a Glance

* Decoupled frontend/backend
* Offline-first + auto-sync
* Bluetooth thermal printing (ESC/POS)
* PDF invoice generation & email automation
* Audit logging
* Role-based access control (Admin/Sales)
* Dockerized deployment with MySQL persistence


