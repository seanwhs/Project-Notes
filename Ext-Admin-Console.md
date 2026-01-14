# 📝 HSH Sales System — Admin Console Software Requirements Specification (SRS)

**Version:** 1.0
**Date:** 2026-01-14
**Author:** Sean Wong

---

## 1️⃣ Introduction

### 1.1 Purpose

This document defines the **functional and non-functional requirements** for the **Admin Console** of the **HSH Sales System**, a full-stack enterprise LPG logistics and sales platform. The Admin Console enables authorized users to **manage system data, monitor operations, generate reports, and enforce compliance**.

### 1.2 Scope

The Admin Console provides:

* **User & Role Management** – Manage users, roles, and permissions.
* **Customer Management** – Add, edit, and manage customer data, rates, and payment terms.
* **Inventory Management** – Track cylinder stock, depots, and equipment.
* **Transaction Management** – View, create, approve transactions, and track payments.
* **Distribution / Delivery Management** – Track deliveries, collections, and returns.
* **Financial Management** – Generate, send, and manage invoices.
* **Reporting & Analytics** – Predefined and custom reports with charts and export capabilities.
* **Audit & Compliance** – Track user actions and generate audit logs.
* **System Configuration** – Manage depots, equipment types, and application settings.

The Admin Console integrates seamlessly with the **React Router v7 frontend** and **Django REST Framework backend**.

### 1.3 Intended Audience

* System Administrators
* Operations Managers
* Finance Department
* Developers & QA Engineers

---

## 2️⃣ Functional Requirements

### 2.1 Dashboard

**Purpose:** Provide a high-level overview of system operations.

**Features:**

* **KPIs / Metrics**

  * Total Transactions (Daily / Monthly / Yearly)
  * Total Revenue
  * Pending vs Completed Deliveries
  * Inventory Levels (Full vs Empty)
  * Payment Status Overview

* **Graphs & Charts**

  * Sales trends over time
  * Inventory by depot
  * Top customers by purchase volume

* **Quick Actions**

  * Add Customer, Transaction, or Inventory
  * Batch approval of deliveries

* **Filters:** Date range, depot, customer type

---

### 2.2 User & Role Management

**Purpose:** Administer system users and permissions.

**Features:**

* CRUD operations for users
* Role assignment (Admin, Sales, Supervisor)
* Activity monitoring: last login, active sessions
* Access audit logs filtered by user/action/date
* Enable/disable user accounts

---

### 2.3 Customer Management

**Purpose:** Manage customers and billing rates.

**Features:**

* CRUD operations for customer records
* Track and update cylinder rates (14kg, 50kg, POL)
* Track customer status (active/inactive)
* Set payment type (Cash, Monthly, Credit)
* Search and filter by name, depot, or last transaction
* Maintain rate history

---

### 2.4 Inventory Management

**Purpose:** Manage stock of LPG cylinders and equipment.

**Features:**

* Depot-wise stock display (full & empty cylinders)
* Low stock alerts for critical items
* Manual adjustments for damaged/lost items
* Batch updates and transfers between depots
* Export inventory reports (CSV / PDF)

---

### 2.5 Transaction Management

**Purpose:** Manage and review sales and meter readings.

**Features:**

* Full transaction listing with filters (customer, date, paid/unpaid)
* Mark transactions as paid/unpaid
* View transaction details, including meter readings
* Download/resend PDF invoices
* Bulk operations: approve, reject, export

---

### 2.6 Distribution / Delivery Management

**Purpose:** Track cylinder deliveries, collections, and returns.

**Features:**

* Create, edit, delete delivery batches
* Track status: pending, in-progress, completed
* Print receipts via thermal printing or PDF
* Depot-wise delivery reporting
* Batch approval for multiple distributions

---

### 2.7 Billing & Financials

**Purpose:** Manage invoices and payment tracking.

**Features:**

* View generated invoices
* Download PDFs or resend via email
* Track payment status
* Integrate with QuickBooks for transaction sync
* Generate reconciliation reports

---

### 2.8 Reporting & Analytics

**Purpose:** Provide analytical insights.

**Features:**

* Predefined reports: daily/weekly/monthly sales, revenue by customer, inventory turnover
* Custom reports: filter by customer, depot, or date range
* Export reports in CSV, Excel, PDF
* Charts: bar, line, pie for visual analysis

---

### 2.9 Audit & Compliance

**Purpose:** Ensure traceability and regulatory compliance.

**Features:**

* Log all user actions (create, update, delete)
* Filterable by user, action type, timestamp
* Generate downloadable compliance reports
* Alerts for suspicious activity

---

### 2.10 System Configuration

**Purpose:** Administer application-level settings.

**Features:**

* Depot management (add/edit/remove depots)
* Equipment/cylinder type management
* Offline sync interval and receipt formatting
* Role & permission management
* Global search functionality

---

## 3️⃣ Non-Functional Requirements

* **Security:** Role-based access control, JWT authentication, HTTPS enforcement
* **Performance:** Dashboard load <2s for 1,000+ transactions; report exports <5s
* **Usability:** Mobile-first, intuitive navigation
* **Reliability:** Offline support with auto-sync
* **Maintainability:** Modular React components, DRF viewsets, documented APIs
* **Scalability:** Support multiple depots and hundreds of concurrent users

---

## 4️⃣ User Stories

| ID   | Role  | Feature                            | Description                                         |
| ---- | ----- | ---------------------------------- | --------------------------------------------------- |
| US1  | Admin | Dashboard                          | View KPIs, charts, quick actions                    |
| US2  | Admin | User Management                    | Add/Edit/Delete users and assign roles              |
| US3  | Admin | Customer Management                | Add/Edit/Delete customer data and rates             |
| US4  | Admin | Inventory Management               | Monitor stock levels and update inventory           |
| US5  | Admin | Transaction Management             | View, approve, mark as paid/unpaid transactions     |
| US6  | Admin | Distribution / Delivery Management | Create delivery batches, print receipts             |
| US7  | Admin | Billing & Financials               | View/send invoices, QuickBooks sync                 |
| US8  | Admin | Reporting & Analytics              | Generate and export reports                         |
| US9  | Admin | Audit & Compliance                 | View audit logs, generate compliance reports        |
| US10 | Admin | System Configuration               | Manage depots, equipment types, roles, app settings |

---

## 5️⃣ System Architecture Overview

* **Frontend:** React SPA with React Router v7, TailwindCSS, offline support via LocalStorage
* **Backend:** Django REST Framework with JWT authentication and role-based permissions
* **Database:** MySQL 8.0
* **Printing:** ESC/POS via Web Bluetooth
* **Deployment:** Dockerized, cloud-ready

---

## 6️⃣ Assumptions & Dependencies

* Users have valid authentication credentials
* Internet connectivity for online operations; offline queue handles intermittent connectivity
* Thermal printer hardware compatible with Web Bluetooth API
* Backend APIs are available and secure

---

✅ **This SRS provides a structured blueprint for developing the Admin Console**, covering functional features, non-functional requirements, user stories, system architecture, and operational constraints.

