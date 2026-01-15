# 🧪 HSH SALES SYSTEM — Postman Test Plan

**Goal:** Verify that the backend system works correctly, enforces business rules, and keeps data safe.
Testing proceeds **module by module**, explaining **purpose, expected results, and verification steps**.

---

## 0️⃣ SET UP POSTMAN ENVIRONMENT

Create a **Postman environment** (`HSH Local`) to make requests dynamic and chainable.

| Variable         | Value                       | Purpose               |
| ---------------- | --------------------------- | --------------------- |
| `base_url`       | `http://127.0.0.1:8000/api` | Local backend URL     |
| `access_token`   | `""`                        | JWT token after login |
| `user_id`        | `""`                        | Sales user ID         |
| `customer_id`    | `""`                        | Customer ID           |
| `depot_id`       | `""`                        | Depot ID              |
| `transaction_id` | `""`                        | Transaction ID        |

> ✅ Using variables allows **reusable, dynamic, and sequential requests**.

---

## 1️⃣ AUTHENTICATION — GET JWT TOKEN

**Purpose:** Log in as Admin and obtain a JWT token to access protected endpoints.

* **Method:** POST
* **URL:** `{{base_url}}/token/`
* **Body:**

```json
{
  "username": "admin",
  "password": "adminpass"
}
```

**Expected Result:** Response contains `access` → save as `{{access_token}}`.

**Postman Test Script:**

```javascript
pm.test("JWT token received", function () {
    const json = pm.response.json();
    pm.expect(json).to.have.property("access");
    pm.environment.set("access_token", json.access);
});
```

---

## 2️⃣ ACCOUNTS — USER MANAGEMENT

**Purpose:** Verify Admin can manage users; Sales users have restricted access.

### 2.1 Create Sales User

* **Method:** POST
* **URL:** `{{base_url}}/accounts/users/`
* **Body:**

```json
{
  "username": "sales1",
  "password": "salespass",
  "role": "SALES",
  "vehicle_no": "LPG-001"
}
```

**Expected:** Status 201, save `id` as `{{user_id}}`.

---

### 2.2 List Users

* **Method:** GET
* **URL:** `{{base_url}}/accounts/users/`
* **Check:** Status 200, JSON array contains users.

---

### 2.3 Role-Based Access Control (RBAC) Test

1. Log in as Sales user.
2. Attempt `GET /accounts/users/`.

**Expected:** 403 Forbidden → ensures Admin-only access.

---

## 3️⃣ CUSTOMERS — CONTRACTS & RATES

**Purpose:** Ensure customer data is correctly created and readable.

### 3.1 Create Customer

* **Method:** POST
* **URL:** `{{base_url}}/customers/`
* **Body:**

```json
{
  "name": "John Doe Industries",
  "address": "123 Industrial Rd",
  "payment_type": "CREDIT",
  "meter_rate": "50.00",
  "rate_9kg": "20.00",
  "rate_12_7kg": "25.00",
  "rate_14kg": "30.00",
  "rate_50kg_pol": "200.00",
  "rate_50kg_l": "220.00",
  "last_meter_reading": "0.00"
}
```

**Expected:** Status 201, save `id` as `{{customer_id}}`.

---

### 3.2 List Customers

* **Method:** GET
* **URL:** `{{base_url}}/customers/`
* **Check:** Status 200, array contains the new customer.

---

## 4️⃣ DEPOTS & INVENTORY (READ-ONLY)

### 4.1 List Depots

* **Method:** GET
* **URL:** `{{base_url}}/depots/`
* **Check:** Status 200, save first `id` as `{{depot_id}}`.

---

### 4.2 List Inventory

* **Method:** GET
* **URL:** `{{base_url}}/inventory/`
* **Check:** Status 200, array of stock quantities, cannot modify via API.

---

## 5️⃣ DISTRIBUTION — STOCK MOVEMENT

**Purpose:** Ensure atomic stock movements and audit logging.

### 5.1 Collection

* **Method:** POST
* **URL:** `{{base_url}}/distribution/execute/`
* **Body:**

```json
{
  "user_id": "{{user_id}}",
  "depot_id": "{{depot_id}}",
  "equipment": "50KG",
  "qty": 5,
  "movement": "COLLECTION"
}
```

**Expected:** Status 200, inventory decreases, audit entry created.

---

### 5.2 Empty Return

* **Method:** POST
* **URL:** same as above
* **Body:**

```json
{
  "user_id": "{{user_id}}",
  "depot_id": "{{depot_id}}",
  "equipment": "50KG",
  "qty": 3,
  "movement": "EMPTY_RETURN"
}
```

**Expected:** Status 200, inventory increases, audit entry created.

---

## 6️⃣ TRANSACTIONS — BILLING

### 6.1 Create Transaction

* **Method:** POST
* **URL:** `{{base_url}}/transactions/`
* **Body:**

```json
{
  "customer_id": "{{customer_id}}",
  "user_id": "{{user_id}}",
  "total": "250.00",
  "paid": false
}
```

**Expected:** Status 201, save `id` as `{{transaction_id}}`.

---

### 6.2 Record Meter Sale

* **Method:** POST
* **URL:** `{{base_url}}/transactions/meter_sale/`
* **Body:**

```json
{
  "transaction_id": "{{transaction_id}}",
  "last": "0.00",
  "latest": "5.00",
  "qty": "5.00",
  "rate": "50.00",
  "subtotal": "250.00"
}
```

**Expected:** Status 201, audit log entry created.

---

## 7️⃣ AUDIT LOG

* **Method:** GET
* **URL:** `{{base_url}}/audit/`
* **Check:** Status 200, logs include Distribution & Transaction actions.

---

## 8️⃣ REPORTS — FINANCE (READ-ONLY)

### 8.1 Summary

* **Method:** GET
* **URL:** `{{base_url}}/reports/summary/`
* **Check:** Status 200, JSON contains `total`.

---

### 8.2 Aging

* **Method:** GET
* **URL:** `{{base_url}}/reports/aging/`
* **Check:** Status 200, keys include `["30","60","90"]`.

---

## 9️⃣ FULL SEQUENCE TEST

1. Authenticate as Admin → get JWT.
2. Create Sales user.
3. Create Customer.
4. Verify Depots & Inventory.
5. Execute Distribution (Collection & Empty Return).
6. Create Transaction & Record Meter Sale.
7. Check Audit logs.
8. Retrieve Reports (Summary & Aging).

> ✅ Ensures **end-to-end functionality**, **business rules**, and **audit compliance**.

---

### ✅ Features

* Step numbers & descriptive titles
* Clear purpose & expected results
* Dynamic environment variables: `access_token`, `user_id`, `customer_id`, `depot_id`, `transaction_id`
* Test scripts to verify status codes, responses, and totals
* Fully importable into Postman for sequential verification


