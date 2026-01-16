# 🏗️ HSH SALES SYSTEM — TECHNICAL PROPOSAL & ARCHITECTURE (v3.1)

---

## 1️⃣ Project Overview

**Purpose:**
An enterprise-grade LPG sales & logistics platform designed to optimize:

* **Field Operations:** Real-time order capture (online/offline), cylinder distribution, empty returns
* **Inventory Management:** Depot-wise tracking of full/empty cylinders
* **Transactions & Billing:** Sales, meter readings, customer invoicing
* **Financial Documents:** PDF invoices, receipts, and automated email dispatch
* **Audit & Compliance:** Traceable logs for all user actions
* **Reporting:** Filterable, exportable transaction history with customer-level detail

**Objectives:**

1. Decoupled architecture: React SPA frontend + Django REST Framework backend
2. Offline-first capabilities with auto-sync queue
3. Role-Based Access Control (RBAC) + detailed audit logging
4. PDF invoice generation & automated email dispatch
5. Dockerized, cloud-ready deployment

---

## 2️⃣ Technology Stack

| Layer            | Technology / Tool                        | Purpose / Benefit                                    |
| ---------------- | ---------------------------------------- | ---------------------------------------------------- |
| Frontend         | React SPA + React Router v7, TailwindCSS | Mobile-first, offline-capable UI                     |
| Backend          | Django REST Framework (Python 3.11)      | REST API, validation, business logic                 |
| Database         | MySQL 8.0                                | ACID transactions, relational integrity, scalability |
| Authentication   | JWT (SimpleJWT)                          | Token-based RBAC, secure sessions                    |
| PDF Generation   | WeasyPrint                               | Server-side HTML → PDF rendering                     |
| Email Dispatch   | Django EmailMessage / SMTP               | Automated invoice/email delivery                     |
| Offline Support  | LocalStorage + auto-sync                 | Field resilience during network downtime             |
| Printing         | ESC/POS via Web Bluetooth                | Thermal receipts for field sales                     |
| Containerization | Docker + Docker Compose                  | Cloud-ready deployment                               |
| Reporting        | DRF + WeasyPrint PDF / Excel             | Exportable, filterable reports                       |

---

## 3️⃣ System Domains

### 3.1 Authentication & RBAC

* JWT token-based sessions for field & admin users
* Role-based access: Admin / Sales / Supervisor
* Audit logs for sensitive actions (e.g., transaction creation, price updates)

### 3.2 Master Data

* Users, Customers, Cylinders/Equipment, Depots
* Each entity managed via REST API endpoints with full CRUD

### 3.3 Operations

* Cylinder distribution & empty returns
* Sales transactions with inventory deduction
* Meter readings & automatic calculation of usage
* Atomic updates: ensure inventory cannot go negative

### 3.4 Financial Documents

* PDF invoice generation from HTML templates
* Email dispatch for invoices & receipts
* Payment status tracking (paid/unpaid)

### 3.5 Reporting

* Customer-level, date-filtered sales reports
* Exportable to PDF and Excel
* Backend aggregation with DRF serializers

### 3.6 Audit & Logging

* All CRUD actions recorded in `audit_logs`
* Includes user ID, entity, action type, and timestamp
* Supports compliance and traceability

---

## 4️⃣ Backend Architecture (v3.1)

```
backend/
├── config/           # Django settings, URLs, WSGI/ASGI
├── apps/
│   ├── accounts/     # Users, JWT Auth, RBAC
│   ├── customers/    # Customer master data & pricing
│   ├── inventory/    # Cylinders, depots, stock tracking
│   ├── distribution/ # Collection & return workflow
│   ├── transactions/ # Sales & meter readings
│   ├── billing/      # PDF & Email service
│   ├── reports/      # Filtered/exportable reports
│   └── audit/        # Action logging
├── shared/
│   ├── permissions/  # Role-based access decorators
│   ├── serializers/  # DRF serializers for all models
│   └── utils/        # Helpers: PDF/email, validation, sync
└── manage.py
```

**Key Principles:**

* Service-layer separation
* Serializer-level validation
* Atomic transactions for inventory & sales
* Soft deletes with audit logging
* Automatic PDF/email generation

---

## 5️⃣ Frontend Architecture

```
frontend/
├── src/
│   ├── api/          # Axios/fetch services
│   ├── auth/         # Login, JWT management, route guards
│   ├── components/   # Reusable UI (tables, modals, forms)
│   ├── pages/        # Customers, Transactions, Delivery, Reports, Login
│   ├── hooks/        # Printer, offline sync, loaders/actions
│   ├── utils/        # Validators, formatters
│   └── App.jsx       # Root SPA + Router
```

**Principles:**

* Controlled forms & dynamic fields
* Real-time calculations (totals, quantities)
* Offline queue handling via LocalStorage
* Role-based routing & UI access
* PDF preview & download capability
* Loader/action patterns (React Router v7) for fetch/post

---

## 6️⃣ Core Database Models (MySQL v3.1)

### Users

