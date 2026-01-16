# 🏗️ HSH SALES SYSTEM — DESIGN & ARCHITECTURE (v4.0)

**Version:** 4.0 — MySQL-focused, full-stack architecture, offline-first, audit-ready

---

## 1️⃣ Executive Overview

The **HSH Sales System** is a **full-stack platform for LPG sales, inventory, delivery, and auditing**, designed for:

* Field operations with **offline-first capability**
* Backend authority with **immutable business rules**
* Transparent, **audit-compliant workflows**
* Role-based security and operational scalability

**Objectives:**

* Enable **field sales staff** to record transactions and deliveries offline.
* Ensure **inventory integrity** and depot-scoped stock management.
* Provide **real-time reconciliation** and automated reporting.
* Maintain **audit trails** for regulatory and operational compliance.

**Core Operational Domains:**

| Domain                    | Responsibilities                                                                       |
| ------------------------- | -------------------------------------------------------------------------------------- |
| **Field Operations**      | Online/offline transaction capture, delivery/empty-return processing, receipt printing |
| **Inventory Management**  | Depot-specific full/empty cylinder tracking, stock reconciliation                      |
| **Transaction & Billing** | Meter, cylinder, service sales, totals validation, automated invoices                  |
| **Audit & Compliance**    | Immutable logging of critical actions, role-based access, timestamped events           |
| **Reporting & Analytics** | Filterable transaction history, exportable administrative reports (CSV/PDF)            |

---

## 2️⃣ Technology Stack

| Layer                | Technology/Tool                     | Role                                                      |
| -------------------- | ----------------------------------- | --------------------------------------------------------- |
| **Frontend**         | React 18 + Vite, JSX, TailwindCSS   | Mobile-first SPA, offline-capable UI, print-ready layouts |
| **Routing**          | React Router v7 (Loaders & Actions) | Data-driven routing and action-based state management     |
| **Backend**          | Django REST Framework (Python 3.11) | API, business rules, domain services, authentication      |
| **Database**         | MySQL 8.0                           | ACID-compliant relational store, primary source of truth  |
| **Authentication**   | JWT (SimpleJWT), RBAC               | Role-based access, secure session handling                |
| **Offline Support**  | LocalStorage + auto-sync            | Queue pending operations for offline usage                |
| **Printing**         | ESC/POS via Web Bluetooth           | Thermal receipt printing for field operations             |
| **Reporting**        | ReportLab (PDF), CSV/Excel          | Invoices and exportable reports                           |
| **Containerization** | Docker + Docker Compose             | Portable deployment for backend/frontend/databases        |
| **Monitoring**       | Sentry, Prometheus + Grafana        | Error tracking, performance monitoring, uptime            |

---

## 3️⃣ Design Principles

1. **Strict Frontend/Backend Separation** – UI never stores authoritative state; backend is the source of truth.
2. **Offline-first Reliability** – Field staff operations continue without connectivity.
3. **Server-owned Invariants** – Inventory counts, pricing, transaction totals are validated server-side.
4. **Auditability by Design** – All critical actions (create/update/delete) are logged immutably.
5. **Role-based Access Control (RBAC)** – Clear separation: Admin, Sales, Supervisor, Delivery.
6. **Operational Portability** – Dockerized deployments with environment-driven configuration.
7. **Scalable Data Modeling** – Optimized for MySQL with FK constraints, indexes, and JSON storage where needed.

---

## 4️⃣ System Architecture

### 4.1 Logical Architecture

```
Frontend (React SPA)
 ├─ Mobile-first responsive UI
 ├─ React Router v7 (Loaders & Actions)
 ├─ Offline queue for pending mutations (LocalStorage)
 └─ Thermal ESC/POS printing for receipts

Backend (Django REST Framework)
 ├─ JWT Authentication + RBAC
 ├─ Domain services:
 │   ├─ TransactionService (totals, validation, stock deduction)
 │   ├─ DistributionService (inventory movement)
 │   ├─ BillingService (PDF/email)
 │   ├─ ReportService (filtered reports)
 │   └─ AuditService (immutable logs)
 └─ MySQL database access (atomic transactions, constraints, triggers)

Database (MySQL 8.0)
 ├─ Normalized tables for Users, Customers, Inventory, Transactions, Distributions
 ├─ JSON columns for flexible cylinder/service items
 └─ AuditLog table for compliance trail
```

