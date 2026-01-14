# 🏗️ HSH SALES SYSTEM — TECHNICAL PROPOSAL & ARCHITECTURE (v3.0)

## 1️⃣ Overview

**Purpose:** Enterprise LPG sales & logistics platform to streamline:

* **Field Operations:** Online/offline order capture, cylinder distribution, returns
* **Inventory:** Depot-wise full/empty cylinder tracking
* **Transactions:** Sales, meter readings, billing
* **Financial Docs:** PDF invoices, receipts, email dispatch
* **Audit & Compliance:** Traceable logs of all user actions
* **Reporting:** Filterable, exportable transaction history

**Objectives:**

1. Decoupled, scalable React SPA + DRF backend
2. Offline-first capability for field personnel
3. Role-based access control & audit logging
4. PDF invoice generation & email automation
5. Dockerized, cloud-ready deployment

---

## 2️⃣ Technology Stack

| Layer            | Tech / Tool                        | Benefit                               |
| ---------------- | ---------------------------------- | ------------------------------------- |
| Frontend         | React SPA + RR7, TailwindCSS       | Mobile-first, offline-capable UI      |
| Backend          | Django REST Framework, Python 3.11 | REST API, validations, business logic |
| Database         | MySQL 8.0                          | ACID transactions, reliability        |
| Authentication   | JWT (SimpleJWT)                    | Token-based RBAC, secure sessions     |
| PDF Generation   | WeasyPrint                         | Server-side HTML → PDF rendering      |
| Emailing         | Django SMTP / EmailMessage         | Automated invoice dispatch            |
| Offline Support  | LocalStorage + auto-sync           | Field resilience                      |
| Printing         | ESC/POS via Web Bluetooth          | Thermal receipts                      |
| Containerization | Docker + Docker Compose            | Cloud-ready, portable deployment      |
| Reporting        | DRF + WeasyPrint PDF               | Exportable, filterable reports        |

---

## 3️⃣ System Domains

### 3.1 Auth & RBAC

* Login / logout
* Role-based access (Admin vs Sales)
* JWT token-based sessions
* Audit logs for sensitive actions

### 3.2 Master Data

* Users, Customers, Equipment/Cylinders, Depots

### 3.3 Operations

* Cylinder distribution & returns
* Transaction capture & inventory deduction
* Meter readings & usage calculations
* Atomic inventory updates

### 3.4 Financial Documents

* PDF invoice generation
* Email dispatch to customers
* Payment status tracking

### 3.5 Reporting

* Customer-specific sales reports
* Filtered transaction history
* PDF/Excel export

### 3.6 Audit & Logging

* User actions (CRUD)
* Rate/price changes
* Transaction and billing logs

---

## 4️⃣ Backend Architecture

```
backend/
├── config/           # settings, URLs, WSGI
├── apps/
│   ├── accounts/     # Users, JWT, RBAC
│   ├── customers/    # Customer master data
│   ├── inventory/    # Cylinders & depots
│   ├── distribution/ # Collection/return workflow
│   ├── transactions/ # Sales, meter readings
│   ├── billing/      # PDF + Email service
│   ├── reports/      # Transaction reports
│   └── audit/        # Action logging
├── shared/
│   ├── permissions/  # Role-based access
│   ├── serializers/  # DRF serializers
│   └── utils/        # Helpers
└── manage.py
```

**Principles:** Service layer separation, serializer validation, atomic transactions, PDF/email automation, soft deletes with audit logging.

---

## 5️⃣ Frontend Architecture

```
frontend/
├── src/
│   ├── api/         # Fetch/axios services
│   ├── auth/        # Login, route guards
│   ├── components/  # Reusable UI
│   ├── pages/       # Distribution, Transactions, Customers, Reports, Login
│   ├── hooks/       # Printer, offline sync
│   ├── utils/       # Helpers & validators
│   └── App.jsx
```

**Principles:** Controlled forms, dynamic items, real-time calculations, PDF preview/download, role-based routing, search/autocomplete.

---

## 6️⃣ Core Database Models

```python
class Customer(models.Model):
    name, customer_id, payment_type
    rates: meter, cyl_9, cyl_12.7, cyl_14, cyl_50

class Transaction(models.Model):
    transaction_id, user, customer, total_amount
    payment_status, timestamp

class MeterReading(models.Model):
    transaction (1:1)
    last_reading, latest_reading, quantity
```

**ERD (simplified):**

```
User 1---* Transaction *---1 Customer
 |
 1
 *
MeterReading
AuditLog: FK -> User
```

---

## 7️⃣ Key Features

### PDF & Email Workflow

* HTML template → WeasyPrint PDF
* Automated email dispatch via Django EmailMessage
* Example endpoint: `POST /api/email_invoice/{transaction_id}`

### QuickBooks Integration (Optional)

* Background Celery task
* Sync transactions to QuickBooks Online via API
* Update local `qb_synced` flag

---

## 8️⃣ Offline & Printing

* **Offline Queue:** LocalStorage saves unsynced transactions
* **Auto-sync:** On network restore
* **Thermal Printing:** `usePrinter` hook + ESC/POS
* **80mm width** printing with CSS media queries
* **Preview & Download** using FileSaver.js

---

## 9️⃣ Deployment Architecture

```
┌─────────┐   ┌─────────┐   ┌─────────────┐
│Frontend │   │Backend  │   │ DB (MySQL)  │
│5173     │   │8000     │   │3306         │
│Vol:/app │   │Vol:/app │   │Vol:mysql_data
└─────────┘   └─────────┘   └─────────────┘
```

* Docker Compose for orchestration
* `.env` for credentials & secrets
* Persistent MySQL volumes
* Cloud-ready (AWS/GCP/Azure)

**Quick Start:**

```bash
docker-compose up --build
docker exec -it hsh-sales-system_backend_1 bash
python manage.py createsuperuser
```

---

## 🔟 Proposal Summary

| Package                | Scope                                    | Cost (S$) |
| ---------------------- | ---------------------------------------- | --------- |
| Stand-alone System     | React + DRF + MySQL + PDF + Email + RBAC | 16,000    |
| QuickBooks Integration | Inventory + Invoice Sync                 | 13,800    |
| Cloud Hosting          | Docker-based AWS/GCP/Azure               | TBD       |

**Outcome:** Fully decoupled, scalable, offline-first system with audit logs, PDF/email automation, printing, and cloud deployment.

