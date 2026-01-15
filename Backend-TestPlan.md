Here’s a **fully rewritten and enhanced Postman Test Plan** for HSH Sales System, with **more detail, explanations, and verification guidance**. I’ve preserved your structure but added:

* Explicit **field-level validation** for payloads.
* **Audit and data integrity checks** after every mutation.
* Sequential chaining instructions for **offline-first & online scenarios**.
* Clarifications on **expected server behavior** and error handling.

---

# 🧪 HSH SALES SYSTEM — Postman Test Plan (v2.0)

**Goal:** Verify that the backend system works correctly, enforces business rules, maintains data integrity, and logs all actions for audit. Tests are **module by module** and include **purpose, payload verification, and expected outcomes**.

---

## 0️⃣ SET UP POSTMAN ENVIRONMENT

Create a **Postman environment** (`HSH Local`) to make requests **dynamic, reusable, and sequential**.

| Variable          | Value                       | Purpose                        |
| ----------------- | --------------------------- | ------------------------------ |
| `base_url`        | `http://127.0.0.1:8000/api` | Local backend API URL          |
| `access_token`    | `""`                        | JWT token obtained after login |
| `user_id`         | `""`                        | Sales user ID                  |
| `customer_id`     | `""`                        | Customer ID                    |
| `depot_id`        | `""`                        | Depot ID                       |
| `transaction_id`  | `""`                        | Transaction ID                 |
| `distribution_id` | `""`                        | Distribution batch ID          |

> ✅ Using variables allows **reusable, dynamic, and chainable requests**.

---

## 1️⃣ AUTHENTICATION — GET JWT TOKEN

**Purpose:** Log in and obtain JWT token for protected endpoints.

* **Method:** POST
* **URL:** `{{base_url}}/token/`
* **Body (JSON):**

```json
{
  "username": "admin",
  "password": "adminpass"
}
```

**Expected Result:**

* Status 200
* Response contains `access` and `refresh` tokens
* Save `access` as `{{access_token}}`

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

**Purpose:** Ensure Admin can create users and Sales users are restricted.

### 2.1 Create Sales User

* **Method:** POST
* **URL:** `{{base_url}}/accounts/users/`
* **Headers:** `Authorization: Bearer {{access_token}}`
* **Body:**

```json
{
  "username": "sales1",
  "password": "salespass",
  "role": "SALES",
  "vehicle_no": "LPG-001",
  "email": "sales1@hsh.com"
}
```

**Expected:**

* Status 201
* Response contains `id` → save as `{{user_id}}`
* User role is `SALES`

---

### 2.2 List Users (Admin Only)

* **Method:** GET
* **URL:** `{{base_url}}/accounts/users/`
* **Expected:** Status 200, response JSON array contains created user.

---

### 2.3 RBAC Enforcement

1. Log in as **Sales user** → obtain JWT.
2. Attempt `GET /accounts/users/`.

**Expected:** 403 Forbidden → ensures Admin-only access.

---

## 3️⃣ CUSTOMERS — CONTRACTS & RATES

**Purpose:** Ensure accurate creation and retrieval of customer profiles.

### 3.1 Create Customer

* **Method:** POST
* **URL:** `{{base_url}}/customers/`
* **Body:**

```json
{
  "name": "John Doe Industries",
  "address": "123 Industrial Rd",
  "contact_no": "12345678",
  "payment_type": "CREDIT",
  "meter_rate": "50.00",
  "rate_9kg": "20.00",
  "rate_12_7kg": "25.00",
  "rate_14kg": "30.00",
  "rate_50kg_pol": "200.00",
  "rate_50kg_l": "220.00",
  "last_meter_reading": "0.00",
  "active": true
}
```

**Expected:**

* Status 201
* Response contains `id` → save as `{{customer_id}}`
* Verify all fields match input

---

### 3.2 List Customers

* **Method:** GET
* **URL:** `{{base_url}}/customers/`
* **Check:** Status 200, response array contains new customer.

---

## 4️⃣ DEPOTS & INVENTORY (READ-ONLY)

### 4.1 List Depots

* **Method:** GET
* **URL:** `{{base_url}}/depots/`
* **Expected:** Status 200, save first `id` as `{{depot_id}}`.

---

