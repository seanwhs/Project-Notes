# 🏗️ HSH SALES SYSTEM — DESIGN & ARCHITECTURE (v3.0)

---

## 1️⃣ Executive Overview

The **HSH Sales System** is a **full-stack LPG sales, delivery, and logistics platform** designed for **field operations, inventory integrity, billing accuracy, and regulatory auditability**.

Key principles:

* **Offline-first operation:** Field staff can continue sales and deliveries without network access.
* **Backend authority:** Inventory, pricing, and transaction totals are **always validated server-side**.
* **Auditability by design:** Every critical action is logged and immutable.
* **Security-first:** JWT authentication, role-based access, and explicit trust boundaries.

### Core Operational Domains

* **Field Operations**

  * Online/offline transaction capture
  * Delivery and empty-return batch handling
  * Immediate receipt printing

* **Inventory Management**

  * Real-time full/empty cylinder tracking
  * Depot-scoped stock management
  * Backend-enforced consistency rules

* **Transaction & Billing**

  * Meter, cylinder, service sales processing
  * Automated invoice generation (PDF + email)
  * Reconciliation and payment tracking

* **Audit & Compliance**

  * Immutable action logging
  * Role-based access enforcement

* **Reporting & Analytics**

  * Filterable transaction history
  * Exportable administrative reports (CSV/PDF)

---

## 2️⃣ Technology Stack

| Layer            | Technology                          | Architectural Role                |
| ---------------- | ----------------------------------- | --------------------------------- |
| Frontend         | React 18 (Vite), JSX, TailwindCSS   | Mobile-first UI, offline-capable  |
| Routing          | React Router v7 (Data APIs)         | Loader + Action-driven workflows  |
| Backend          | Django REST Framework (Python 3.11) | API, business logic, security     |
| Database         | MySQL 8.0                           | ACID-compliant persistence        |
| Authentication   | JWT (SimpleJWT) + RBAC              | Role-based trust enforcement      |
| Offline Support  | LocalStorage queue + auto-sync      | Field resiliency                  |
| Printing         | ESC/POS via Web Bluetooth           | Thermal receipt generation        |
| Reporting        | ReportLab (PDF), CSV/Excel exports  | Invoices & administrative reports |
| Containerization | Docker + Docker Compose             | Portable deployment               |

---

## 3️⃣ Design Principles

1. **Strict frontend/backend separation** – UI never owns business truth.
2. **Offline-first reliability** – Operations continue without connectivity.
3. **Backend-owned invariants** – Inventory, pricing, numbering validated server-side.
4. **Auditability by design** – All critical actions logged.
5. **Role-based access control (RBAC)** – Clear Admin vs Sales responsibilities.
6. **Operational portability** – Dockerized deployment, environment-driven configuration.

---

## 4️⃣ System Architecture

### 4.1 Logical Architecture

```
Frontend (React SPA)
 ├─ Mobile-first UI
 ├─ React Router loaders & actions
 ├─ Offline queue (LocalStorage)
 └─ Thermal ESC/POS printing

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

## 5️⃣ Domain Modules

| Module           | Responsibility                                       |
| ---------------- | ---------------------------------------------------- |
| **Accounts**     | User auth, JWT, RBAC                                 |
| **Customers**    | Profiles, pricing tiers, payment methods             |
| **Inventory**    | Full/empty cylinder stock, depot allocation          |
| **Distribution** | Delivery batches, empty returns, inventory movement  |
| **Transactions** | Sales creation, totals, stock deduction              |
| **Billing**      | PDF invoice generation and email dispatch            |
| **Reports**      | Filtered transaction history, administrative exports |
| **Audit**        | Immutable logging of critical actions                |
| **Frontend**     | Offline-first SPA, printing, routing                 |

---

## 6️⃣ Data Model Overview

### 6.1 Core Entities & Fields

| Entity           | Key Fields (with types)                                                                                                                                                                          | Notes              |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------ |
| **User**         | id (PK), username (str), role (enum: Admin/Sales/Supervisor), vehicle_no (str), email, created_at, updated_at                                                                                    | Admin/Sales users  |
| **Customer**     | id (PK), name (str), contact_no (str), address (str), payment_type (enum: Cash/Credit), rate_14kg (decimal), rate_50kg (decimal), active (bool), created_at, updated_at                          | Pricing authority  |
| **Inventory**    | id (PK), item_name (str), item_type (enum: Cylinder/Meter/Service), full_qty (int), empty_qty (int), depot_id (FK), last_updated                                                                 | Depot-scoped stock |
| **Distribution** | id (PK), distribution_no (str), user_id (FK), item_id (FK), quantity (int), status (enum: Pending/Completed/Cancelled), created_at, delivered_at                                                 | Delivery batches   |
| **Transaction**  | id (PK), customer_id (FK), user_id (FK), meter_qty (decimal), cylinder_items (JSON), service_items (JSON), total_amount (decimal), is_paid (bool), payment_method (enum), created_at, updated_at | Sales records      |
| **AuditLog**     | id (PK), user_id (FK), action_type (enum), payload (JSON), timestamp, ip_address, device_info                                                                                                    | Compliance trail   |

---

### 6.2 Enhanced ERD (with fields)

```
+-------------------------+
|        User             |
|-------------------------|
| id (PK)                 |
| username                |
| role                    |
| vehicle_no              |
| email                   |
| created_at              |
| updated_at              |
+-------------------------+
       |
       |1
       *
