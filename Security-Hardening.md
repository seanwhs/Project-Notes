# 🔐 HSH Sales System — Security Hardening Pack

**Document Type:** Security Architecture & Hardening Specification
**Audience:** Architects, Security Reviewers, Senior Engineers, Auditors
**Scope:** Frontend (React SPA), Backend (Django REST Framework), Infrastructure, Offline Operations

---

## 1️⃣ Purpose & Security Posture

This document defines the **complete security hardening strategy** for the HSH Sales System. It extends the Security Architecture by specifying **threat models, enforcement matrices, replay protections, secrets management, and incident response controls**.

**Security posture:**

* Default-deny
* Server-authoritative
* Offline-tolerant but zero-trust
* Auditable by design

---

## 2️⃣ Trust Zones & Threat Assumptions

| Zone | Name                     | Trust Level    | Assumptions                            |
| ---- | ------------------------ | -------------- | -------------------------------------- |
| Z1   | Client Device / Browser  | Untrusted      | Can be compromised, modified, replayed |
| Z2   | Network / Transport      | Semi-trusted   | TLS protects integrity only            |
| Z3   | Backend API (DRF)        | Trusted        | Enforces all validation & authority    |
| Z4   | Database & Secrets Store | Highly Trusted | Single source of truth                 |

**Golden Rule:** *No data originating in Z1 is ever trusted without server-side verification.*

---

## 3️⃣ STRIDE Threat Model (By Trust Zone)

### Z1 — Client / Browser

| Threat                 | Example            | Mitigation                                 |
| ---------------------- | ------------------ | ------------------------------------------ |
| Spoofing               | Forged requests    | JWT verification, signature checks         |
| Tampering              | Modified payloads  | Server-side validation, schema enforcement |
| Repudiation            | Denying actions    | Audit logs with user & timestamp           |
| Information Disclosure | XSS                | CSP, escaping, no secrets in JS            |
| DoS                    | Excess submissions | Rate limiting, idempotency keys            |
| Elevation              | UI role bypass     | Backend RBAC enforcement                   |

### Z2 — Network

| Threat | Example         | Mitigation               |
| ------ | --------------- | ------------------------ |
| MITM   | Packet sniffing | HTTPS/TLS only           |
| Replay | Resent requests | Nonces, idempotency keys |

### Z3 — Backend API

| Threat               | Example                | Mitigation                      |
| -------------------- | ---------------------- | ------------------------------- |
| Privilege escalation | Access admin endpoints | Permission classes, RBAC matrix |
| Injection            | SQL injection          | ORM, parameterized queries      |
| Logic abuse          | Inventory manipulation | Service-layer invariants        |

---

## 4️⃣ RBAC Enforcement Matrix

| Endpoint                         | Action | Admin | Sales | Supervisor |
| -------------------------------- | ------ | ----- | ----- | ---------- |
| /api/users/                      | CRUD   | ✅     | ❌     | ❌          |
| /api/customers/                  | CRUD   | ✅     | ✅     | ❌          |
| /api/inventory/                  | Update | ✅     | ❌     | ❌          |
| /api/transactions/create_tx/     | Create | ✅     | ✅     | ❌          |
| /api/distributions/create_batch/ | Create | ✅     | ❌     | ✅          |
| /api/audit/                      | View   | ✅     | ❌     | ❌          |

**Rule:** RBAC is enforced **server-side only**. UI visibility does not imply permission.

---

## 5️⃣ Offline Sync & Replay Protection

### 5.1 Threats

* Replay attacks from queued offline payloads
* Duplicate submissions
* Payload tampering

### 5.2 Controls

* **Idempotency Key** per action (UUID v4)
* **Server-side replay table**
* **Time-bound validity** (e.g. 24 hours)
* **Signature check** against authenticated user

```
Client Action
 → generate idempotency_key
 → store locally
 → submit when online

Server
 → check key uniqueness
 → process once only
 → reject duplicates
```

---

## 6️⃣ Audit Log Integrity Guarantees

Audit logs are **append-only** and immutable.

| Property    | Enforcement             |
| ----------- | ----------------------- |
| No updates  | Model-level constraints |
| No deletes  | Admin UI disabled       |
| Attribution | user_id (FK) required   |
| Timestamp   | Server-generated only   |

Audit logs include:

* User
* Action
* Entity
* Before/After (hash or snapshot)
* IP / User-Agent (optional)

---

## 7️⃣ Secrets & Key Management

### 7.1 JWT

* Short-lived access tokens
* Refresh tokens rotated
* Signing key stored in env / secret store

### 7.2 Application Secrets

| Secret      | Storage              |
| ----------- | -------------------- |
| SECRET_KEY  | Environment variable |
| DB_PASSWORD | Docker secret / env  |
| Email creds | Secret store         |

**Rules:**

* Never committed to source control
* Rotated periodically
* Separate per environment

---

## 8️⃣ API Hardening Controls

* Rate limiting (per-IP, per-user)
* Strict serializers (no dynamic fields)
* Input size limits
* Explicit allow-lists
* Atomic DB transactions
* No business logic in serializers

---

## 9️⃣ Frontend Security Controls

* CSP headers
* No tokens in LocalStorage (JWT in memory or HttpOnly cookie)
* Offline queue encrypted-at-rest (optional)
* No hidden admin features

---

## 🔟 Incident Response & Monitoring

### 10.1 Detection

* Auth failures
* Permission violations
* Inventory anomalies
* Replay rejections

### 10.2 Response

| Step      | Action                          |
| --------- | ------------------------------- |
| Detect    | Alert / log review              |
| Contain   | Disable account / revoke tokens |
| Eradicate | Patch vulnerability             |
| Recover   | Restore integrity               |
| Review    | Post-incident report            |

---

## 1️⃣1️⃣ Security Non-Negotiables

* UI is never authoritative
* Inventory changes only server-side
* All writes audited
* Offline == untrusted
* Printing is post-commit only

---

## 1️⃣2️⃣ Security Validation Checklist

* [ ] All endpoints protected by permission classes
* [ ] Idempotency enforced
* [ ] Audit logs immutable
* [ ] Secrets externalized
* [ ] HTTPS enforced
* [ ] Offline replay tested

---

## ✅ Outcome

This hardening pack elevates the HSH Sales System to **enterprise-grade security**, making it suitable for:

* Formal security reviews
* Compliance audits
* High-risk offline environments
* Long-term maintainability

**Security is structural, not optional.**
