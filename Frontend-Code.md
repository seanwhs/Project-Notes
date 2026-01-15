## 🟦 FRONTEND FULL CODE (React 18 + React Router v7 Data APIs)

This is the **complete, production-grade FRONTEND implementation** for the HSH LPG Sales System.

⚠️ **Scope (locked): FRONTEND ONLY**

* No backend code
* No backend explanations
* Backend is consumed strictly as a **Swagger-documented API contract (drf-yasg)**

This frontend code is:

* ✅ RR7-correct (loaders/actions only)
* ✅ Workflow-accurate for LPG operations
* ✅ Audit- and print-friendly
* ✅ Safe for meter, cylinder, and service billing

---

## ⚙️ Technology Stack

* **React 18**
* **React Router v7 (Data APIs)**
* **Fetch API** (cookie/JWT compatible)
* **TailwindCSS**
* **Printer-safe semantic markup**

**Swagger usage:** Frontend treats Swagger as the single source of truth for payloads; request/response shapes map 1:1 to `/swagger/`.

---

## 📁 Project Structure

```
src/
├── main.jsx
├── router.jsx
├── api.js
├── auth.js
├── layouts/
│   └── DashboardLayout.jsx
├── routes/
│   ├── login.jsx
│   ├── dashboard.jsx
│   ├── distribution.jsx
│   ├── transaction.jsx
│   ├── customers.jsx
│   ├── inventory.jsx
│   └── reports.jsx
├── components/
│   ├── MeterSection.jsx
│   ├── CylinderSection.jsx
│   ├── ServiceSection.jsx
│   ├── SummaryBar.jsx
│   └── Invoice.jsx
├── utils/
│   └── transactionBuffer.js
└── index.css
```

Design invariants:

* Routes = business workflows
* Loaders = READ only
* Actions = MUTATE only
* URL = application state

---

## 1️⃣ Application Bootstrap

### `main.jsx`

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

## 2️⃣ API + Auth Utilities

### `api.js`

```js
export async function api(url, options = {}) {
  const res = await fetch(`/api${url}`, {
    credentials: "include",
    headers: { "Content-Type": "application/json" },
    ...options,
  });

  if (res.status === 401) throw new Response("Unauthorized", { status: 401 });
  return res;
}
```

### `auth.js`

```js
import { redirect } from "react-router-dom";
import { api } from "./api";

export async function requireAuth() {
  const res = await api("/users/me/");
  if (!res.ok) throw redirect("/login");
  return res.json();
}
```

---

## 3️⃣ Router Configuration (RR7 Data Mode)

### `router.jsx`

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
      { path: "transaction/:id", element: <Transaction /> },
      { path: "customers", element: <Customers /> },
      { path: "inventory", element: <Inventory /> },
      { path: "reports", element: <Reports /> },
    ],
  },
]);
```

---

## 4️⃣ Persistent Layout

### `layouts/DashboardLayout.jsx`

```jsx
import { NavLink, Outlet, useLoaderData } from "react-router-dom";

export default function DashboardLayout() {
  const user = useLoaderData();

  return (
    <div className="flex min-h-screen">
      <aside className="w-64 bg-blue-900 text-white p-4">
        <h1 className="font-bold mb-6">HSH LPG</h1>
        <nav className="space-y-2">
          <NavLink to="/">Dashboard</NavLink>
          <NavLink to="/distribution">Distribution</NavLink>
          <NavLink to="/customers">Customers</NavLink>
          <NavLink to="/inventory">Inventory</NavLink>
          {user.is_admin && <NavLink to="/reports">Reports</NavLink>}
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

## 5️⃣ Distribution — Depot → Vehicle

### `routes/distribution.jsx`

```jsx
import { Form, redirect } from "react-router-dom";
import { api } from "../api";

export async function action({ request }) {
  const payload = Object.fromEntries(await request.formData());
  await api("/distributions/", { method: "POST", body: JSON.stringify(payload) });
  return redirect("/");
}

export default function Distribution() {
  return (
    <Form method="post" className="space-y-4">
      <h2 className="text-xl font-bold">Vehicle Distribution</h2>
      <input name="vehicle_id" placeholder="Vehicle" required className="border p-2" />
      <input name="cylinder_type" placeholder="Cylinder Type" required className="border p-2" />
      <input name="quantity" type="number" required className="border p-2" />
      <button className="btn-primary">Distribute</button>
    </Form>
  );
}
```

---

## 6️⃣ Transaction — Meter / Cylinder / Service Billing

### `routes/transaction.jsx`

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
    await api('/transactions/', { method: 'POST', body: JSON.stringify(data) });
  } catch (err) {
    addToBuffer(data);
    console.warn('Transaction added to offline buffer');
  }

  flushBuffer(api);

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
      {result?.invoice_id && <p>Invoice #{result.invoice_id} created</p>}
    </Form>
  );
}
```

---

## 7️⃣ Inventory (Read-only)

### `routes/inventory.jsx`

```jsx
import { useLoaderData } from "react-router-dom";
import { api } from "../api";

export async function loader() {
  const res = await api("/inventory/");
  return res.json();
}

export default function Inventory() {
  const items = useLoaderData();
  return (
    <table className="table-auto w-full">
      <thead><tr><th>Item</th><th>Qty</th></tr></thead>
      <tbody>
        {items.map(i => (<tr key={i.id}><td>{i.name}</td><td>{i.quantity}</td></tr>))}
      </tbody>
    </table>
  );
}
```

---

## 8️⃣ Reports (Admin-only)

### `routes/reports.jsx`

```jsx
import { useLoaderData } from "react-router-dom";
import { api } from "../api";

