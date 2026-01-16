# 🏗️ HSH SALES SYSTEM — FULL-STACK CHEAT SHEET v3.2 (MySQL Edition)

---

## 1️⃣ Project Structure — Detailed

```
hsh-sales-system/
├── backend/           # Django REST API + business logic
│   ├── accounts/      # Users, JWT Auth, RBAC
│   ├── customers/     # Customer info & pricing
│   ├── inventory/     # Cylinder stock tracking
│   ├── distribution/  # Collection & empty return logs
│   ├── transactions/  # Sales processing
│   ├── billing/       # PDF invoices & email services
│   ├── reports/       # Admin transaction reports
│   ├── audit/         # Action logs
│   └── manage.py
├── frontend/          # React SPA (Vite + Router v7)
│   ├── routes/        # Pages: Dashboard, Customers, Inventory, Transactions, Delivery
│   ├── hooks/         # usePrinter, OfflineService, utilities
│   ├── services/      # API & offline helpers
│   └── root.jsx       # Root layout + Outlet
├── docker-compose.yml
└── README.md
```

---

## 2️⃣ Backend REST API — Endpoints & Examples

### Accounts

| Endpoint      | Method | Auth      | Description     |
| ------------- | ------ | --------- | --------------- |
| `/api/users/` | GET    | Admin JWT | List all users  |
| `/api/users/` | POST   | Admin JWT | Create new user |

**POST Example**

```json
{
  "username": "john",
  "email": "john@example.com",
  "password": "pass123",
  "role": "sales",
  "vehicle_no": "SG1234A"
}
```

**Response**

```json
{
  "id": 1,
  "username": "john",
  "email": "john@example.com",
  "role": "sales",
  "vehicle_no": "SG1234A"
}
```

---

### Customers

| Endpoint          | Method | Auth | Notes            |
| ----------------- | ------ | ---- | ---------------- |
| `/api/customers/` | GET    | JWT  | List customers   |
| `/api/customers/` | POST   | JWT  | Add new customer |

**POST Example**

```json
{
  "name": "Acme Industries",
  "payment_type": "monthly",
  "rate_14kg": 50.0,
  "rate_50kg": 180.0
}
```

---

### Inventory

| Endpoint          | Method | Auth      | Notes            |
| ----------------- | ------ | --------- | ---------------- |
| `/api/inventory/` | GET    | JWT       | List stock items |
| `/api/inventory/` | POST   | Admin JWT | Add/adjust stock |

---

### Transactions

| Endpoint                       | Method | Auth | Notes                  |
| ------------------------------ | ------ | ---- | ---------------------- |
| `/api/transactions/`           | GET    | JWT  | List all transactions  |
| `/api/transactions/create_tx/` | POST   | JWT  | Create new transaction |

**POST Example**

```json
{
  "customer_id": 2,
  "qty_14": 3,
  "qty_50": 1
}
```

**Response**

```json
{
  "id": 12,
  "customer": 2,
  "user": 1,
  "total_amount": 330.0,
  "is_paid": false,
  "created_at": "2026-01-14T10:00:00Z"
}
```

---

### Distribution

| Endpoint                           | Method | Auth | Notes                            |
| ---------------------------------- | ------ | ---- | -------------------------------- |
| `/api/distributions/`              | GET    | JWT  | List distributions               |
| `/api/distributions/create_batch/` | POST   | JWT  | Batch create (collection/return) |

**POST Example**

```json
{
  "items": [
    { "depot": "Main", "equipment": "CYL 14", "quantity": 5, "status": "collection" },
    { "depot": "Main", "equipment": "CYL 50", "quantity": 2, "status": "empty_return" }
  ]
}
```

**Response**

```json
{
  "distribution_no": "DIST-AB12CD34"
}
```

---

### Audit

| Endpoint      | Method | Auth | Notes           |
| ------------- | ------ | ---- | --------------- |
| `/api/audit/` | GET    | JWT  | List audit logs |

---

## 3️⃣ Frontend — React Router v7 Integration

### root.jsx

```jsx
import { Outlet } from "react-router";

export default function Root() {
  return (
    <div className="min-h-screen bg-gray-100">
      <header className="bg-blue-700 text-white p-4 text-xl font-bold">
        HSH Sales System
      </header>
      <main className="p-4">
        <Outlet />
      </main>
    </div>
  );
}
```

---

### Offline Service

```javascript
export const OfflineService = {
  save: (key, data) => localStorage.setItem(key, JSON.stringify(data)),
  load: (key) => JSON.parse(localStorage.getItem(key) || "null"),
  remove: (key) => localStorage.removeItem(key),
};
```

> 💡 Example: `OfflineService.save('delivery', formData)`

---

### Printer Hook

```javascript
export function usePrinter() {
  const printReceipt = (data) => {
    console.log("Printing receipt:", data);
    // ESC/POS integration over Web Bluetooth
  };
  return { printReceipt };
}
```

---

### Loader & Action Helpers