```sql
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  username VARCHAR(50) UNIQUE,
  email VARCHAR(100) UNIQUE,
  password VARCHAR(128),
  role ENUM('admin','sales','supervisor'),
  vehicle_no VARCHAR(20),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### Customers

```sql
CREATE TABLE customers (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  payment_type ENUM('cash','monthly'),
  rate_14kg DECIMAL(10,2),
  rate_50kg DECIMAL(10,2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### Inventory

```sql
CREATE TABLE inventory (
  id INT AUTO_INCREMENT PRIMARY KEY,
  equipment VARCHAR(50),
  depot VARCHAR(50),
  full_qty INT,
  empty_qty INT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### Transactions

```sql
CREATE TABLE transactions (
  id INT AUTO_INCREMENT PRIMARY KEY,
  customer_id INT,
  user_id INT,
  qty_14 INT,
  qty_50 INT,
  total_amount DECIMAL(10,2),
  is_paid BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (customer_id) REFERENCES customers(id),
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```

### Distributions

```sql
CREATE TABLE distributions (
  id INT AUTO_INCREMENT PRIMARY KEY,
  distribution_no VARCHAR(20),
  depot VARCHAR(50),
  equipment VARCHAR(50),
  quantity INT,
  status ENUM('collection','empty_return'),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### Audit Logs

```sql
CREATE TABLE audit_logs (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT,
  action VARCHAR(50),
  entity VARCHAR(50),
  timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```

### Offline Transactions (LocalStorage Queue Mirror)

```sql
CREATE TABLE offline_transactions (
  id INT AUTO_INCREMENT PRIMARY KEY,
  payload JSON,
  status ENUM('pending','sent','failed'),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

---

## 7️⃣ ERD — Simplified (ASCII)

```
┌──────────┐        ┌────────────┐        ┌─────────────┐
│  users   │        │ customers  │        │ inventory   │
│──────────│        │────────────│        │─────────────│
│ id PK    │◄───────│ id PK      │        │ id PK       │
│ username │        │ name       │        │ equipment   │
│ email    │        │ payment_type│       │ depot       │
│ password │        │ rate_14kg  │       │ full_qty    │
│ role     │        │ rate_50kg  │       │ empty_qty   │
│ vehicle_no│       │ created_at │       │ created_at  │
└──────────┘        └────────────┘       └─────────────┘
       ▲                     ▲
       │                     │
       │                     │
       │                     │
┌───────────────┐             │
│ transactions  │             │
│───────────────│             │
│ id PK         │             │
│ customer_id FK│─────────────┘
│ user_id FK    │
│ qty_14        │
│ qty_50        │
│ total_amount  │
│ is_paid       │
│ created_at    │
└───────────────┘
       │
       ▼
┌───────────────┐
│ distributions │
│───────────────│
│ id PK         │
│ distribution_no│
│ depot         │
│ equipment     │
│ quantity      │
│ status        │
│ created_at    │
└───────────────┘

┌───────────────┐
│ audit_logs    │
│───────────────│
│ id PK         │
│ user_id FK    │
│ action        │
│ entity        │
│ timestamp     │
└───────────────┘

┌────────────────────┐
│ offline_transactions│
│────────────────────│
│ id PK              │
│ payload JSON       │
│ status             │
│ created_at         │
└────────────────────┘
```

---

## 8️⃣ Offline & Printing Workflow

* **Offline Queue:** Frontend saves unsynced transactions/distributions in LocalStorage (`transaction_buffer` / `distribution_buffer`)
* **Auto-sync:** When network restored, loader/action posts queued JSON payloads to backend
* **Thermal Printing:** `usePrinter` hook handles ESC/POS printing
* **PDF Invoice:** Generated server-side via WeasyPrint
* **Email Dispatch:** Backend sends invoice automatically post-transaction

---

## 9️⃣ Deployment Architecture

```
┌─────────┐   ┌─────────┐   ┌─────────────┐
│Frontend │   │Backend  │   │ MySQL DB    │
│5173     │   │8000     │   │3306         │
│Vol:/app │   │Vol:/app │   │Vol:mysql_data
└─────────┘   └─────────┘   └─────────────┘
```

* Docker Compose orchestration
* Persistent MySQL volumes
* `.env` for secrets and credentials
* Cloud-ready for AWS/GCP/Azure

**Quick Start:**

```bash
docker-compose up --build
docker exec -it hsh-sales-system_backend_1 bash
python manage.py createsuperuser
```

---

## 🔟 Proposal Summary

| Package                | Scope                                        | Cost (S$) |
| ---------------------- | -------------------------------------------- | --------- |
| Stand-alone System     | React SPA + DRF + MySQL + PDF + Email + RBAC | 16,000    |
| QuickBooks Integration | Inventory & Invoice Sync                     | 13,800    |
| Cloud Hosting          | Dockerized AWS/GCP/Azure                     | TBD       |

**Outcome:** Fully decoupled, scalable, offline-first system with audit logs, PDF/email automation, printing, and cloud deployment.

