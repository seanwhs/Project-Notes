# 📝 HSH Sales System — Frontend & Backend

## Software Requirements Specification (SRS)

**Version:** 1.0
**Date:** 2026-01-14
**Author:** Sean Wong

---

## 1️⃣ Introduction

### 1.1 Purpose

This document defines the **functional and non-functional requirements** for the **HSH Sales System**, a full-stack LPG sales and distribution platform comprising a modern web frontend and a secure RESTful backend.

The system delivers:

* **Frontend:** A React single-page application (SPA) built with React Router v7, featuring modular routing, offline capability, and receipt printing integration.
* **Backend:** A Django REST Framework (DRF) API layer providing secure, role-based access to users, customers, inventory, transactions, deliveries, and audit logs.

This SRS is intended for **software developers, QA engineers, system administrators, architects**, and project stakeholders.

---

### 1.2 Scope

The system includes the following major capabilities:

#### Frontend Scope

* React SPA using **React Router v7 (loaders & actions)**
* Core pages:

  * Dashboard
  * Customers
  * Inventory
  * Transactions
  * Delivery
* Declarative data fetching and mutation via loaders/actions
* Offline data persistence using LocalStorage
* Receipt printing support via ESC/POS (Web Bluetooth placeholder)

#### Backend Scope

* RESTful APIs implemented using Django REST Framework
* Core modules:

  * Users
  * Customers
  * Inventory
  * Transactions
  * Distributions
  * Audit Logs
* JWT-based authentication and role-based authorization
* Custom API actions for transaction and batch creation
* APIs designed for direct consumption by frontend loaders/actions

---

## 2️⃣ Functional Requirements

### 2.1 Frontend Pages & Features

| Page            | Path            | Description                                                   |
| --------------- | --------------- | ------------------------------------------------------------- |
| Root            | `/`             | Application shell with header, navigation, and `<Outlet>`     |
| Dashboard       | `/dashboard`    | KPIs, transaction totals, revenue overview, charts            |
| Customers       | `/customers`    | Customer CRUD, rate management, search and filtering          |
| Inventory       | `/inventory`    | Depot-wise stock levels, alerts, CSV/PDF export               |
| Transactions    | `/transactions` | Transaction creation, listing, payment status tracking        |
| Delivery        | `/delivery`     | Delivery batch creation, equipment movement, receipt printing |
| Offline Support | N/A             | Local persistence and queued submissions                      |
| Printing        | N/A             | Receipt generation via `usePrinter` hook                      |

---

#### 2.1.1 Loaders and Actions

* **Loaders**

  * Responsible for fetching data from backend APIs
  * Executed before route rendering
* **Actions**

  * Handle all data mutations (POST / PUT)
  * Represent the only write path from UI to backend

This ensures a **clear separation between reads, writes, and side effects**.

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

* JWT-based authentication
* Role-based access control (Admin, Sales, Supervisor)
* HTTPS enforced across all environments

### Performance

* Dashboard load time < 2 seconds with 1,000+ transactions
* API response time < 200 ms for standard operations

### Usability

* Mobile-first, responsive UI
* Clear navigation and form feedback states

### Reliability

* Offline support with deferred synchronization
* Graceful handling of API and network failures

### Maintainability

* Modular React components
* DRF ViewSets with consistent patterns
* Well-documented APIs and services

### Scalability

* Support for multiple depots
* Hundreds of concurrent authenticated users

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
* LocalStorage-based offline support
* Printing via `usePrinter` hook

### Backend

* Django REST Framework
* JWT authentication
* Role-based permissions
* Custom transactional service layer

### Database

* MySQL 8.0

### Deployment

* Dockerized services
* Cloud-ready architecture

### Tooling

* Vite
* React Router v7 plugin

---

## 6️⃣ Assumptions & Dependencies

* All users authenticate with valid credentials
* Online connectivity is required for real-time sync
* Offline queue handles intermittent network loss
* Web Bluetooth-compatible hardware is available for printing
* Backend APIs are consistently available and secured
* React Router v7 conventions are applied uniformly

---

## ✅ Outcome

This SRS serves as a **single source of truth** for the HSH Sales System by defining:

* Frontend pages, navigation, and data flows
* Backend APIs, models, and permissions
* Loader/action integration patterns
* Offline-first behavior and printing concerns
* A scalable, maintainable, production-ready architecture

---

### Architectural note (optional, but strong):

> This system deliberately enforces **explicit data boundaries**:
> UI → Router Actions → APIs → Database.
> All side effects (writes, printing, offline queuing) are isolated, observable, and testable.