```javascript
export async function loader(endpoint) {
  try {
    const res = await fetch(endpoint);
    return res.json();
  } catch (err) {
    console.error("Loader error:", err);
    return [];
  }
}

export async function action(endpoint, formData) {
  try {
    const res = await fetch(endpoint, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(formData),
    });
    return res.json();
  } catch (err) {
    console.error("Action error:", err);
    return { error: err.message };
  }
}
```

> ✅ Reusable in all pages (`Customers.jsx`, `Transactions.jsx`, `Delivery.jsx`)

---

### Example — Delivery.jsx

```jsx
import { Form, useActionData, useNavigation } from "react-router";
import { usePrinter } from "../hooks/usePrinter";
import { OfflineService } from "../hooks/OfflineService";

export async function action({ request }) {
  const data = Object.fromEntries(await request.formData());
  OfflineService.save('delivery', data); // queue offline if needed
  return fetch("/api/distributions/create_batch/", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ items: [data] }),
  }).then(res => res.json());
}

export default function Delivery() {
  const actionData = useActionData();
  const navigation = useNavigation();
  const { printReceipt } = usePrinter();
  const isSubmitting = navigation.state === "submitting";

  return (
    <div>
      <h2 className="text-2xl font-bold mb-4">Delivery</h2>
      <Form method="post" className="space-y-2">
        <input name="depot" placeholder="Depot" required className="p-2 border"/>
        <input name="equipment" placeholder="Equipment" required className="p-2 border"/>
        <input name="quantity" type="number" placeholder="Quantity" required className="p-2 border"/>
        <select name="status" className="p-2 border">
          <option value="collection">Collection</option>
          <option value="empty_return">Empty Return</option>
        </select>
        <button type="submit" disabled={isSubmitting} className="bg-blue-600 text-white p-2 rounded">
          {isSubmitting ? "Saving..." : "Save"}
        </button>
      </Form>

      {actionData?.distribution_no && (
        <div className="text-green-600 mt-2">
          Distribution Saved: {actionData.distribution_no}
        </div>
      )}

      <button
        type="button"
        className="mt-4 bg-gray-600 text-white p-2 rounded"
        onClick={() => printReceipt(actionData)}
      >
        Print Receipt
      </button>
    </div>
  );
}
```

---

## 4️⃣ Transaction Lifecycle (Frontend ↔ Backend)

1. User fills **form** → optionally saved offline.
2. React Router **action** posts data to backend API.
3. Backend validates user, updates inventory, creates transaction/distribution.
4. Backend generates **PDF invoice + email**.
5. Backend logs **AuditLog** entry.
6. Frontend receives response → displays confirmation → prints receipt.

---

## 5️⃣ Docker Deployment Overview

```
┌─────────┐   ┌─────────┐   ┌─────────┐
│Frontend │   │Backend  │   │MySQL DB │
└─────────┘   └─────────┘   └─────────┘
Ports: 5173   Ports: 8000   Ports: 3306
Volumes:
./frontend    ./backend     mysql_data
:/app/frontend:/app/backend /var/lib/mysql
```

---

## 6️⃣ Key Features — v3.2

* Fully **decoupled frontend & backend**
* **Offline-first SPA** with auto-sync queue
* **Bluetooth thermal printing** (ESC/POS)
* **PDF invoice generation + automated email**
* **Audit logging** for compliance
* **Role-based access** (Admin / Sales / Supervisor)
* **Dockerized deployment** with persistent MySQL
* **React Router v7 loaders/actions** fully integrated
* **Mobile-first UI**, optimized for tablets & phones
* **Queue persistence** with `OfflineService` for network flapping
* **Consistent error handling** and user-friendly messages

---

# 🗄️ HSH SALES SYSTEM — MySQL ERD (v3.2)

### Tables & Fields

#### users

| Field      | Type                               | PK/FK | Notes                                                 |
| ---------- | ---------------------------------- | ----- | ----------------------------------------------------- |
| id         | INT AUTO_INCREMENT                 | PK    | Primary Key                                           |
| username   | VARCHAR(50)                        |       | Unique                                                |
| email      | VARCHAR(100)                       |       | Unique                                                |
| password   | VARCHAR(128)                       |       | Hashed                                                |
| role       | ENUM('admin','sales','supervisor') |       | RBAC                                                  |
| vehicle_no | VARCHAR(20)                        |       | Optional                                              |
| created_at | TIMESTAMP                          |       | Default CURRENT_TIMESTAMP                             |
| updated_at | TIMESTAMP                          |       | Default CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP |

#### customers

| Field        | Type                   | PK/FK | Notes                                                 |
| ------------ | ---------------------- | ----- | ----------------------------------------------------- |
| id           | INT AUTO_INCREMENT     | PK    | Primary Key                                           |
| name         | VARCHAR(100)           |       | Customer Name                                         |
| payment_type | ENUM('cash','monthly') |       | Billing method                                        |
| rate_14kg    | DECIMAL(10,2)          |       | Price per 14kg cylinder                               |
| rate_50kg    | DECIMAL(10,2)          |       | Price per 50kg cylinder                               |
| created_at   | TIMESTAMP              |       | Default CURRENT_TIMESTAMP                             |
| updated_at   | TIMESTAMP              |       | Default CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP |

