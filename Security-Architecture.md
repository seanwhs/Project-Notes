# 🔐 HSH SALES SYSTEM — SECURITY ARCHITECTURE (v2.0)

**Author:** Sean Wong | **Date:** 2026-01-15

**Purpose:** Define how the HSH Sales System enforces **security by design**, protecting data integrity, user roles, transactions, and offline operations.

---

## 1️⃣ Security Principles

1. **Zero Trust at All Boundaries** – Never trust the client, browser, or network implicitly.
2. **Backend as the Single Source of Truth** – Pricing, inventory, and transactions validated server-side.
3. **Explicit Trust Elevation** – Requests must be authenticated, authorized, and validated.
4. **Side-Effect Isolation** – UI state, printing, and offline data cannot mutate authoritative data directly.
5. **Auditability by Default** – All critical actions are logged with immutable timestamps.

---

## 2️⃣ Trust Zones

| Zone | Name             | Trust Level    | Key Components                            |
| ---- | ---------------- | -------------- | ----------------------------------------- |
| Z1   | Client Device    | Untrusted      | Browser, JS runtime, LocalStorage         |
| Z2   | Network Boundary | Untrusted      | Internet, HTTPS, TLS                      |
| Z3   | Backend App      | Trusted        | Django REST Framework APIs, Service Layer |
| Z4   | Data Store       | Highly Trusted | MySQL/PostgreSQL DB with ACID enforcement |

**Concept:** Explicit boundaries enforce security by **minimizing assumptions of trust**.

---

## 3️⃣ Authentication

* **Mechanism:** JWT (DRF SimpleJWT) – short-lived access + refresh tokens.
* **Flow:**

```
Login → Credential Validation → JWT Issued (role embedded) → Stored in memory → API requests include JWT
```

* **Mitigations:**

  * Short-lived tokens
  * Signature verification
  * Rate limiting & lockout policies
  * Audit logging of login attempts

---

## 4️⃣ Authorization (RBAC)

| Role       | Permissions                                         |
| ---------- | --------------------------------------------------- |
| Admin      | Full system access (all transactions, reports)      |
| Sales      | Transaction creation, delivery, read-only inventory |
| Supervisor | Reporting, delivery oversight                       |

* **Enforcement:** Backend view + service layers (defense-in-depth)
* **Note:** Client-side enforcement is **UI-only**; server-side authority is mandatory.

---

## 5️⃣ Attack Surface Analysis

### Client-Side (Z1)

| Vector              | Risk             | Mitigation             |
| ------------------- | ---------------- | ---------------------- |
| LocalStorage tamper | Data poisoning   | Server-side validation |
| JS manipulation     | Forged payloads  | Backend recalculation  |
| Offline replay      | Duplicate writes | Idempotent endpoints   |

### Backend / API (Z2/Z3)

| Vector               | Risk                   | Mitigation                  |
| -------------------- | ---------------------- | --------------------------- |
| Replay attacks       | Duplicate transactions | Idempotency keys            |
| Injection            | Data corruption        | ORM + Serializer validation |
| Privilege escalation | Unauthorized access    | RBAC enforcement            |

---

## 6️⃣ Offline Security Model

* **Offline data is never trusted**
* Stored locally in **transaction queue** (buffer)
* Replayed when online with full backend validation

```
Offline Payload → Reconnect → API Validation (JWT + RBAC + Inventory + Pricing) → Accept/Reject
```

* Offline replay **cannot bypass inventory or pricing rules**
* Audit logs include replayed transaction metadata

---

## 7️⃣ Transaction & Inventory Integrity

* Updates occur **only in backend service layer**
* **Atomic DB transactions** ensure consistency
* **Optimistic/row-level locking** prevents double deductions
* **Client cannot manipulate stock**; all calculations validated server-side

**Flow:**

```
Client → Action → Backend Validation → DB Transaction → Audit Log → Response
```

---

## 8️⃣ Audit & Non-Repudiation

* **Logged Events:**

  * Login/logout
  * Transaction creation & edits
  * Batch distribution
  * Inventory changes
  * Admin actions

* **Guarantees:**

  * Append-only
  * Server-side timestamps
  * Read-only access for auditing
  * No deletions allowed

---

## 9️⃣ Deployment Security

* Backend + DB isolated in private Docker network
* Frontend served **HTTPS only**
* Secrets via environment variables
* Database **never publicly exposed**

---

## 🔟 Security Non-Negotiables

* UI cannot write system truth
* All writes via **authenticated, authorized APIs**
* Inventory changes are **atomic**
* Printing cannot block persistence
* Audit logs are immutable

---

## 1️⃣1️⃣ ERD — Entities & Fields (Enhanced)

```
┌───────────────────┐
│     Users          │
├───────────────────┤
│ id (PK)            │
│ username           │
│ email              │
│ password_hash      │
│ role (Admin/Sales) │
│ created_at         │
│ updated_at         │
└───────────────────┘

┌───────────────────────┐
│    Customers           │
├───────────────────────┤
│ id (PK)                │
│ name                   │
│ address                │
│ contact_number         │
│ created_at             │
└───────────────────────┘

┌─────────────────────────┐
│   Transactions           │
├─────────────────────────┤
│ id (PK)                  │
│ customer_id (FK)         │
│ meter_qty                │
│ cylinder_items JSON      │
│ service_items JSON       │
│ total_amount             │
│ status (Pending/Posted)  │
│ created_by (FK User)     │
│ created_at               │
└─────────────────────────┘

┌───────────────────────┐
│ Inventory              │
├───────────────────────┤
│ id (PK)                │
│ item_name              │
│ item_type (Cylinder/Service) │
│ quantity               │
│ depot_id               │
└───────────────────────┘
```

**Note:** JSON fields for cylinders & services allow flexible extension and offline buffering.

---

## 1️⃣2️⃣ Sequence Diagram — Offline Transaction

```
Client                   Backend
  |                        |
  |----Action (Offline)--> |
  |   store in buffer      |
  |                        |
  |<---UI Confirmation---- |
  |                        |
  |----Reconnect---------->|
  |   replay buffered tx   |
  |----Validation----------|
  |----DB Transaction------|
  |----Audit Log---------->|
  |<---Success Response--- |
```

---

## 1️⃣3️⃣ Security Posture Summary

* **Structural trust boundaries** reduce attack surface
* Backend is **authoritative source**
* Offline operations **securely validated**
* Audit & compliance ready
* Security **enforced by design**