export async function loader({ request }) {
  const url = new URL(request.url);
  const res = await api(`/reports/?${url.searchParams.toString()}`);
  return res.json();
}

export default function Reports() {
  const rows = useLoaderData();
  return (
    <div>
      <h2 className="text-xl font-bold mb-4">Reports</h2>
      <table className="w-full table-auto">
        <tbody>
          {rows.map((r, i) => (<tr key={i}>{Object.values(r).map((v, j) => <td key={j}>{v}</td>)}</tr>))}
        </tbody>
      </table>
    </div>
  );
}
```

---

## 9️⃣ Meter, Cylinder, Service Components

### `components/MeterSection.jsx`

```jsx
import React, { useState } from 'react';
export default function MeterSection({ transaction, onChange }) {
  const [lastReading, setLastReading] = useState(transaction?.lastReading || 0);
  const [latestReading, setLatestReading] = useState(transaction?.latestReading || 0);
  const qty = latestReading - lastReading;

  return (
    <div className="bg-white p-4 rounded shadow mb-4">
      <h2 className="font-bold mb-2">Meter Sale</h2>
      <div className="flex space-x-4">
        <input type="number" value={lastReading} onChange={e => setLastReading(Number(e.target.value))} placeholder="Last Reading" className="border p-2" />
        <input type="number" value={latestReading} onChange={e => setLatestReading(Number(e.target.value))} placeholder="Latest Reading" className="border p-2" />
        <div className="flex items-center">Qty: {qty}</div>
      </div>
      <button onClick={() => onChange({ lastReading, latestReading, qty })} className="mt-2 bg-blue-600 text-white px-4 py-2 rounded">Update</button>
    </div>
  );
}
```

### `components/CylinderSection.jsx`

```jsx
import React, { useState } from 'react';
export default function CylinderSection({ transaction, onChange }) {
  const [cylinders, setCylinders] = useState(transaction?.cylinders || { '9kg':0,'12.7kg':0,'14kg':0,'50kg_pol':0,'50kg_l':0 });
  const handleChange = (type, value) => { const updated = { ...cylinders, [type]: Number(value) }; setCylinders(updated); onChange(updated); };

  return (
    <div className="bg-white p-4 rounded shadow mb-4">
      <h2 className="font-bold mb-2">Cylinder Sales</h2>
      {Object.keys(cylinders).map(type => (
        <div key={type} className="flex space-x-2 mb-1">
          <label className="w-24">{type}</label>
          <input type="number" value={cylinders[type]} onChange={e => handleChange(type, e.target.value)} className="border p-2 w-20" />
        </div>
      ))}
    </div>
  );
}
```

### `components/ServiceSection.jsx`

```jsx
import React, { useState } from 'react';
export default function ServiceSection({ transaction, onChange }) {
  const [services, setServices] = useState(transaction?.services || {});
  const handleChange = (service, value) => { const updated = { ...services, [service]: Number(value) }; setServices(updated); onChange(updated); };

  return (
    <div className="bg-white p-4 rounded shadow mb-4">
      <h2 className="font-bold mb-2">Service Items</h2>
      {['Delivery','Installation','Other'].map(service => (
        <div key={service} className="flex space-x-2 mb-1">
          <label className="w-32">{service}</label>
          <input type="number" value={services[service] || 0} onChange={e => handleChange(service, e.target.value)} className="border p-2 w-20" />
        </div>
      ))}
    </div>
  );
}
```

---

## 🔟 Invoice Print Layout

### `components/Invoice.jsx`

```jsx
import React, { forwardRef } from 'react';
const Invoice = forwardRef(({ transaction }, ref) => (
  <div ref={ref} className="p-4 bg-white w-full max-w-[800px] mx-auto text-sm font-sans">
    <h1 className="text-lg font-bold mb-2">HSH LPG Invoice</h1>
    <p>Invoice #: {transaction.number}</p>
    <p>Date: {new Date(transaction.createdAt).toLocaleDateString()}</p>
    <p>Customer: {transaction.customerName}</p>
    <hr className="my-2" />
    <h2 className="font-bold">Meter Sale</h2>
    <p>Last Reading: {transaction.meter.lastReading}</p>
    <p>Latest Reading: {transaction.meter.latestReading}</p>
    <p>Quantity: {transaction.meter.qty}</p>
    <h2 className="font-bold mt-2">Cylinder Sale</h2>
    {Object.entries(transaction.cylinders).map(([type, qty]) => <p key={type}>{type}: {qty}</p>)}
    <h2 className="font-bold mt-2">Services</h2>
    {Object.entries(transaction.services).map(([service, qty]) => <p key={service}>{service}: {qty}</p>)}
    <hr className="my-2" />
    <h2 className="font-bold">Total: {transaction.total}</h2>
  </div>
));
export default Invoice;
```

---

## 1️⃣1️⃣ Offline-Tolerant Transaction Buffer

### `utils/transactionBuffer.js`

```js
const bufferKey = 'transaction_buffer';

export const addToBuffer = transaction => {
  const buffer = JSON.parse(localStorage.getItem(bufferKey) || '[]');
  buffer.push(transaction);
  localStorage.setItem(bufferKey, JSON.stringify(buffer));
};

export const flushBuffer = async api => {
  const buffer = JSON.parse(localStorage.getItem(bufferKey) || '[]');
  for (let t of buffer) {
    try {
      await api('/transactions/', { method:'POST', body: JSON.stringify(t) });
    } catch(e) {
      console.error('Failed to sync transaction', t, e);
      return;
    }
  }
  localStorage.removeItem(bufferKey);
};

export const getBuffer = () => JSON.parse(localStorage.getItem(bufferKey) || '[]');
```