#### inventory

| Field      | Type               | PK/FK | Notes                                                 |
| ---------- | ------------------ | ----- | ----------------------------------------------------- |
| id         | INT AUTO_INCREMENT | PK    | Primary Key                                           |
| equipment  | VARCHAR(50)        |       | CYL 14 / CYL 50                                       |
| depot      | VARCHAR(50)        |       | Depot name                                            |
| full_qty   | INT                |       | Full cylinders                                        |
| empty_qty  | INT                |       | Empty cylinders                                       |
| created_at | TIMESTAMP          |       | Default CURRENT_TIMESTAMP                             |
| updated_at | TIMESTAMP          |       | Default CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP |

#### transactions

| Field        | Type               | PK/FK | Notes                                                 |
| ------------ | ------------------ | ----- | ----------------------------------------------------- |
| id           | INT AUTO_INCREMENT | PK    | Primary Key                                           |
| customer_id  | INT                | FK    | → customers.id                                        |
| user_id      | INT                | FK    | → users.id                                            |
| qty_14       | INT                |       | Sold 14kg cylinders                                   |
| qty_50       | INT                |       | Sold 50kg cylinders                                   |
| total_amount | DECIMAL(10,2)      |       | Calculated total                                      |
| is_paid      | BOOLEAN            |       | Default FALSE                                         |
| created_at   | TIMESTAMP          |       | Default CURRENT_TIMESTAMP                             |
| updated_at   | TIMESTAMP          |       | Default CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP |

#### distributions

| Field           | Type                              | PK/FK | Notes                                                 |
| --------------- | --------------------------------- | ----- | ----------------------------------------------------- |
| id              | INT AUTO_INCREMENT                | PK    | Primary Key                                           |
| distribution_no | VARCHAR(20)                       |       | Auto-generated                                        |
| depot           | VARCHAR(50)                       |       | Depot name                                            |
| equipment       | VARCHAR(50)                       |       | Cylinder type                                         |
| quantity        | INT                               |       | Number of items                                       |
| status          | ENUM('collection','empty_return') |       | Distribution type                                     |
| created_at      | TIMESTAMP                         |       | Default CURRENT_TIMESTAMP                             |
| updated_at      | TIMESTAMP                         |       | Default CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP |

#### audit_logs

| Field     | Type               | PK/FK | Notes                     |
| --------- | ------------------ | ----- | ------------------------- |
| id        | INT AUTO_INCREMENT | PK    | Primary Key               |
| user_id   | INT                | FK    | → users.id                |
| action    | VARCHAR(50)        |       | e.g., CREATE_TRANSACTION  |
| entity    | VARCHAR(50)        |       | Table/entity affected     |
| timestamp | TIMESTAMP          |       | Default CURRENT_TIMESTAMP |

#### offline_transactions

| Field      | Type                            | PK/FK | Notes                                                 |
| ---------- | ------------------------------- | ----- | ----------------------------------------------------- |
| id         | INT AUTO_INCREMENT              | PK    | Primary Key                                           |
| payload    | JSON                            |       | Full transaction object                               |
| status     | ENUM('pending','sent','failed') |       | Queue state                                           |
| created_at | TIMESTAMP                       |       | Default CURRENT_TIMESTAMP                             |
| updated_at | TIMESTAMP                       |       | Default CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP |

---

### Relationships / Foreign Keys

* `transactions.customer_id → customers.id`
* `transactions.user_id → users.id`
* `audit_logs.user_id → users.id`

---

### ASCII ERD

```
┌──────────┐        ┌────────────┐        ┌─────────────┐
│  users   │        │ customers  │        │ inventory   │
│──────────│        │────────────│        │─────────────│
│ id PK    │◄───────│ id PK      │        │ id PK       │
│ username │        │ name       │        │ equipment   │
│ email    │        │ payment_type│       │ depot       │
│ password │        │ rate_14kg  │       │ full_qty    │
│ role     │        │ rate_50kg  │       │ empty_qty   │
│ vehicle_no│       │ created_at │       │ created_at  │
└──────────┘        └────────────┘       └─────────────┘
       ▲                     ▲
       │                     │
┌───────────────┐             │
│ transactions  │             │
│───────────────│             │
│ id PK         │             │
│ customer_id FK│─────────────┘
│ user_id FK    │
│ qty_14        │
│ qty_50        │
│ total_amount  │
│ is_paid       │
│ created_at    │
└───────────────┘
       │
       ▼
┌───────────────┐
│ distributions │
│───────────────│
│ id PK         │
│ distribution_no│
│ depot         │
│ equipment     │
│ quantity      │
│ status        │
│ created_at    │
└───────────────┘

┌───────────────┐
│ audit_logs    │
│───────────────│
│ id PK         │
│ user_id FK    │
│ action        │
│ entity        │
│ timestamp     │
└───────────────┘

┌────────────────────┐
│ offline_transactions│
│────────────────────│
│ id PK              │
│ payload JSON       │
│ status             │
│ created_at         │
└────────────────────┘
```