### 4.2 Container Architecture

```
┌────────────┐     ┌─────────────┐     ┌───────────────┐
│ Frontend   │     │ Backend     │     │ MySQL         │
│ :5173      │     │ :8000       │     │ :3306         │
│ Vol:/app   │     │ Vol:/app    │     │ Vol:mysql_data│
└────────────┘     └─────────────┘     └───────────────┘
```

---

## 5️⃣ Domain Modules

| Module           | Responsibilities                                               |
| ---------------- | -------------------------------------------------------------- |
| **Accounts**     | Users, roles, JWT auth, RBAC                                   |
| **Customers**    | Profiles, pricing tiers, contact info, payment terms           |
| **Inventory**    | Depot-based stock, full/empty cylinder tracking, service items |
| **Distribution** | Delivery batches, empty-return handling, inventory movement    |
| **Transactions** | Sales creation, total calculation, stock deduction             |
| **Billing**      | Invoice generation (PDF), automated email dispatch             |
| **Reports**      | Filterable, exportable transaction history, financial reports  |
| **Audit**        | Immutable logging for compliance, non-repudiable actions       |
| **Frontend**     | SPA offline-first operations, queue replay, printing, routing  |

---

## 6️⃣ Data Model & Detailed ERD

### 6.1 Users

| Field      | Type                                          | Key/Constraint                                        | Description               |
| ---------- | --------------------------------------------- | ----------------------------------------------------- | ------------------------- |
| id         | BIGINT UNSIGNED                               | PK, AI                                                | Unique user identifier    |
| username   | VARCHAR(50)                                   | UNIQUE, NOT NULL                                      | Login username            |
| email      | VARCHAR(100)                                  | UNIQUE                                                | Email                     |
| password   | VARCHAR(128)                                  | NOT NULL                                              | Hashed password           |
| role       | ENUM('Admin','Sales','Supervisor','Delivery') | NOT NULL                                              | User role                 |
| vehicle_no | VARCHAR(20)                                   | NULLABLE                                              | Assigned delivery vehicle |
| created_at | DATETIME                                      | DEFAULT CURRENT_TIMESTAMP                             | Creation timestamp        |
| updated_at | DATETIME                                      | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | Last update               |

### 6.2 Customers

| Field        | Type                  | Key/Constraint                                        | Description             |
| ------------ | --------------------- | ----------------------------------------------------- | ----------------------- |
| id           | BIGINT UNSIGNED       | PK, AI                                                | Customer identifier     |
| name         | VARCHAR(100)          | NOT NULL                                              | Customer name           |
| contact_no   | VARCHAR(20)           | NULLABLE                                              | Phone number            |
| address      | VARCHAR(255)          | NULLABLE                                              | Physical address        |
| payment_type | ENUM('Cash','Credit') | NOT NULL                                              | Payment preference      |
| rate_14kg    | DECIMAL(8,2)          | NOT NULL                                              | Price per 14kg cylinder |
| rate_50kg    | DECIMAL(10,2)         | NOT NULL                                              | Price per 50kg cylinder |
| active       | BOOLEAN               | DEFAULT TRUE                                          | Active customer flag    |
| created_at   | DATETIME              | DEFAULT CURRENT_TIMESTAMP                             | Creation timestamp      |
| updated_at   | DATETIME              | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | Last update             |

### 6.3 Inventory

| Field        | Type                               | Key/Constraint                                        | Description                 |
| ------------ | ---------------------------------- | ----------------------------------------------------- | --------------------------- |
| id           | BIGINT UNSIGNED                    | PK, AI                                                | Inventory item ID           |
| depot_id     | BIGINT UNSIGNED                    | FK → Depot(id)                                        | Depot location              |
| item_name    | VARCHAR(50)                        | NOT NULL                                              | Cylinder/Meter/Service name |
| item_type    | ENUM('Cylinder','Meter','Service') | NOT NULL                                              | Item category               |
| full_qty     | INT UNSIGNED                       | NOT NULL                                              | Full cylinder quantity      |
| empty_qty    | INT UNSIGNED                       | NOT NULL                                              | Empty cylinder quantity     |
| last_updated | DATETIME                           | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | Last update                 |

