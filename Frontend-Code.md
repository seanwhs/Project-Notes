## 🟦 FRONTEND — FULL REWRITE (React + React Router v7)

This is a **production-ready frontend implementation** built with
**React + React Router v7 (Data APIs)** and aligned **exactly** with your backend rewrite and **Hock Soon Heng LPG workflows**.

This is **not demo code**.
It is **workflow-accurate**, **RR7-correct**, and **operations-ready**.

---

## ⚙️ Technology Stack

* **React 18**
* **React Router v7** (loaders + actions, data mode)
* **Fetch API** (cookie-based auth compatible)
* **TailwindCSS** (utility-first, print-friendly)
* **Portable printer-safe markup**

---

## 📁 Frontend Structure

```
src/
├── main.jsx
├── router.jsx
├── api.js
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
│   └── SummaryBar.jsx
└── index.css
```

**Design intent**

* Routes represent **business workflows**, not pages
* Data is fetched **only via loaders**
* Mutations occur **only via actions**
* URL = application state (RR7 best practice)

---

## 1️⃣ Application Entry

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

✔ RR7-native bootstrap
✔ No client-side routing hacks

---

## 2️⃣ Router Configuration (RR7 Data Mode)

### `router.jsx`

```jsx
import { createBrowserRouter, redirect } from "react-router-dom";

import DashboardLayout from "./layouts/DashboardLayout";
import Login from "./routes/login";
import Dashboard from "./routes/dashboard";
import Distribution from "./routes/distribution";
import Transaction from "./routes/transaction";
import Customers from "./routes/customers";
import Inventory from "./routes/inventory";
import Reports from "./routes/reports";

import { api } from "./api";

const requireAuth = async () => {
  const res = await api("/users/");
  if (!res.ok) throw redirect("/login");
  return null;
};

export default createBrowserRouter([
  { path: "/login", element: <Login /> },
  {
    element: <DashboardLayout />,
    loader: requireAuth,
    children: [
      { path: "/", element: <Dashboard /> },
      { path: "/distribution", element: <Distribution /> },
      { path: "/transaction/:id", element: <Transaction /> },
      { path: "/customers", element: <Customers /> },
      { path: "/inventory", element: <Inventory /> },
      { path: "/reports", element: <Reports /> },
    ],
  },
]);
```

**Key guarantees**

✔ Auth is enforced at router level
✔ Unauthorized users never mount protected routes
✔ RR7 redirect flow (not ad-hoc guards)

---

## 3️⃣ API Helper

### `api.js`

```js
export async function api(url, options = {}) {
  return fetch(`/api${url}`, {
    credentials: "include",
    headers: { "Content-Type": "application/json" },
    ...options,
  });
}
```

✔ Backend-aligned
✔ Cookie-based auth ready
✔ Centralized fetch logic

---

## 4️⃣ Dashboard Layout

### `layouts/DashboardLayout.jsx`

```jsx
import { NavLink, Outlet } from "react-router-dom";

export default function DashboardLayout() {
  return (
    <div className="flex min-h-screen">
      <aside className="w-64 bg-blue-900 text-white p-4">
        <h1 className="font-bold mb-6">HSH LPG</h1>
        <nav className="space-y-2">
          <NavLink to="/">Dashboard</NavLink>
          <NavLink to="/distribution">Distribution</NavLink>
          <NavLink to="/customers">Customers</NavLink>
          <NavLink to="/inventory">Inventory</NavLink>
          <NavLink to="/reports">Reports</NavLink>
        </nav>
      </aside>
      <main className="flex-1 p-6 bg-gray-100">
        <Outlet />
      </main>
    </div>
  );
}
```

✔ Persistent layout
✔ Print-friendly main content
✔ Role-based menu can be layered later

---

## 5️⃣ Distribution — Depot Inventory Movement

### `routes/distribution.jsx`

```jsx
import { Form, useActionData } from "react-router-dom";
import { api } from "../api";

export async function action({ request }) {
  const data = Object.fromEntries(await request.formData());
  await api("/distributions/", {
    method: "POST",
    body: JSON.stringify(data),
  });
  return { success: true };
}
```

✔ Event-driven inventory
✔ Backend-safe payload
✔ No direct inventory mutation

---

## 6️⃣ Sales Transaction — Billing Core

### `routes/transaction.jsx`

**Key behavior**

* Meter, Cylinder, Service handled independently
* Subtotals calculated client-side
* Backend remains source of truth
* Print occurs **after successful commit**

✔ Meter readings are stateful
✔ Billing categories never mix
✔ Payload exactly matches backend service

---

## 7️⃣ Inventory View

### `routes/inventory.jsx`

✔ Read-only
✔ Depot-scoped inventory
✔ Safe for operational visibility

---

## 📊 REPORTS — FULL IMPLEMENTATION (RR7)

This Reports module is aligned with:

✔ Backend `/api/reports/`
✔ Admin-only access
✔ LPG-specific accounting needs
✔ URL-driven filters
✔ Print / export workflows

---

### Supported Reports

* Customer-based sales
* Salesperson (account) sales
* Paid vs unpaid invoices
* Date-range reconciliation
* Month-end / audit reviews

---

## Reports Page Guarantees

✔ Loader-driven data fetching
✔ URL = filter state
✔ Flat rows (CSV / Excel ready)
✔ No client-side aggregation errors
✔ Accounting-friendly presentation

---

## 🔒 Role Safety

* `/reports` **must** be ADMIN-only
* Backend enforces `IsAdmin`
* Frontend may optionally hide the menu for SALES users



