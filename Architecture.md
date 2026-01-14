# 🏗️ HSH SALES SYSTEM

## Design & Architecture Document (Refined Edition)

---

## 1️⃣ Executive Overview

The **HSH Sales System** is a **full-stack LPG sales, delivery, and logistics platform** designed to support **field operations, inventory integrity, billing accuracy, and regulatory auditability**.

The system is built with an **offline-first, security-aware architecture**, ensuring uninterrupted operations in low-connectivity environments while maintaining strict backend authority over inventory, pricing, and transactions.

### Core Operational Domains

* **Field Operations**

  * Online/offline transaction capture
  * Delivery and empty-return batch handling
  * Immediate receipt printing

* **Inventory Management**

  * Real-time full/empty cylinder tracking
  * Server-enforced consistency rules

* **Transaction & Billing**

  * Sales processing
  * Automated invoice generation
  * Email dispatch

* **Audit & Compliance**

  * Immutable action logging
  * Role-based access enforcement

* **Reporting & Analytics**

  * Filterable transaction history
  * Exportable administrative reports

---

## 2️⃣ Technology Stack

| Layer            | Technology                              | Architectural Role                   |
| ---------------- | --------------------------------------- | ------------------------------------ |
| Frontend         | React SPA (Vite), JSX, TailwindCSS      | Mobile-first UI, offline-capable     |
| Routing          | React Router v7 (Data / Framework Mode) | Declarative data loading & mutations |
| Backend          | Django REST Framework (Python 3.11)     | API, business logic, security        |
| Database         | MySQL 8.0                               | ACID-compliant persistence           |
| Authentication   | JWT (SimpleJWT) + RBAC                  | Trust boundary enforcement           |
| Offline Support  | LocalStorage queue + auto-sync          | Field resiliency                     |
| Printing         | ESC/POS via Web Bluetooth               | Thermal receipts                     |
| Reporting        | ReportLab (PDF)                         | Invoices & reports                   |
| Containerization | Docker + Docker Compose                 | Portable deployment                  |

---

## 3️⃣ Design Principles & Goals

1. **Strict frontend/backend separation** – UI never owns business truth
2. **Offline-first reliability** – Field operations continue regardless of connectivity
3. **Backend-owned invariants** – Inventory, pricing, and numbering are server-controlled
4. **Auditability by design** – All critical actions are logged
5. **Role-based access control** – Clear Admin vs Sales responsibility boundaries
6. **Operational portability** – Dockerized, environment-driven deployment

---

## 4️⃣ System Architecture

### 4.1 Logical Architecture

```
Frontend (React SPA)
 ├─ Mobile-first UI
 ├─ React Router loaders & actions
 ├─ Offline queue (LocalStorage)
 └─ Bluetooth thermal printing

Backend (Django REST Framework)
 ├─ JWT authentication + RBAC
 ├─ Domain services (Transactions, Distribution, Billing)
 ├─ Audit logging
 └─ PDF generation & email delivery

Database (MySQL)
 ├─ Users, Customers, Inventory
 ├─ Transactions, Distributions
 └─ AuditLog
```

### 4.2 Container Architecture

```
┌─────────┐   ┌─────────┐   ┌─────────────┐
│Frontend │   │Backend  │   │MySQL        │
│:5173    │   │:8000    │   │:3306        │
│Vol:/app │   │Vol:/app │   │Vol:mysql_data
└─────────┘   └─────────┘   └─────────────┘
```

---

## 5️⃣ Domain Modules & Responsibilities

| Module           | Responsibility                                      |
| ---------------- | --------------------------------------------------- |
| **Accounts**     | User model, JWT auth, RBAC                          |
| **Customers**    | Customer profiles, pricing tiers, payment types     |
| **Inventory**    | Full/empty cylinder tracking and enforcement        |
| **Distribution** | Delivery batches, empty returns, inventory movement |
| **Transactions** | Sales creation, totals calculation, stock deduction |
| **Billing**      | PDF invoice generation and email dispatch           |
| **Reports**      | Filtered transaction history and admin reports      |
| **Audit**        | Immutable logging of critical user actions          |
| **Frontend**     | Mobile UI, offline queue, printing                  |

