# 🏗️ HSH SALES SYSTEM — DESIGN & ARCHITECTURE DOCUMENT 

## 1️⃣ Overview

The **HSH Sales System** is a **full-stack LPG logistics and sales platform** designed to streamline:

* **Field operations** — support online/offline order capture, delivery, and cylinder returns
* **Inventory management** — real-time tracking of full and empty cylinders
* **Transaction processing** — sales, billing, and automated invoicing
* **Audit & compliance** — action logging and traceability
* **Reporting & analytics** — filtering and exportable transaction history

**Technology Stack**:

| Layer            | Technology                                         | Notes                               |
| ---------------- | -------------------------------------------------- | ----------------------------------- |
| Frontend         | React SPA (Vite), JSX, TailwindCSS                 | Mobile-first, offline-capable       |
| Backend          | Django REST Framework, Python 3.11                 | Modular services, JWT/RBAC          |
| Database         | MySQL 8.0                                          | ACID-compliant, persistent          |
| Containerization | Docker, Docker Compose                             | Consistent environment, portability |
| Authentication   | JWT (Simple JWT), Role-based Access Control (RBAC) | Admin/Sales segregation             |
| Offline Support  | LocalStorage queue, auto-sync                      | Field resiliency                    |
| Printing         | ESC/POS via Web Bluetooth                          | Thermal receipt printing            |
| Reporting        | Django services + PDF (ReportLab)                  | Invoice & report generation         |

**Design Goals**:

1. **Decoupled frontend/backend** for scalability and maintainability
2. **Offline-first operations** to ensure continuity without connectivity
3. **Audit logging** for traceability and compliance
4. **Automated billing** with PDF/email dispatch
5. **Role-based access control** (Admin/Sales)
6. **Dockerized deployment** with MySQL persistence

---

## 2️⃣ System Architecture

### 2.1 Logical Architecture

```
Frontend (React SPA)
 ├─ Mobile-first UI
 ├─ Offline storage (LocalStorage queue)
 └─ Bluetooth thermal printing

Backend (Django REST Framework)
 ├─ JWT authentication + RBAC
 ├─ Business logic (Distribution, Transactions, Billing)
 ├─ Audit logging
 └─ PDF generation & email service

Database (MySQL)
 ├─ Users, Customers, Inventory, Transactions, AuditLog
 └─ ACID-compliant transactional storage
```

### 2.2 Container Architecture

```
┌─────────┐   ┌─────────┐   ┌─────────────┐
│Frontend │   │Backend  │   │DB (MySQL)   │
│5173     │   │8000     │   │3306         │
│Vol:/app │   │Vol:/app │   │Vol:mysql_data
└─────────┘   └─────────┘   └─────────────┘
```

---

## 3️⃣ Module Responsibilities

| Module           | Responsibilities                                              |
| ---------------- | ------------------------------------------------------------- |
| **Accounts**     | User model, JWT authentication, RBAC                          |
| **Customers**    | Customer information, payment types, pricing tiers            |
| **Inventory**    | Cylinder stock tracking (full/empty), consistency enforcement |
| **Distribution** | Collection & empty return logging, inventory updates          |
| **Transactions** | Order creation, total calculations, inventory deduction       |
| **Billing**      | PDF invoice generation, email dispatch                        |
| **Reports**      | Filtered transaction history and admin reports                |
| **Audit**        | Logs user actions for compliance                              |
| **Frontend**     | Mobile-first UI, offline queue, thermal printing              |

---

## 4️⃣ Data Model & ERD

### 4.1 Entity Descriptions

| Entity           | Key Fields                                                   | Notes                         |
| ---------------- | ------------------------------------------------------------ | ----------------------------- |
| **User**         | `username`, `role` (admin/sales), `vehicle_no`               | Core system user              |
| **Customer**     | `name`, `payment_type`, `rate_14kg`, `rate_50kg`             | Pricing and billing           |
| **Inventory**    | `equipment`, `full_qty`, `empty_qty`                         | Cylinder stock tracking       |
| **Distribution** | `distribution_no`, `user`, `equipment`, `quantity`, `status` | Collection/return operations  |
| **Transaction**  | `customer`, `user`, `total_amount`, `is_paid`, `created_at`  | Sales records                 |
| **AuditLog**     | `user`, `action`, `created_at`                               | Action logging for compliance |

---

### 4.2 ASCII ERD Diagram

```
+---------+        +-----------------+        +-------------+
|  User   |1------*|  Distribution   |*------1|  Inventory  |
+---------+        +-----------------+        +-------------+
      |
      |1
      * 
+-----------------+
|  Transaction    |*------1
+-----------------+
      |
      * 
+-----------------+
|   Customer      |
+-----------------+

+-----------------+
|   AuditLog      |
+-----------------+
Logs actions of User (FK: user_id)
```

> **Notes:**
>
> * `User` is linked to `Distribution` and `Transaction`.
> * `Customer` is linked to `Transaction`.
> * `Inventory` tracks equipment quantities updated via `Distribution` & `Transaction`.
> * `AuditLog` tracks all critical user actions.