### 6.4 Distribution

| Field           | Type                                    | Key/Constraint            | Description                   |
| --------------- | --------------------------------------- | ------------------------- | ----------------------------- |
| id              | BIGINT UNSIGNED                         | PK, AI                    | Distribution ID               |
| distribution_no | VARCHAR(20)                             | UNIQUE                    | System-generated batch number |
| user_id         | BIGINT UNSIGNED                         | FK → User(id)             | Responsible staff             |
| item_id         | BIGINT UNSIGNED                         | FK → Inventory(id)        | Item distributed              |
| quantity        | INT UNSIGNED                            | NOT NULL                  | Quantity moved                |
| status          | ENUM('Pending','Completed','Cancelled') | NOT NULL                  | Current batch status          |
| created_at      | DATETIME                                | DEFAULT CURRENT_TIMESTAMP | Creation timestamp            |
| delivered_at    | DATETIME                                | NULLABLE                  | Completion timestamp          |

### 6.5 Transactions

| Field          | Type                           | Key/Constraint                                        | Description                |
| -------------- | ------------------------------ | ----------------------------------------------------- | -------------------------- |
| id             | BIGINT UNSIGNED                | PK, AI                                                | Transaction ID             |
| customer_id    | BIGINT UNSIGNED                | FK → Customer(id)                                     | Customer                   |
| user_id        | BIGINT UNSIGNED                | FK → User(id)                                         | Salesperson                |
| meter_qty      | DECIMAL(10,2)                  | DEFAULT 0                                             | Meter quantity sold        |
| cylinder_items | JSON                           | NOT NULL                                              | `{"9kg":2,"12.7kg":3}`     |
| service_items  | JSON                           | NOT NULL                                              | `{"Regulator":1,"Hose":2}` |
| total_amount   | DECIMAL(12,2)                  | NOT NULL                                              | Total sale amount          |
| is_paid        | BOOLEAN                        | DEFAULT FALSE                                         | Payment status             |
| payment_method | ENUM('Cash','Credit','Wallet') | NOT NULL                                              | Payment method             |
| created_at     | DATETIME                       | DEFAULT CURRENT_TIMESTAMP                             | Creation timestamp         |
| updated_at     | DATETIME                       | DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | Last update                |

### 6.6 AuditLog

| Field       | Type                                              | Key/Constraint            | Description             |
| ----------- | ------------------------------------------------- | ------------------------- | ----------------------- |
| id          | BIGINT UNSIGNED                                   | PK, AI                    | Audit record ID         |
| user_id     | BIGINT UNSIGNED                                   | FK → User(id)             | Actor                   |
| action_type | ENUM('CREATE','UPDATE','DELETE','LOGIN','LOGOUT') | NOT NULL                  | Type of action          |
| payload     | JSON                                              | NULLABLE                  | Before/After context    |
| timestamp   | DATETIME                                          | DEFAULT CURRENT_TIMESTAMP | When action occurred    |
| ip_address  | VARCHAR(45)                                       | NULLABLE                  | Source IP               |
| device_info | VARCHAR(255)                                      | NULLABLE                  | Browser/device metadata |

---

## 7️⃣ ASCII ERD with Relationships

