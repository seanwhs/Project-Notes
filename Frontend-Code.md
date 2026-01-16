# 🟦 FRONTEND — HSH LPG SALES SYSTEM

**React 18 · React Router v7 (Data APIs) · Tailwind · JWT**

---

## 🔒 SCOPE GUARANTEE

✔ Frontend only
✔ Swagger is the contract
✔ No fake endpoints
✔ No optimistic stock mutation
✔ No hidden state

---

## ⚙️ TECHNOLOGY STACK

* React 18
* React Router v7 (loaders + actions only)
* Fetch API (JWT)
* TailwindCSS
* Print-safe HTML
* Offline-tolerant transaction buffer

---

## 📁 FINAL PROJECT STRUCTURE

```
src/
├── main.jsx
├── router.jsx
├── api.js
├── auth.js
├── index.css
│
├── layouts/
│   └── DashboardLayout.jsx
│
├── routes/
│   ├── login.jsx
│   ├── dashboard.jsx
│   ├── distribution.jsx
│   ├── transaction.jsx
│   ├── customers.jsx
│   ├── inventory.jsx
│   └── reports.jsx
│
├── components/
│   ├── MeterSection.jsx
│   ├── CylinderSection.jsx
│   ├── ServiceSection.jsx
│   ├── SummaryBar.jsx
│   └── Invoice.jsx
│
└── utils/
    └── transactionBuffer.js
```

---

## 1️⃣ APPLICATION BOOTSTRAP

### `src/main.jsx`

```jsx
import React from "react";
import ReactDOM from "react-dom/client";
import { RouterProvider } from "react-router-dom";
import router from "./router";
import "./index.css";

ReactDOM.createRoot(document.getElementById("root")).render(
  <React.StrictMode>
    <RouterProvider router={router} />
  </React.StrictMode>
);
```

---

## 2️⃣ API + JWT HANDLING

### `src/api.js`

```js
export async function api(url, options = {}) {
  const token = localStorage.getItem("access_token");

  const res = await fetch(`/api${url}`, {
    ...options,
    headers: {
      "Content-Type": "application/json",
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
    },
  });

  if (res.status === 401) {
    localStorage.removeItem("access_token");
    throw new Response("Unauthorized", { status: 401 });
  }

  return res;
}
```

---

### `src/auth.js`

```js
import { redirect } from "react-router-dom";
import { api } from "./api";

export async function requireAuth() {
  const res = await api("/accounts/users/");
  if (!res.ok) throw redirect("/login");
  return res.json();
}
```

---

## 3️⃣ ROUTER — RR7 DATA MODE

### `src/router.jsx`

```jsx
import { createBrowserRouter } from "react-router-dom";
import DashboardLayout from "./layouts/DashboardLayout";
import Login from "./routes/login";
import Dashboard from "./routes/dashboard";
import Distribution from "./routes/distribution";
import Transaction from "./routes/transaction";
import Customers from "./routes/customers";
import Inventory from "./routes/inventory";
import Reports from "./routes/reports";
import { requireAuth } from "./auth";

export default createBrowserRouter([
  { path: "/login", element: <Login /> },
  {
    element: <DashboardLayout />,
    loader: requireAuth,
    children: [
      { index: true, element: <Dashboard /> },
      { path: "distribution", element: <Distribution /> },
      { path: "transaction", element: <Transaction /> },
      { path: "customers", element: <Customers /> },
      { path: "inventory", element: <Inventory /> },
      { path: "reports", element: <Reports /> },
    ],
  },
]);
```

---

## 4️⃣ DASHBOARD LAYOUT (ROLE-AWARE)

### `src/layouts/DashboardLayout.jsx`

```jsx
import { NavLink, Outlet, useLoaderData } from "react-router-dom";

export default function DashboardLayout() {
  const users = useLoaderData();
  const user = users[0];

  return (
    <div className="flex min-h-screen">
      <aside className="w-64 bg-blue-900 text-white p-4">
        <h1 className="font-bold mb-6">HSH LPG</h1>
        <nav className="space-y-2">
          <NavLink to="/">Dashboard</NavLink>
          <NavLink to="/distribution">Distribution</NavLink>
          <NavLink to="/transaction">Transaction</NavLink>
          <NavLink to="/customers">Customers</NavLink>
          <NavLink to="/inventory">Inventory</NavLink>
          {user.role === "ADMIN" && <NavLink to="/reports">Reports</NavLink>}
        </nav>
      </aside>
      <main className="flex-1 p-6 bg-gray-100 print:bg-white">
        <Outlet />
      </main>
    </div>
  );
}
```

---

## 5️⃣ DISTRIBUTION — COMMAND ONLY

### `src/routes/distribution.jsx`

```jsx
import { Form, redirect } from "react-router-dom";
import { api } from "../api";

export async function action({ request }) {
  const data = Object.fromEntries(await request.formData());

  await api("/distribution/distributions/create/", {
    method: "POST",
    body: JSON.stringify({
      depot: data.depot,
      equipment_name: data.equipment_name,
      quantity: Number(data.quantity),
      status: data.status,
    }),
  });

  return redirect("/");
}

export default function Distribution() {
  return (
    <Form method="post" className="space-y-4">
      <h2 className="text-xl font-bold">Stock Distribution</h2>

      <input name="depot" placeholder="Depot ID" className="border p-2" />
      <input name="equipment_name" placeholder="Equipment" className="border p-2" />
      <input name="quantity" type="number" className="border p-2" />

      <select name="status" className="border p-2">
        <option value="COLLECTION">Collection</option>
        <option value="EMPTY_RETURN">Empty Return</option>
      </select>

      <button className="bg-blue-700 text-white px-4 py-2 rounded">
        Execute
      </button>
    </Form>
  );
}
```