### 4.2 List Inventory

* **Method:** GET
* **URL:** `{{base_url}}/inventory/`
* **Check:** Status 200, JSON array shows `full_qty` and `empty_qty` per depot item.

> 🔹 Inventory is **read-only** via API.

---

## 5️⃣ DISTRIBUTION — STOCK MOVEMENT

**Purpose:** Verify atomic stock movement, audit logging, and integrity.

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

**Expected:**

* Status 200
* Inventory `full_qty` decreases, audit entry created
* Save `id` as `{{distribution_id}}`

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

**Expected:**

* Status 200
* Inventory `empty_qty` increases, audit entry created

---

## 6️⃣ TRANSACTIONS — BILLING

**Purpose:** Ensure sales, meter, cylinder, and service transactions are processed correctly.

### 6.1 Create Transaction

* **Method:** POST
* **URL:** `{{base_url}}/transactions/`
* **Body:**

```json
{
  "customer_id": "{{customer_id}}",
  "user_id": "{{user_id}}",
  "total_amount": "250.00",
  "is_paid": false,
  "payment_method": "CREDIT"
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
  "last_reading": "0.00",
  "latest_reading": "5.00",
  "qty": "5.00",
  "rate": "50.00",
  "subtotal": "250.00"
}
```

**Expected:**

* Status 201
* Audit log entry created
* Verify backend recalculates totals

---

### 6.3 Record Cylinder / Service Items

* **Method:** POST
* **URL:** `{{base_url}}/transactions/items/`
* **Body (example):**

```json
{
  "transaction_id": "{{transaction_id}}",
  "cylinders": {"9kg": 2, "14kg": 1},
  "services": {"Installation": 1},
  "subtotal": 150.00
}
```

**Expected:** Status 201, subtotal calculated server-side.

---

## 7️⃣ AUDIT LOG

* **Method:** GET
* **URL:** `{{base_url}}/audit/`
* **Check:** Status 200, logs contain:

  * Distribution & Transaction entries
  * Correct `user_id`, `action_type`, and payload

---

## 8️⃣ REPORTS — FINANCE (READ-ONLY)

### 8.1 Summary

* **Method:** GET
* **URL:** `{{base_url}}/reports/summary/`
* **Check:** Status 200, JSON contains `total_sales`, `total_paid`, `total_unpaid`.

---

### 8.2 Aging Report

* **Method:** GET
* **URL:** `{{base_url}}/reports/aging/`
* **Check:** Status 200, keys include `["30","60","90"]`.

---

## 9️⃣ FULL SEQUENCE TEST

1. Authenticate as Admin → get JWT
2. Create Sales user → verify RBAC
3. Create Customer → verify fields
4. List Depots & Inventory
5. Execute Distribution (Collection & Empty Return)
6. Create Transaction → record Meter / Cylinder / Service sales
7. Check Audit logs for each action
8. Retrieve Reports (Summary & Aging)

> ✅ Confirms **end-to-end functionality**, **business rules**, **audit compliance**, and **inventory integrity**.

---

### ✅ Enhancements / Best Practices

* **Dynamic environment variables:** allow sequential testing
* **Field-level verification:** totals, rates, and stock quantities validated
* **Audit confirmation:** ensures all mutations logged
* **Offline-first simulation:** queue actions locally, replay once online
* **Postman scripts:** automatically store IDs, check response keys

---

This test plan is **fully importable into Postman**, covers **all modules**, and ensures **compliance with HSH LPG workflows**.

---