---

## 6️⃣ Data Model Overview

### 6.1 Core Entities

| Entity           | Key Fields                                        | Notes             |
| ---------------- | ------------------------------------------------- | ----------------- |
| **User**         | username, role, vehicle_no                        | Sales/Admin users |
| **Customer**     | name, payment_type, rate_14kg, rate_50kg          | Pricing authority |
| **Inventory**    | equipment, full_qty, empty_qty                    | Stock truth       |
| **Distribution** | distribution_no, user, equipment, qty, status     | Delivery batches  |
| **Transaction**  | customer, user, total_amount, is_paid, created_at | Sales records     |
| **AuditLog**     | user, action, created_at                          | Compliance trail  |

### 6.2 ERD (ASCII)

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
(User FK)
```

---

## 7️⃣ Backend Architecture

### 7.1 Configuration

* Environment-based settings (`.env`)
* MySQL with `STRICT_TRANS_TABLES`
* JWT authentication (SimpleJWT)
* CORS enabled for SPA
* Modular Django apps per domain

### 7.2 Domain Services

| Service                 | Responsibility                       |
| ----------------------- | ------------------------------------ |
| **TransactionService**  | Pricing, totals, inventory deduction |
| **DistributionService** | Batch creation, inventory movement   |
| **BillingService**      | PDF generation and email             |
| **ReportService**       | Filtered reporting                   |
| **AuditService**        | Action logging                       |

### 7.3 Security Model

* JWT authentication
* Role-based permissions at view level
* Atomic inventory updates
* Server-generated identifiers
* Audit logging on sensitive mutations

---

## 8️⃣ Frontend Architecture

* React SPA (Vite)
* React Router v7 (loaders/actions)
* TailwindCSS (mobile-first)
* `offline.js` — LocalStorage queue
* `usePrinter.js` — ESC/POS printing
* Root layout manages routing, auth state, and layout shell

**Offline strategy:**
Mutations are queued locally and replayed automatically when connectivity is restored.

---

## 9️⃣ Transaction Data Flow (High Level)

```
User
 ↓
React UI
 ↓
Router Action
 ↓
[Offline? → Local Queue]
 ↓
Backend API
 ↓
Domain Service
 ↓
MySQL (atomic commit)
 ↓
AuditLog
 ↓
Frontend confirmation + optional printing
```

---

## 🔟 Deployment Model

* Docker Compose orchestrates:

  * `frontend` (5173)
  * `backend` (8000)
  * `db` (3306)
* Persistent volumes for MySQL
* Secrets managed via `.env`

**Bootstrap:**

```bash
docker-compose up --build
docker exec -it backend bash
python manage.py createsuperuser
```

---

## 1️⃣1️⃣ Offline & Printing Strategy

* Local queue for unsynced writes
* Automatic replay on reconnect
* Immediate ESC/POS receipt printing
* Printing isolated from persistence logic

---

## 1️⃣2️⃣ Key System Characteristics

* Decoupled architecture
* Offline-first resilience
* Backend-owned business truth
* Thermal printing support
* Automated billing
* Audit-ready compliance
* ACID-safe inventory & transactions

---

# 🔁 Sequence Diagrams — Core Business Flows

---

## 1️⃣ Transaction Creation Flow

```
User
 │
 ▼
React UI
 │
 ▼
Router Action
 │
 ├─ Offline → Queue locally
 │
 └─ Online → POST /api/transactions/
        │
        ▼
   DRF View
        │
        ▼
TransactionService
        │
        ▼
MySQL (atomic commit)
        │
        ▼
AuditLog
        │
        ▼
Response JSON
 │
 ▼
