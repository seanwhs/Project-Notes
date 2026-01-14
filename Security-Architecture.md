# 🔐 HSH Sales System — Security Architecture Document

**Version:** 1.0
**Date:** 2026-01-14
**Author:** Sean Wong

---

## 1️⃣ Purpose & Scope

This document defines the **security architecture** of the **HSH Sales System**, describing how security is enforced **by design**, not by convention. It focuses on:

* Trust boundaries and threat zones
* Authentication and authorization strategy
* Attack surface identification and mitigation
* Offline-first security considerations
* Audit, integrity, and non-repudiation

This document is intended for **architects, senior engineers, security reviewers, auditors, and DevOps teams**.

---

## 2️⃣ Security Design Principles

The system is designed around the following non-negotiable principles:

1. **Zero Trust at Boundaries**
   No client (browser, device, or network) is trusted by default.

2. **Backend as the Source of Truth**
   All authoritative decisions (pricing, inventory, totals, IDs) occur server-side.

3. **Explicit Trust Transitions**
   Trust is only elevated through authenticated, authorized, and validated requests.

4. **Side-Effect Isolation**
   Printing, offline storage, and UI state never mutate system truth directly.

5. **Auditability by Default**
   All critical state changes are logged and traceable.

---

## 3️⃣ Trust Zones Overview

The system is divided into **four explicit trust zones**, each with different threat assumptions.

### Trust Zone Classification

| Zone | Name                | Trust Level    | Description                       |
| ---- | ------------------- | -------------- | --------------------------------- |
| Z1   | Client Device       | Untrusted      | Browser, JS runtime, LocalStorage |
| Z2   | Network Boundary    | Untrusted      | Internet, HTTPS transport         |
| Z3   | Application Backend | Trusted        | DRF APIs, service layer           |
| Z4   | Data Store          | Highly Trusted | MySQL database                    |

---

## 4️⃣ Security Boundary Diagram (Logical)

```
┌──────────────────────────────────────────────┐
│ Z1 — Client Device (UNTRUSTED)                │
│                                              │
│  React SPA                                   │
│  ├─ UI Components                            │
│  ├─ Router Loaders / Actions                 │
│  ├─ OfflineService (LocalStorage Queue)      │
│  └─ usePrinter (ESC/POS)                     │
│                                              │
│  ⚠ User-controlled environment               │
└───────────────┬──────────────────────────────┘
                │ HTTPS + JWT
                │ (Authentication Boundary)
┌───────────────▼──────────────────────────────┐
│ Z2 — Network Boundary (UNTRUSTED)             │
│                                              │
│  Internet / Mobile Networks                   │
│  TLS termination                              │
│                                              │
│  ⚠ MITM, replay, tampering threats            │
└───────────────┬──────────────────────────────┘
                │ Verified HTTPS
                │ JWT Validation
┌───────────────▼──────────────────────────────┐
│ Z3 — Backend Application (TRUSTED)            │
│                                              │
│  Django REST Framework                        │
│  ├─ Auth & RBAC                               │
│  ├─ ViewSets / Actions                        │
│  ├─ Service Layer                             │
│  │   ├─ TransactionService                   │
│  │   ├─ DistributionService                  │
│  │   ├─ BillingService                       │
│  │   └─ AuditService                         │
│                                              │
│  ✔ Business rules enforced here               │
└───────────────┬──────────────────────────────┘
                │ SQL (Private Network)
┌───────────────▼──────────────────────────────┐
│ Z4 — Database (HIGH TRUST)                    │
│                                              │
│  MySQL 8.0                                   │
│  ├─ Users                                    │
│  ├─ Inventory                                │
│  ├─ Transactions                             │
│  ├─ Distributions                            │
│  └─ AuditLog                                 │
│                                              │
│  ✔ ACID, integrity enforced                  │
└──────────────────────────────────────────────┘
```

---

## 5️⃣ Authentication Architecture

### 5.1 Mechanism

* **JWT (JSON Web Tokens)** using DRF SimpleJWT
* Access token sent via `Authorization: Bearer <token>` header
* Refresh token rotation supported

### 5.2 Authentication Flow

```
User Login
 → Credentials submitted
 → Backend validates credentials
 → JWT issued (role embedded)
 → Token stored in memory (not LocalStorage)
 → Token attached to all API requests
```

### 5.3 Threat Mitigations

| Threat              | Mitigation                   |
| ------------------- | ---------------------------- |
| Token theft         | Short-lived access tokens    |
| Token tampering     | Signature verification       |
| Credential stuffing | Rate limiting, audit logging |

---

## 6️⃣ Authorization & RBAC

### 6.1 Roles

| Role       | Capabilities                                  |
| ---------- | --------------------------------------------- |
| Admin      | Full system access                            |
| Sales      | Transactions, deliveries, read-only inventory |
| Supervisor | Reporting, delivery oversight                 |

### 6.2 Enforcement Points

* View-level permission classes (`IsAdmin`, `IsSales`)
* Service-layer assertions (defense in depth)
* No client-side authorization decisions trusted

---

## 7️⃣ Attack Surface Analysis

### 7.1 Client-Side Attack Surface (Z1)

| Vector                 | Risk             | Mitigation             |
| ---------------------- | ---------------- | ---------------------- |
| LocalStorage tampering | Data poisoning   | Server-side validation |
| JS manipulation        | Forged payloads  | Backend recalculation  |
| Offline replay         | Duplicate writes | Idempotent endpoints   |

### 7.2 API Attack Surface (Z2/Z3)

| Vector               | Risk                   | Mitigation       |
| -------------------- | ---------------------- | ---------------- |
| Replay attacks       | Duplicate transactions | Idempotency keys |
| Injection            | Data corruption        | ORM + validation |
| Privilege escalation | Unauthorized access    | RBAC enforcement |

---

## 8️⃣ Offline-First Security Model

Offline mode is treated as **hostile but tolerated**.

### Rules

* Offline data is **never trusted**
* Offline actions are replayed as normal API requests
* Backend re-validates:

  * JWT
  * Permissions
  * Inventory availability
  * Pricing

```
Offline Payload
 → Stored locally
 → Replayed on reconnect
 → Full backend validation
 → Accepted or rejected
```

---

## 9️⃣ Inventory & Transaction Integrity

* Inventory updates occur **only** inside backend services
* Wrapped in **database transactions**
* Optimistic locking / row-level locking applied
* Prevents:

  * Double deduction
  * Race conditions
  * Client-forced stock changes

---

## 🔟 Audit & Non-Repudiation

### 10.1 Audit Events

Logged events include:

* Login / logout
* Transaction creation
* Distribution batch creation
* Inventory adjustments
* Administrative changes

### 10.2 Audit Guarantees

* Append-only logs
* No UI endpoint for deletion
* Admin read-only access
* Timestamps generated server-side

---

## 1️⃣1️⃣ Secure Deployment Model

* Backend and DB isolated in private Docker network
* Frontend exposed only via HTTPS
* Secrets stored in environment variables
* Database not exposed publicly

---

## 1️⃣2️⃣ Security Non-Negotiables

* UI never writes system truth
* All writes pass through authenticated APIs
* All inventory changes are atomic
* Printing never blocks persistence
* Audit logs cannot be modified

---

## 1️⃣3️⃣ Security Posture Summary

This architecture:

* Minimizes trust in hostile environments
* Reduces attack surface through strict boundaries
* Centralizes authority in the backend
* Supports offline operations safely
* Meets audit and compliance expectations

**Security is structural, not optional.**