+-------------------------+
|     Transaction         |
|-------------------------|
| id (PK)                 |
| customer_id (FK)        |
| user_id (FK)            |
| meter_qty               |
| cylinder_items (JSON)   |
| service_items (JSON)    |
| total_amount            |
| is_paid                 |
| payment_method          |
| created_at              |
| updated_at              |
+-------------------------+
       |
       *
+-------------------------+
|      Customer           |
|-------------------------|
| id (PK)                 |
| name                    |
| contact_no              |
| address                 |
| payment_type            |
| rate_14kg               |
| rate_50kg               |
| active                  |
| created_at              |
| updated_at              |
+-------------------------+

+-------------------------+
|    Distribution         |
|-------------------------|
| id (PK)                 |
| distribution_no         |
| user_id (FK)            |
| item_id (FK)            |
| quantity                |
| status                  |
| created_at              |
| delivered_at            |
+-------------------------+
       |
       *
+-------------------------+
|      Inventory          |
|-------------------------|
| id (PK)                 |
| item_name               |
| item_type               |
| full_qty                |
| empty_qty               |
| depot_id (FK)           |
| last_updated            |
+-------------------------+

+-------------------------+
|       AuditLog          |
|-------------------------|
| id (PK)                 |
| user_id (FK)            |
| action_type             |
| payload (JSON)          |
| timestamp               |
| ip_address              |
| device_info             |
+-------------------------+
```

**Notes:**

* `Transaction.cylinder_items` and `service_items` store **typed quantities per category** in JSON (e.g., `{"9kg":2, "12.7kg":3}`).
* `AuditLog.payload` captures **full before/after context** for non-repudiation.
* `Inventory` tracks **full/empty quantities per depot**, updated atomically via backend services.

---

## 7️⃣ Backend Architecture

* **Configuration:** Environment-based settings, MySQL strict mode, JWT auth, modular Django apps
* **Domain Services:**

  * TransactionService → totals, inventory deduction
  * DistributionService → batch creation, inventory movement
  * BillingService → PDF/email dispatch
  * ReportService → filtered reports
  * AuditService → immutable logging
* **Security:** JWT + RBAC, atomic updates, server-generated IDs, audit logging

---

## 8️⃣ Frontend Architecture

* React SPA (Vite, JSX)
* React Router v7 loaders & actions
* TailwindCSS, mobile-first
* Offline queue (`offline.js`)
* Thermal printing (`usePrinter.js`)
* Root layout manages routing, auth, and layout shell

**Offline Strategy:** Mutations queued locally and replayed automatically when connectivity is restored.

---

## 9️⃣ Transaction & Distribution Flow

### 9.1 Transaction Creation

```
User
 │
 ▼
React SPA UI
 │
 ▼
Router Action
 │
 ├─ Offline → Queue locally
 │
 └─ Online → POST /api/transactions/
        │
        ▼
TransactionService → MySQL (atomic)
        │
        ▼
AuditLog → Frontend confirmation → Optional Print
```

### 9.2 Delivery/Distribution Batch

```
User
 │
 ▼
React SPA UI
 │
 ▼
Router Action
 │
 ├─ Offline → Queue batch
 │
 └─ Online → POST /api/distributions/
        │
        ▼
DistributionService → Inventory movement
        │
        ▼
AuditLog → UI confirmation → Optional Print
```

### 9.3 Failure & Recovery Pattern

```
Network/API failure
 │
 ▼
Router Action → Catches error
 │
 ▼
Queue persists payload
 │
 ▼
UI indicates "Saved Offline"
 │
 ▼
Network restored
 │
 ▼
Queued mutations replayed → Backend commits & audits
```