UI Update → Optional Print
```

**Notes:**

* All mutations flow through router actions
* Pricing and inventory logic is backend-owned
* Printing is post-commit and non-blocking

---

## 2️⃣ Delivery / Distribution Batch Flow

```
User
 │
 ▼
React UI
 │
 ▼
Router Action
 │
 ├─ Offline → Save batch locally
 │
 └─ Online → POST /api/distributions/
        │
        ▼
DistributionService
        │
        ├─ Generate distribution_no
        ├─ Apply inventory movement
        ▼
MySQL (atomic commit)
        │
        ▼
AuditLog
 │
 ▼
UI Confirmation → Print (optional)
```

**Notes:**

* Batch operations are atomic
* Inventory changes are server-enforced
* Printing never blocks persistence

---

## 3️⃣ Failure & Recovery Pattern

```
Network / API Failure
        ↓
Router Action catches error
        ↓
Offline queue persists payload
        ↓
UI indicates "Saved Offline"
        ↓
Network restored
        ↓
Queued mutations replayed
        ↓
Backend commits & audits
```

---

## 4️⃣ Architectural Strengths

* Clear trust boundaries
* No UI-owned invariants
* Offline-safe by construction
* Side effects isolated
* Highly testable flows
* Compliance-ready audit trail

---

# 🏗️ HSH SALES SYSTEM — Full-Stack Architecture Map

```
                  ┌───────────────────────┐
                  │        User           │
                  │  (Sales / Admin)      │
                  └─────────┬─────────────┘
                            │
                            ▼
            ┌─────────────────────────────────┐
            │       React SPA (Vite)          │
            │ Mobile-first UI + TailwindCSS  │
            ├─────────────────────────────────┤
            │ Components:                     │
            │ ├─ MeterSection                 │
            │ ├─ CylinderSection              │
            │ ├─ ServiceSection               │
            │ └─ SummaryBar                   │
            ├─────────────────────────────────┤
            │ React Router v7                 │
            │ ├─ Loader → fetch data          │
            │ └─ Action → mutations           │
            └─────────┬───────────────────────┘
                      │
                      ▼
         ┌─────────────────────────────┐
         │ Offline Queue (LocalStorage) │
         │ Auto-sync on reconnect       │
         └─────────┬───────────────────┘
                   │
                   ▼
         ┌─────────────────────────────┐
         │   Django REST Framework      │
         │  (Backend API + Domain)     │
         ├─────────────────────────────┤
         │ Auth: JWT + RBAC            │
         │ Services:                   │
         │ ├─ TransactionService       │
         │ ├─ DistributionService      │
         │ ├─ BillingService           │
         │ ├─ ReportService            │
         │ └─ AuditService             │
         └─────────┬───────────────────┘
                   │
        ┌──────────┴───────────┐
        ▼                      ▼
┌───────────────┐       ┌───────────────┐
│ Inventory DB  │       │ Transactions  │
│ (Depot-scoped │       │ & Customer DB │
│ full/empty)   │       │               │
└───────────────┘       └───────────────┘
        │                      │
        └──────────┬───────────┘
                   ▼
             ┌───────────────┐
             │  AuditLog     │
             │ Immutable log │
             └───────────────┘
                   │
                   ▼
             ┌───────────────┐
             │ Reporting     │
             │ CSV / Excel   │
             │ PDF / Email   │
             └───────────────┘
                   │
                   ▼
             ┌───────────────┐
             │ Thermal Print │
             │ ESC/POS       │
             └───────────────┘
```

---

**Key Highlights:**

1. Frontend → Backend Flow: React SPA drives workflow UI, with loaders for data and actions for mutations. Offline queue ensures continuity.
2. Backend Services: Transaction, Distribution, Billing, Report, Audit — all atomic, audited, and backend-owned.
3. Database: Depot-scoped inventory ensures stock truth; transactions and customers maintain historical integrity.
4. Printing & Reporting: ESC/POS printing post-commit; flat CSV/PDF export for auditing.
5. Security: JWT + RBAC enforces Admin vs Sales separation; backend is single source of truth.

