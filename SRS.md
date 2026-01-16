# 📝 HSH Sales System — Frontend & Backend SRS (v1.1)

**Version:** 1.1
**Date:** 2026-01-16
**Author:** Sean Wong

---

## 1️⃣ Introduction

### 1.1 Purpose

Defines the **functional and non-functional requirements** for the **HSH Sales System**, a full-stack LPG sales and distribution platform.

**System delivers:**

* **Frontend:** React SPA with React Router v7 (loaders & actions), modular routing, offline persistence, and ESC/POS printing integration.
* **Backend:** Django REST Framework (DRF) API providing secure, role-based access to users, customers, inventory, transactions, deliveries, and audit logs.

**Audience:** Developers, QA engineers, system administrators, architects, project stakeholders.

---

### 1.2 Scope

#### Frontend

* React SPA using **React Router v7** with loaders/actions for declarative data fetching and mutation
* Core pages: Dashboard, Customers, Inventory, Transactions, Delivery
* Offline persistence via LocalStorage and queued actions
* Receipt printing via `usePrinter` hook (ESC/POS integration placeholder)
* Mobile-first, responsive UI with TailwindCSS

#### Backend

* REST APIs with DRF and JWT authentication
* Modules: Users, Customers, Inventory, Transactions, Distributions, Audit
* Role-based authorization (Admin, Sales, Supervisor)
* Transactional and batch operations exposed for frontend consumption
* APIs optimized for loader/action integration

---

## 2️⃣ Functional Requirements

### 2.1 Frontend Pages & Features

| Page            | Path            | Description                                              |
| --------------- | --------------- | -------------------------------------------------------- |
| Root            | `/`             | Application shell with header, navigation, `<Outlet>`    |
| Dashboard       | `/dashboard`    | KPIs, revenue, transaction totals, charts                |
| Customers       | `/customers`    | CRUD operations, rate management, search/filter          |
| Inventory       | `/inventory`    | Depot-wise stock management, alerts, CSV/PDF export      |
| Transactions    | `/transactions` | Create/list transactions, payment tracking               |
| Delivery        | `/delivery`     | Create delivery batches, track equipment, print receipts |
| Offline Support | N/A             | Local persistence, queued submissions for offline usage  |
| Printing        | N/A             | Receipt generation via `usePrinter` hook                 |

#### 2.1.1 Loaders & Actions

* **Loaders:** Pre-fetch route data before rendering
* **Actions:** Handle mutations (POST/PUT/PATCH)

> Ensures **separation of read vs write operations**, improves testability, and allows offline-first design.

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

### 2.3 Frontend ↔ Backend Data Model Mapping

| Entity       | Frontend Fields                          | Backend Fields                        | Notes                                 |
| ------------ | ---------------------------------------- | ------------------------------------- | ------------------------------------- |
| User         | username, email, role, vehicle_no        | id, username, email, role, vehicle_no | Loader: `/users/me/`                  |
| Customer     | name, payment_type, rate_14kg, rate_50kg | id, name, payment_type, rates         | Editable via `/customers/`            |
| Inventory    | equipment, depot, full_qty, empty_qty    | id, equipment, depot, quantities      | Read-only table, alerts for low stock |
| Transaction  | customer_id, qty_14, qty_50              | id, customer, qty, total, is_paid     | POST/PATCH via loaders/actions        |
| Distribution | depot, equipment, quantity, status       | id, depot, equipment, status, number  | Batch creation supported              |
| Audit        | user, action, entity                     | id, user, action, timestamp           | Logged per mutation                   |

---

## 3️⃣ Non-Functional Requirements

### Security

* JWT-based authentication
* Role-based access (Admin, Sales, Supervisor)
* HTTPS enforced for all endpoints

### Performance

* Dashboard load < 2s with 1,000+ transactions
* API response < 200ms for standard CRUD operations

### Usability

* Mobile-first, responsive UI
* Clear navigation and form validation feedback

### Reliability

* Offline support with deferred synchronization
* Graceful handling of API/network failures

### Maintainability

* Modular React components
* DRF ViewSets with consistent patterns
* Documented APIs and services

### Scalability

* Multi-depot support
* Support hundreds of concurrent users

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

* React SPA with React Router v7 (loaders & actions)
* TailwindCSS for responsive design
* Offline via LocalStorage
* Printing via `usePrinter` hook
* Modular pages and components

### Backend

* Django REST Framework
* JWT authentication & role-based permissions
* Transactional service layer and audit logging
* Optimized for loader/action integration

### Database

* MySQL 8.0

### Deployment

* Dockerized containers
* Cloud-ready architecture (CI/CD friendly)

### Tooling

* Vite
* React Router v7 plugin

---

## 6️⃣ Assumptions & Dependencies

* Valid user authentication required
* Online connectivity for real-time sync
* Offline queue handles intermittent network loss
* Web Bluetooth-compatible printer hardware
* Backend APIs are available and secure
* Uniform React Router v7 conventions applied

---

## ✅ Outcome

This SRS provides a **single source of truth** for:

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