```
+---------------------------+
|           User            |
|---------------------------|
| id           BIGINT PK    |
| username     VARCHAR(50)  |
| email        VARCHAR(100) |
| password     VARCHAR(128) |
| role         ENUM(...)    |
| vehicle_no   VARCHAR(20)  |
| created_at   DATETIME     |
| updated_at   DATETIME     |
+---------------------------+
           1
           |
           | *
+---------------------------+
|       Transaction         |
|---------------------------|
| id             BIGINT PK  |
| customer_id    BIGINT FK → Customer.id |
| user_id        BIGINT FK → User.id     |
| meter_qty      DECIMAL(10,2)           |
| cylinder_items JSON             |
| service_items  JSON             |
| total_amount   DECIMAL(12,2)           |
| is_paid        BOOLEAN                  |
| payment_method ENUM('Cash','Credit','Wallet') |
| created_at     DATETIME                 |
| updated_at     DATETIME                 |
+---------------------------+
           *
           |
           1
+---------------------------+
|        Customer           |
|---------------------------|
| id            BIGINT PK   |
| name          VARCHAR(100)|
| contact_no    VARCHAR(20) |
| address       VARCHAR(255)|
| payment_type  ENUM('Cash','Credit') |
| rate_14kg     DECIMAL(8,2)  |
| rate_50kg     DECIMAL(10,2) |
| active        BOOLEAN       |
| created_at    DATETIME      |
| updated_at    DATETIME      |
+---------------------------+

+---------------------------+
|        Distribution       |
|---------------------------|
| id              BIGINT PK |
| distribution_no VARCHAR(20) UNIQUE |
| user_id         BIGINT FK → User.id |
| item_id         BIGINT FK → Inventory.id |
| quantity        INT UNSIGNED        |
| status          ENUM('Pending','Completed','Cancelled') |
| created_at      DATETIME            |
| delivered_at    DATETIME NULLABLE   |
+---------------------------+
           *
           |
           1
+---------------------------+
|        Inventory          |
|---------------------------|
| id           BIGINT PK     |
| depot_id     BIGINT FK → Depot.id  |
| item_name    VARCHAR(50)   |
| item_type    ENUM('Cylinder','Meter','Service') |
| full_qty     INT UNSIGNED  |
| empty_qty    INT UNSIGNED  |
| last_updated DATETIME      |
+---------------------------+

+---------------------------+
|        AuditLog           |
|---------------------------|
| id          BIGINT PK      |
| user_id     BIGINT FK → User.id |
| action_type ENUM('CREATE','UPDATE','DELETE','LOGIN','LOGOUT') |
| payload     JSON             |
| timestamp   DATETIME       |
| ip_address  VARCHAR(45)    |
| device_info VARCHAR(255)   |
+---------------------------+
```

---

## 8️⃣ Offline-First Flow Diagram

```
                     ┌───────────────────────┐
                     │    React SPA Frontend │
                     │  (Vite, JSX, Tailwind)│
                     └─────────┬─────────────┘
                               │
                 ┌─────────────┴─────────────┐
                 │ Offline Queue (LocalStorage)│
                 └─────────────┬─────────────┘
                               │
             ┌─────────────────┴─────────────────┐
             │ Network Available?                 │
             │  ┌─────────────┐                   │
             │  │ Yes         │                   │
             │  │ POST /api/transactions       │
             │  └─────────────┘                   │
             │                                     │
             │  ┌─────────────┐                   │
             │  │ No (Offline)│                   │
             │  │ Keep queued │                   │
             │  │ mutation    │                   │
             │  └─────────────┘                   │
             └─────────────┬─────────────┘
                           │
             ┌─────────────▼─────────────┐
             │  Django REST Framework    │
             │  Backend API Services     │
             │ ┌───────────────────────┐│
             │ │ TransactionService    ││
             │ │  - Validate totals    ││
             │ │  - Deduct inventory   ││
             │ └───────────────────────┘│
             │ ┌───────────────────────┐│
             │ │ DistributionService   ││
             │ │  - Update inventory   ││
             │ └───────────────────────┘│
             │ ┌───────────────────────┐│
             │ │ AuditService          ││
             │ │  - Log mutation       ││
             │ │  - Capture JSON state ││
             │ └───────────────────────┘│
             └─────────────┬─────────────┘
                           │
            ┌──────────────▼──────────────┐
            │        MySQL Database       │
            └──────────────┬─────────────┘
                           │
            ┌──────────────▼─────────────┐
            │ Frontend Confirmation/UI   │
            └───────────────────────────┘
```

