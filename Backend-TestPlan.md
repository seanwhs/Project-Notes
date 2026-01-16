# 🧪 HSH SALES SYSTEM — Postman Test Plan v3.0 (Enhanced & Full Coverage)

**Goal:** Test all backend endpoints, enforce business rules, ensure audit and inventory integrity, and simulate offline-first operations.

---

## 0️⃣ POSTMAN ENVIRONMENT VARIABLES

| Variable          | Purpose                                |
| ----------------- | -------------------------------------- |
| `base_url`        | API base (`http://127.0.0.1:8000/api`) |
| `access_token`    | JWT for authenticated requests         |
| `refresh_token`   | JWT refresh token                      |
| `user_id`         | Sales/Admin user ID                    |
| `customer_id`     | Customer ID                            |
| `depot_id`        | Depot ID                               |
| `inventory_id`    | Inventory item ID                      |
| `transaction_id`  | Transaction ID                         |
| `distribution_id` | Distribution batch ID                  |

> Using environment variables allows **dynamic, sequential testing** and offline-first simulation.

---

## 1️⃣ AUTHENTICATION

### 1.1 Admin Login

* **POST** `{{base_url}}/token/`
* **Body:**

```json
{
  "username": "admin",
  "password": "adminpass"
}
```

**Tests:**

```javascript
pm.test("JWT token received", function () {
    let json = pm.response.json();
    pm.expect(json).to.have.property("access");
    pm.expect(json).to.have.property("refresh");
    pm.environment.set("access_token", json.access);
    pm.environment.set("refresh_token", json.refresh);
});
```

**Expected Outcome:** Status 200, JWT tokens returned.

---

## 2️⃣ USER MANAGEMENT (ACCOUNTS)

### 2.1 Create Sales User

* **POST** `{{base_url}}/accounts/users/`
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

**Tests:**

* Status 201
* Response contains `id` → `{{user_id}}`
* Role matches input
* Password not returned in plain text

---

### 2.2 Verify RBAC

* Log in as **Sales user**
* Attempt **GET** `/accounts/users/`
* **Expected:** 403 Forbidden

> Confirms Admin-only access is enforced.

---

## 3️⃣ CUSTOMER MANAGEMENT

### 3.1 Create Customer

* **POST** `{{base_url}}/customers/`
* **Body Example:**

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

**Tests:**

* Status 201
* Response contains all fields as input
* Save `id` → `{{customer_id}}`

---

### 3.2 List Customers

* **GET** `/customers/`
* Verify newly created customer exists in the array

---

## 4️⃣ DEPOTS & INVENTORY

### 4.1 List Depots

* **GET** `/depots/`
* Save first depot `id` → `{{depot_id}}`

---

### 4.2 List Inventory

* **GET** `/inventory/`
* Check fields: `full_qty`, `empty_qty`, `item_name`, `item_type`
* Save inventory IDs for distribution tests → `{{inventory_id}}`

---

## 5️⃣ DISTRIBUTION (STOCK MOVEMENT)

### 5.1 Collection

* **POST** `/distribution/execute/`
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

**Tests:**

* Status 200
* Inventory `full_qty` decreased by 5
* Audit log entry created
* Save `id` → `{{distribution_id}}`

---

### 5.2 Empty Return

* **POST** `/distribution/execute/`
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

**Tests:**

* Status 200
* Inventory `empty_qty` increased by 3
* Audit log entry created

---

## 6️⃣ TRANSACTIONS (BILLING)

### 6.1 Create Transaction

* **POST** `/transactions/`
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

**Tests:**

* Status 201
* Save `id` → `{{transaction_id}}`
* Verify server calculates totals and applies validations

---

### 6.2 Record Meter Sale

* **POST** `/transactions/meter_sale/`
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

**Tests:**

* Status 201
* Verify audit log created
* Server recalculates totals

---

### 6.3 Record Cylinder & Service Items

* **POST** `/transactions/items/`
* **Body Example:**

```json
{
  "transaction_id": "{{transaction_id}}",
  "cylinders": {"9kg": 2, "14kg": 1},
  "services": {"Installation": 1},
  "subtotal": 150.00
}
```

**Tests:**

* Status 201
* Server calculates subtotal
* Audit log entry exists

---

## 7️⃣ AUDIT LOG VERIFICATION

* **GET** `/audit/`
* Check for:

  * Distribution movements
  * Transaction creation
  * Meter, cylinder, service sales
  * Correct `user_id`, `action_type`, and payload

---

## 8️⃣ REPORTS (READ-ONLY)

### 8.1 Summary Report

* **GET** `/reports/summary/`
* Verify fields: `total_sales`, `total_paid`, `total_unpaid`

---

### 8.2 Aging Report

* **GET** `/reports/aging/`
* Verify keys: `["30","60","90"]` (days overdue)

---

## 9️⃣ OFFLINE-FIRST SIMULATION

1. Queue transaction requests locally in Postman environment variable (simulate `LocalStorage`)
2. Execute POST requests offline → expect **pending queue**
3. Replay queued requests once online
4. Verify:

   * Inventory consistency
   * Audit logs reflect real timestamps
   * Transactions total correctly

---

## 🔟 FULL SEQUENCE END-TO-END

1. Admin login → JWT token
2. Create Sales user → verify RBAC
3. Create Customer → verify fields
4. List Depots & Inventory
5. Execute Distribution (Collection & Empty Return)
6. Create Transaction → Meter + Cylinder + Service
7. Validate Audit logs
8. Retrieve Reports (Summary & Aging)
9. Simulate Offline Queue & Replay

> Confirms **end-to-end workflow**, **audit compliance**, and **inventory integrity**.