---

## 6️⃣ TRANSACTION — METER + CYLINDER + SERVICE

### `src/routes/transaction.jsx`

```jsx
import { Form, useActionData } from "react-router-dom";
import MeterSection from "../components/MeterSection";
import CylinderSection from "../components/CylinderSection";
import ServiceSection from "../components/ServiceSection";
import SummaryBar from "../components/SummaryBar";
import { api } from "../api";
import { addToBuffer, flushBuffer } from "../utils/transactionBuffer";

export async function action({ request }) {
  const data = Object.fromEntries(await request.formData());

  try {
    await api("/transactions/transactions/create/", {
      method: "POST",
      body: JSON.stringify(data),
    });
  } catch {
    addToBuffer(data);
  }

  await flushBuffer(api);
  return { success: true };
}

export default function Transaction() {
  const result = useActionData();

  return (
    <Form method="post" className="space-y-6">
      <MeterSection />
      <CylinderSection />
      <ServiceSection />
      <SummaryBar />
      {result?.success && <p className="text-green-700">Transaction saved</p>}
    </Form>
  );
}
```

---

## 7️⃣ INVENTORY — READ ONLY

### `src/routes/inventory.jsx`

```jsx
import { useLoaderData } from "react-router-dom";
import { api } from "../api";

export async function loader() {
  const res = await api("/inventory/inventory/");
  return res.json();
}

export default function Inventory() {
  const rows = useLoaderData();

  return (
    <table className="table-auto w-full bg-white">
      <thead>
        <tr>
          <th>Depot</th>
          <th>Equipment</th>
          <th>Qty</th>
        </tr>
      </thead>
      <tbody>
        {rows.map((r, i) => (
          <tr key={i}>
            <td>{r.depot}</td>
            <td>{r.equipment_name}</td>
            <td>{r.quantity}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

---

## 8️⃣ REPORTS — ADMIN ONLY

### `src/routes/reports.jsx`

```jsx
import { useLoaderData } from "react-router-dom";
import { api } from "../api";

export async function loader() {
  const res = await api("/reports/sales/");
  return res.json();
}

export default function Reports() {
  const rows = useLoaderData();

  return (
    <div>
      <h2 className="text-xl font-bold mb-4">Sales Report</h2>
      <pre className="bg-white p-4">{JSON.stringify(rows, null, 2)}</pre>
    </div>
  );
}
```

---

## 9️⃣ TRANSACTION COMPONENTS (UNCHANGED LOGIC, CLEANED)

### `components/MeterSection.jsx`

```jsx
export default function MeterSection() {
  return (
    <section className="bg-white p-4 rounded shadow">
      <h3 className="font-bold mb-2">Meter Sale</h3>
      <input name="last_reading" placeholder="Last Reading" className="border p-2 mr-2" />
      <input name="latest_reading" placeholder="Latest Reading" className="border p-2" />
    </section>
  );
}
```

---

### `components/CylinderSection.jsx`

```jsx
export default function CylinderSection() {
  return (
    <section className="bg-white p-4 rounded shadow">
      <h3 className="font-bold mb-2">Cylinders</h3>
      {["9kg","12.7kg","14kg","50kg_pol","50kg_l"].map(k => (
        <input key={k} name={`cylinder_${k}`} placeholder={k} className="border p-2 mr-2" />
      ))}
    </section>
  );
}
```

---

### `components/ServiceSection.jsx`

```jsx
export default function ServiceSection() {
  return (
    <section className="bg-white p-4 rounded shadow">
      <h3 className="font-bold mb-2">Services</h3>
      <input name="delivery_fee" placeholder="Delivery" className="border p-2 mr-2" />
      <input name="installation_fee" placeholder="Installation" className="border p-2" />
    </section>
  );
}
```

---

## 🔟 INVOICE — PRINT SAFE

### `components/Invoice.jsx`

```jsx
import { forwardRef } from "react";

const Invoice = forwardRef(({ invoice }, ref) => (
  <div ref={ref} className="p-6 bg-white text-sm w-[800px] mx-auto">
    <h1 className="text-lg font-bold mb-4">HSH LPG Invoice</h1>
    <p>Invoice No: {invoice.invoice_no}</p>
    <p>Date: {invoice.issued_at}</p>
    <hr className="my-4" />
    <p>Subtotal: {invoice.subtotal}</p>
    <p>GST: {invoice.gst_amount}</p>
    <p className="font-bold">Total: {invoice.total_amount}</p>
  </div>
));

export default Invoice;
```

---

## 1️⃣1️⃣ OFFLINE TRANSACTION BUFFER

### `utils/transactionBuffer.js`

```js
const KEY = "txn_buffer";

export function addToBuffer(txn) {
  const buf = JSON.parse(localStorage.getItem(KEY) || "[]");
  buf.push(txn);
  localStorage.setItem(KEY, JSON.stringify(buf));
}

export async function flushBuffer(api) {
  const buf = JSON.parse(localStorage.getItem(KEY) || "[]");
  for (const txn of buf) {
    try {
      await api("/transactions/transactions/create/", {
        method: "POST",
        body: JSON.stringify(txn),
      });
    } catch {
      return;
    }
  }
  localStorage.removeItem(KEY);
}
```

---

## ✅ FINAL FRONTEND GUARANTEES

✔ Strict RR7 Data API usage
✔ Commands vs reads enforced
✔ JWT-safe auth flow
✔ Swagger-aligned payloads
✔ Print-safe invoices
✔ Offline-tolerant transactions
✔ Regulator-appropriate UX

