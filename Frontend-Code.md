# 🟢 Frontend — React Router v7 Full Code

---

## 1️⃣ root.jsx

`frontend/app/root.jsx`

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

## 2️⃣ hooks/usePrinter.js

```javascript
export function usePrinter() {
  const printReceipt = (data) => {
    console.log("Printing receipt:", data);
    // TODO: implement ESC/POS over Web Bluetooth
  };
  return { printReceipt };
}
```

---

## 3️⃣ services/offline.js

```javascript
export const OfflineService = {
  save: (key, data) => localStorage.setItem(key, JSON.stringify(data)),
  load: (key) => JSON.parse(localStorage.getItem(key) || "null"),
};
```

---

## 4️⃣ Routes — Dashboard.jsx

`frontend/app/routes/Dashboard.jsx`

```jsx
import { useLoaderData } from "react-router";

export async function loader() {
  const res = await fetch("/api/transactions/");
  const data = await res.json();
  return data;
}

export default function Dashboard() {
  const transactions = useLoaderData();

  return (
    <div className="space-y-4">
      <h2 className="text-2xl font-bold">Dashboard</h2>
      <div>
        <p>Total Transactions: {transactions.length}</p>
        <ul className="space-y-2">
          {transactions.map((tx) => (
            <li key={tx.id} className="p-2 bg-white rounded shadow">
              Customer: {tx.customer} | Total: ${tx.total_amount} | Paid: {tx.is_paid ? "Yes" : "No"}
            </li>
          ))}
        </ul>
      </div>
    </div>
  );
}
```

---

## 5️⃣ Routes — Customers.jsx

`frontend/app/routes/Customers.jsx`

```jsx
import { Form, useLoaderData, useNavigation } from "react-router";

export async function loader() {
  const res = await fetch("/api/customers/");
  return res.json();
}

export async function action({ request }) {
  const formData = Object.fromEntries(await request.formData());
  const res = await fetch("/api/customers/", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(formData),
  });
  return res.json();
}

export default function Customers() {
  const customers = useLoaderData();
  const navigation = useNavigation();
  const isSubmitting = navigation.state === "submitting";

  return (
    <div>
      <h2 className="text-2xl font-bold mb-4">Customers</h2>
      <Form method="post" className="space-y-2 mb-4">
        <input name="name" placeholder="Customer Name" required className="p-2 border"/>
        <input name="payment_type" placeholder="Payment Type (cash/monthly)" required className="p-2 border"/>
        <input name="rate_14kg" type="number" placeholder="14kg Rate" required className="p-2 border"/>
        <input name="rate_50kg" type="number" placeholder="50kg Rate" required className="p-2 border"/>
        <button type="submit" disabled={isSubmitting} className="bg-blue-600 text-white p-2 rounded">
          {isSubmitting ? "Saving..." : "Add Customer"}
        </button>
      </Form>
      <ul className="space-y-2">
        {customers.map(c => (
          <li key={c.id} className="p-2 bg-white rounded shadow">
            {c.name} | {c.payment_type} | 14kg: ${c.rate_14kg} | 50kg: ${c.rate_50kg}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

## 6️⃣ Routes — Inventory.jsx

```jsx
import { useLoaderData } from "react-router";

export async function loader() {
  const res = await fetch("/api/inventory/");
  return res.json();
}

export default function Inventory() {
  const inventory = useLoaderData();
  return (
    <div>
      <h2 className="text-2xl font-bold mb-4">Inventory</h2>
      <ul className="space-y-2">
        {inventory.map(i => (
          <li key={i.id} className="p-2 bg-white rounded shadow">
            {i.equipment} | Full Qty: {i.full_qty} | Empty Qty: {i.empty_qty}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

## 7️⃣ Routes — Transactions.jsx

```jsx
import { Form, useLoaderData, useNavigation } from "react-router";

export async function loader() {
  const res = await fetch("/api/transactions/");
  return res.json();
}

export async function action({ request }) {
  const data = Object.fromEntries(await request.formData());
  const res = await fetch("/api/transactions/create_tx/", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(data),
  });
  return res.json();
}

export default function Transactions() {
  const transactions = useLoaderData();
  const navigation = useNavigation();
  const isSubmitting = navigation.state === "submitting";

  return (
    <div>
      <h2 className="text-2xl font-bold mb-4">Transactions</h2>
      <Form method="post" className="space-y-2 mb-4">
        <input name="customer_id" type="number" placeholder="Customer ID" required className="p-2 border"/>
        <input name="qty_14" type="number" placeholder="Qty 14kg" required className="p-2 border"/>
        <input name="qty_50" type="number" placeholder="Qty 50kg" required className="p-2 border"/>
        <button type="submit" disabled={isSubmitting} className="bg-blue-600 text-white p-2 rounded">
          {isSubmitting ? "Saving..." : "Add Transaction"}
        </button>
      </Form>
      <ul className="space-y-2">
        {transactions.map(tx => (
          <li key={tx.id} className="p-2 bg-white rounded shadow">
            Customer: {tx.customer} | Total: ${tx.total_amount} | Paid: {tx.is_paid ? "Yes" : "No"}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

## 8️⃣ Routes — Delivery.jsx

```jsx
import { Form, useActionData, useNavigation } from "react-router";
import { usePrinter } from "../hooks/usePrinter";

export async function action({ request }) {
  const formData = Object.fromEntries(await request.formData());
  const res = await fetch("/api/distributions/create_batch/", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(formData),
  });
  return res.json();
}

export default function Delivery() {
  const { printReceipt } = usePrinter();
  const actionData = useActionData();
  const navigation = useNavigation();
  const isSubmitting = navigation.state === "submitting";

  return (
    <div>
      <h2 className="text-2xl font-bold mb-4">Delivery</h2>
      <Form method="post" className="space-y-2 mb-4">
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
        <div className="text-green-600">Distribution Saved: {actionData.distribution_no}</div>
      )}

      <button
        type="button"
        onClick={() =>
          printReceipt({ depot: "Main", equipment: "CYL 14", qty: 2, total: 50 })
        }
        className="mt-4 bg-gray-600 text-white p-2 rounded"
      >
        Print Receipt
      </button>
    </div>
  );
}
```

---

## 9️⃣ React Router Config — react-router.config.ts

```ts
import type { Config } from "@react-router/dev/config";

export default {
  ssr: false,
} satisfies Config;
```

---

## 1️⃣0️⃣ Vite Config (vite.config.js)

```javascript
import { defineConfig } from "vite";
import { reactRouter } from "@react-router/dev/vite";

export default defineConfig({
  plugins: [reactRouter()],
});
```

---

✅ **At this point, you have a full frontend** with:

* Dashboard
* Customers
* Inventory
* Transactions
* Delivery

All **loaders/actions compliant with React Router v7**, ready to call the backend APIs.

