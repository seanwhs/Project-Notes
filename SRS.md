# 📝 HSH Sales System — Frontend & Backend SRS

**Version:** 1.0
**Date:** 2026-01-14
**Author:** Sean Wong

---

## 1️⃣ Introduction

### 1.1 Purpose

Defines the **functional and non-functional requirements** for the **HSH Sales System**, a full-stack LPG sales & distribution platform.

**System delivers:**

* **Frontend:** React SPA with React Router v7 (loaders & actions), modular routing, offline persistence, and ESC/POS printing integration.
* **Backend:** Django REST Framework (DRF) API providing secure, role-based access to users, customers, inventory, transactions, deliveries, and audit logs.

**Audience:** Developers, QA engineers, system administrators, architects, and project stakeholders.

---

### 1.2 Scope

#### Frontend

* React SPA using **React Router v7** (loaders & actions)
* Core pages: Dashboard, Customers, Inventory, Transactions, Delivery
* Declarative fetch & mutation via loaders/actions
* Offline persistence via LocalStorage
* Receipt printing via `usePrinter` (ESC/POS placeholder)

#### Backend

* REST APIs with DRF
* Modules: Users, Customers, Inventory, Transactions, Distributions, Audit
* JWT authentication and role-based authorization
* Custom actions for transactions and batch creation
* APIs designed for **direct consumption by frontend loaders/actions**

---

## 2️⃣ Functional Requirements

### 2.1 Frontend Pages & Features

| Page            | Path            | Description                                               |
| --------------- | --------------- | --------------------------------------------------------- |
| Root            | `/`             | Application shell with header, navigation, and `<Outlet>` |
| Dashboard       | `/dashboard`    | KPIs, transaction totals, revenue, charts                 |
| Customers       | `/customers`    | CRUD, rate management, search/filter                      |
| Inventory       | `/inventory`    | Depot-wise stock, alerts, CSV/PDF export                  |
| Transactions    | `/transactions` | Create/list transactions, track payment status            |
| Delivery        | `/delivery`     | Create delivery batches, track equipment, print receipts  |
| Offline Support | N/A             | Local persistence and queued submissions                  |
| Printing        | N/A             | Receipt generation via `usePrinter` hook                  |

---

#### 2.1.1 Loaders & Actions

* **Loaders:** Fetch data before rendering routes
* **Actions:** Handle all mutations (POST/PUT)

> Ensures **separation of reads, writes, and side effects**.

---

### 2.2 Backend API Requirements

| Module       | Endpoint              | Methods                | Purpose            | Notes                     |
| ------------ | --------------------- | ---------------------- | ------------------ | ------------------------- |
| Accounts     | `/api/users/`         | GET, POST, PUT, DELETE | User management    | Admin-only                |
| Customers    | `/api/customers/`     | GET, POST, PUT, DELETE | Customer records   | Used by Customers page    |
| Inventory    | `/api/inventory/`     | GET, POST, PUT, DELETE | Stock management   | Used by Inventory page    |
| Transactions | `/api/transactions/`  | GET, POST, PUT, DELETE | Sales transactions | Includes `create_tx`      |
| Distribution | `/api/distributions/` | GET, POST, PUT, DELETE | Delivery tracking  | Includes `create_batch`   |
| Audit        | `/api/audit/`         | GET                    | Audit logs         | Compliance & traceability |
| Admin        | `/admin/`             | GET                    | Django admin       | Optional ops tooling      |

---

### 2.3 Data Model Mapping (Frontend ↔ Backend)

| Entity       | Frontend Fields                          | Backend Fields                        |
| ------------ | ---------------------------------------- | ------------------------------------- |
| User         | username, email, role, vehicle_no        | id, username, email, role, vehicle_no |
| Customer     | name, payment_type, rate_14kg, rate_50kg | id, name, payment_type, rates         |
| Inventory    | equipment, depot, full_qty, empty_qty    | id, equipment, depot, quantities      |
| Transaction  | customer_id, qty_14, qty_50              | id, customer, qty, total, is_paid     |
| Distribution | depot, equipment, quantity, status       | id, depot, equipment, status, number  |
| Audit        | user, action, entity                     | id, user, action, timestamp           |

---

## 3️⃣ Non-Functional Requirements

### Security

* JWT authentication
* Role-based access (Admin, Sales, Supervisor)
* HTTPS enforced

### Performance

* Dashboard load < 2s with 1,000+ transactions
* API response < 200 ms for standard ops

### Usability

* Mobile-first, responsive UI
* Clear navigation and form feedback

### Reliability

* Offline support with deferred sync
* Graceful handling of API/network failures

### Maintainability

* Modular React components
* DRF ViewSets with consistent patterns
* Documented APIs and services

### Scalability

* Multi-depot support
* Hundreds of concurrent users

---

## 4️⃣ User Stories

| ID  | Role  | Feature      | Description                                |
| --- | ----- | ------------ | ------------------------------------------ |
| US1 | Admin | Dashboard    | View KPIs and operational metrics          |
| US2 | Admin | Customers    | Manage customer records                    |
| US3 | Admin | Inventory    | Monitor and adjust stock                   |
| US4 | Admin | Transactions | Record and manage sales                    |
| US5 | Admin | Delivery     | Create delivery batches and print receipts |
| US6 | Admin | Audit        | Review system activity                     |
| US7 | Admin | Offline      | Continue work during connectivity loss     |
| US8 | Admin | Printing     | Print receipts via Web Bluetooth           |

---

## 5️⃣ System Architecture Overview

### Frontend

* React SPA
* React Router v7 (loaders & actions)
* TailwindCSS
* Offline via LocalStorage
* Printing via `usePrinter` hook

### Backend

* Django REST Framework
* JWT authentication
* Role-based permissions
* Custom transactional service layer

### Database

* MySQL 8.0

### Deployment

* Dockerized, cloud-ready architecture

### Tooling

* Vite
* React Router v7 plugin

---

## 6️⃣ Assumptions & Dependencies

* Valid user authentication required
* Online connectivity for real-time sync
* Offline queue handles intermittent network loss
* Web Bluetooth-compatible printer hardware
* Backend APIs available and secure
* Uniform React Router v7 conventions applied

---

## ✅ Outcome

This SRS provides a **single source of truth**:

* Frontend pages, navigation, and data flows
* Backend APIs, models, and permissions
* Loader/action integration patterns
* Offline-first and printing behavior
* Scalable, maintainable, production-ready architecture

---

### Architectural Note

> Explicit data boundaries enforced:
> **UI → Router Actions → APIs → Database**
> All side effects (writes, printing, offline queuing) are **isolated, observable, and testable**.


