# 🏗️ HSH SALES SYSTEM — FULL-STACK CHEAT SHEET v3.0 (MySQL Edition)

---

## 1️⃣ Project Structure — Detailed

```
hsh-sales-system/
├── backend/           # Django REST API + business logic
│   ├── accounts/      # Users, JWT Auth, RBAC
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   └── permissions.py
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
│   ├── hooks/         # usePrinter
│   ├── services/      # OfflineService
│   └── root.jsx
├── docker-compose.yml
└── README.md
```

---

## 2️⃣ Backend REST API — Endpoints & Examples

### Accounts

| Endpoint      | Method | Auth      | Description    |
| ------------- | ------ | --------- | -------------- |
| `/api/users/` | GET    | Admin JWT | List all users |
| `/api/users/` | POST   | Admin JWT | Create user    |

**Request Example (POST /api/users/)**

```json
{
  "username": "john",
  "email": "john@example.com",
  "password": "pass123",
  "role": "sales",
  "vehicle_no": "SG1234A"
}
```

**Response Example**

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

**Request Example**

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

| Endpoint          | Method | Auth      | Notes               |
| ----------------- | ------ | --------- | ------------------- |
| `/api/inventory/` | GET    | JWT       | List stock items    |
| `/api/inventory/` | POST   | Admin JWT | Add or adjust stock |

---

### Transactions

| Endpoint                       | Method | Auth | Notes                  |
| ------------------------------ | ------ | ---- | ---------------------- |
| `/api/transactions/`           | GET    | JWT  | List all transactions  |
| `/api/transactions/create_tx/` | POST   | JWT  | Create new transaction |

**Request Example**

```json
{
  "customer_id": 2,
  "qty_14": 3,
  "qty_50": 1
}
```

**Response Example**

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

**Request Example**

```json
{
  "items": [
    {
      "depot": "Main",
      "equipment": "CYL 14",
      "quantity": 5,
      "status": "collection"
    },
    {
      "depot": "Main",
      "equipment": "CYL 50",
      "quantity": 2,
      "status": "empty_return"
    }
  ]
}
```

**Response Example**

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

## 3️⃣ Frontend — React Router v7 — Full Integration

### 3.1 root.jsx

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

### 3.2 Offline Service

```javascript
export const OfflineService = {
  save: (key, data) => localStorage.setItem(key, JSON.stringify(data)),
  load: (key) => JSON.parse(localStorage.getItem(key) || "null"),
  remove: (key) => localStorage.removeItem(key),
};
```

*All forms can auto-save offline using `OfflineService.save('key', formData)`.*

---

### 3.3 Printer Hook

```javascript
export function usePrinter() {
  const printReceipt = (data) => {
    console.log("Printing receipt:", data);
    // Integrate with ESC/POS over Web Bluetooth
  };
  return { printReceipt };
}
```

---

### 3.4 Loader & Action Patterns

```javascript
// Generic Loader
export async function loader(endpoint) {
  try {
    const res = await fetch(endpoint);
    return res.json();
  } catch (err) {
    console.error("Loader error:", err);
    return [];
  }
}

// Generic Action
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

*All routes (`Customers.jsx`, `Transactions.jsx`, `Delivery.jsx`) reuse this loader/action pattern.*

---

### 3.5 Example — Delivery.jsx

```jsx
import { Form, useActionData, useNavigation } from "react-router";
import { usePrinter } from "../hooks/usePrinter";

export async function action({ request }) {
  const data = Object.fromEntries(await request.formData());
  return await fetch("/api/distributions/create_batch/", {
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

*Offline support:* before POST, call `OfflineService.save('delivery', formData)` to queue offline.

---

## 4️⃣ Transaction Lifecycle (Frontend ↔ Backend)

1. User fills **form** → offline queue save optional.
2. React Router **action** posts data to backend API.
3. Backend validates user, updates inventory, creates transaction/distribution.
4. Backend triggers **PDF invoice generation + email**.
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

## 6️⃣ Key Features — v3.0

* **Fully decoupled** frontend & backend
* **Offline-first SPA** with auto-sync queue
* **Bluetooth thermal printing** (ESC/POS)
* **PDF invoice generation + automated email**
* **Audit logging** for compliance
* **Role-based access** (Admin / Sales)
* **Dockerized deployment** with persistent MySQL
* **React Router v7 loaders/actions** fully integrated
* **Mobile-first UI**, optimized for tablets & phones
* **Loader/action helpers** for easy form + fetch reuse