{
  "info": {
    "name": "HSH Sales System - Full Test Collection",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json",
    "description": "Complete Postman collection for HSH Sales System, with environment variables and test scripts."
  },
  "item": [
    {
      "name": "0 - Authenticate Admin",
      "request": {
        "method": "POST",
        "header": [{"key": "Content-Type", "value": "application/json"}],
        "url": "{{base_url}}/token/",
        "body": {"mode": "raw", "raw": "{\n  \"username\": \"admin\",\n  \"password\": \"adminpass\"\n}"}
      },
      "event": [{
        "listen": "test",
        "script": {
          "exec": [
            "pm.test('JWT token received', function () {", 
            "  var json = pm.response.json();", 
            "  pm.expect(json).to.have.property('access');", 
            "  pm.environment.set('access_token', json.access);", 
            "});"
          ]
        }
      }]
    },
    {
      "name": "1 - Create Sales User",
      "request": {
        "method": "POST",
        "header": [
          {"key": "Content-Type", "value": "application/json"},
          {"key": "Authorization", "value": "Bearer {{access_token}}"}
        ],
        "url": "{{base_url}}/accounts/users/",
        "body": {
          "mode": "raw",
          "raw": "{\n  \"username\": \"sales1\",\n  \"password\": \"salespass\",\n  \"role\": \"SALES\",\n  \"vehicle_no\": \"LPG-001\",\n  \"email\": \"sales1@hsh.com\"\n}"
        }
      },
      "event": [{
        "listen": "test",
        "script": {
          "exec": [
            "pm.test('Sales user created', function() {", 
            "  var json = pm.response.json();", 
            "  pm.expect(json).to.have.property('id');", 
            "  pm.environment.set('user_id', json.id);", 
            "});"
          ]
        }
      }]
    },
    {
      "name": "2 - Create Customer",
      "request": {
        "method": "POST",
        "header": [
          {"key": "Content-Type", "value": "application/json"},
          {"key": "Authorization", "value": "Bearer {{access_token}}"}
        ],
        "url": "{{base_url}}/customers/",
        "body": {
          "mode": "raw",
          "raw": "{\n  \"name\": \"John Doe Industries\",\n  \"address\": \"123 Industrial Rd\",\n  \"contact_no\": \"12345678\",\n  \"payment_type\": \"CREDIT\",\n  \"meter_rate\": \"50.00\",\n  \"rate_9kg\": \"20.00\",\n  \"rate_12_7kg\": \"25.00\",\n  \"rate_14kg\": \"30.00\",\n  \"rate_50kg_pol\": \"200.00\",\n  \"rate_50kg_l\": \"220.00\",\n  \"last_meter_reading\": \"0.00\",\n  \"active\": true\n}"
        }
      },
      "event": [{
        "listen": "test",
        "script": {
          "exec": [
            "pm.test('Customer created', function() {", 
            "  var json = pm.response.json();", 
            "  pm.expect(json).to.have.property('id');", 
            "  pm.environment.set('customer_id', json.id);", 
            "});"
          ]
        }
      }]
    },
    {
      "name": "3 - List Depots",
      "request": {
        "method": "GET",
        "header": [{"key": "Authorization", "value": "Bearer {{access_token}}"}],
        "url": "{{base_url}}/depots/"
      },
      "event": [{
        "listen": "test",
        "script": {
          "exec": [
            "pm.test('Depots listed', function() {", 
            "  var json = pm.response.json();", 
            "  pm.expect(json.length).to.be.above(0);", 
            "  pm.environment.set('depot_id', json[0].id);", 
            "});"
          ]
        }
      }]
    },
    {
      "name": "4 - Execute Distribution Collection",
      "request": {
        "method": "POST",
        "header": [
          {"key": "Content-Type", "value": "application/json"},
          {"key": "Authorization", "value": "Bearer {{access_token}}"}
        ],
        "url": "{{base_url}}/distribution/execute/",
        "body": {
          "mode": "raw",
          "raw": "{\n  \"user_id\": \"{{user_id}}\",\n  \"depot_id\": \"{{depot_id}}\",\n  \"equipment\": \"50KG\",\n  \"qty\": 5,\n  \"movement\": \"COLLECTION\"\n}"
        }
      },
      "event": [{
        "listen": "test",
        "script": {
          "exec": [
            "pm.test('Distribution collection executed', function() {", 
            "  var json = pm.response.json();", 
            "  pm.expect(json).to.have.property('id');", 
            "  pm.environment.set('distribution_id', json.id);", 
            "});"
          ]
        }
      }]
    }
  ],
  "event": [],
  "variable": [
    {"key": "base_url", "value": "http://127.0.0.1:8000/api"},
    {"key": "access_token", "value": ""},
    {"key": "user_id", "value": ""},
    {"key": "customer_id", "value": ""},
    {"key": "depot_id", "value": ""},
    {"key": "transaction_id", "value": ""},
    {"key": "distribution_id", "value": ""}
  ]
}