---

## 5️⃣ Backend Architecture

### 5.1 Django Settings

* Environment-driven config (`.env`) for development, Docker, production
* MySQL connection with `STRICT_TRANS_TABLES` for ACID compliance
* JWT authentication using DRF SimpleJWT
* CORS enabled for SPA access
* Modular app structure for Accounts, Customers, Inventory, Distribution, Transactions, Billing, Reports, Audit

### 5.2 Services

| Service                 | Responsibilities                                      |
| ----------------------- | ----------------------------------------------------- |
| **DistributionService** | Creates distribution records, updates inventory       |
| **TransactionService**  | Processes sales, deducts inventory, calculates totals |
| **BillingService**      | Generates PDF invoice, sends email                    |
| **ReportService**       | Returns filtered transaction history                  |
| **AuditService**        | Logs user actions (create, update, delete)            |

### 5.3 Security & Permissions

* JWT-based authentication
* Role-based access (Admin / Sales)
* Permissions enforced at view level (`IsAdmin`, `IsSales`)
* Audit logging for sensitive actions
* Inventory updates atomic to prevent race conditions

---

## 6️⃣ Frontend Architecture

* React SPA (Vite) for performance
* Mobile-first layout with TailwindCSS
* Offline queue using LocalStorage
* Delivery Form: 14kg & 50kg quantities capture
* Printer Hook (`usePrinter.js`) — ESC/POS via Web Bluetooth
* Root layout (`root.jsx`) manages routing and containers

**Offline Workflow**: Unsynced transactions stored locally, automatically sync when network is available. Receipts can be printed anytime.

---

## 7️⃣ Data Flow Example — Transaction

```
User (Sales)
   │ Initiates transaction
   ▼
Frontend SPA
   │ If offline → save to LocalStorage
   ▼
LocalStorage Queue
   │ Network available? → send to Backend
   ▼
Backend API
   │ Validate JWT & RBAC
   │ Deduct inventory (atomic)
   │ Create Transaction record
   │ Generate PDF invoice
   │ Send email
   │ Log Audit entry
   ▼
MySQL DB
   │ Commit transaction → Inventory updated → Audit logged
   ▼
Frontend SPA
   │ Display confirmation, print receipt
```

---

## 8️⃣ Deployment Architecture

* Docker Compose orchestrates:

  * `backend` — Django, port 8000
  * `frontend` — React SPA, port 5173
  * `db` — MySQL, port 3306
* Volume persistence:

  * `mysql_data` → MySQL storage
  * Bind mounts for live code: `./backend`, `./frontend`
* `.env` contains secrets and DB credentials

**Quick Start**:

```bash
docker-compose up --build
docker exec -it hsh-sales-system_backend_1 bash
python manage.py createsuperuser
```

> **Production Notes**: Disable `DEBUG=True`, set strong `SECRET_KEY`, secure DB access.

---

## 9️⃣ Offline & Printing

* **Offline queue** — `offline.js` stores unsynced transactions
* **Auto-sync** — automatically syncs when network restored
* **Thermal printing** — `usePrinter.js` supports ESC/POS via Web Bluetooth
* Users can print receipts immediately, even while offline

---

## 🔟 Features & Highlights

* Decoupled frontend/backend
* Offline-first with automatic sync
* Thermal printing support (Bluetooth)
* PDF invoice generation and email dispatch
* Audit logging for compliance
* Role-based access control
* Dockerized deployment with persistent MySQL storage
* ACID-compliant transactions for inventory and sales integrity

---

## 1️⃣1️⃣ Diagrams

### 11.1 System Architecture (ASCII)

```
Frontend SPA ──HTTPS/REST──> Backend API ──SQL──> MySQL DB
      │                          │
      │                          ├─ TransactionService
      │                          ├─ DistributionService
      │                          ├─ BillingService
      │                          ├─ ReportService
      │                          └─ AuditService
      │
      └─ Offline queue & print
```

### 11.2 Container Layout

```
┌─────────┐   ┌─────────┐   ┌─────────────┐
│frontend │   │backend  │   │db (MySQL)   │
│5173     │   │8000     │   │3306         │
│Vol:/app │   │Vol:/app │   │Vol:mysql_data
└─────────┘   └─────────┘   └─────────────┘
```

### 11.3 Database ERD (ASCII)

```
+---------+        +-----------------+        +-------------+
|  User   |1------*|  Distribution   |*------1|  Inventory  |
+---------+        +-----------------+        +-------------+
      |
      |1
      * 
+-----------------+
|  Transaction    |*------1
+-----------------+
      |
      * 
+-----------------+
|   Customer      |
+-----------------+

+-----------------+
|   AuditLog      |
+-----------------+
Logs actions of User (FK: user_id)
```

---

✅ **This document covers**:

* Project structure and module responsibilities
* Backend & frontend architecture
* Data flow, offline handling, and printing
* Database model & ASCII ERDs
* Security, RBAC, and audit logging
* Deployment architecture with Docker
* Professional diagrams suitable for developer onboarding or stakeholder review

