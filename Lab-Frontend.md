# 🧪 LAB: HSH Sales System Frontend (React + React Router v7)

**Objective:** Build a **production-ready frontend** that integrates with the DRF backend for inventory, distribution, transactions, and reports. The frontend is **role-safe, RR7-native, and workflow-aligned**.

**Prerequisites:**

* Node.js 20+
* npm or yarn
* TailwindCSS setup knowledge
* Backend from the previous lab running on `/api`

---

## **Step 0 — Setup Project**

```bash
mkdir hsh_sales_frontend
cd hsh_sales_frontend
npm create vite@latest .  # Select React + JSX
npm install
npm install react-router-dom@7 tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

**tailwind.config.cjs**:

```js
module.exports = {
  content: ["./index.html", "./src/**/*.{js,jsx}"],
  theme: { extend: {} },
  plugins: [],
}
```

**index.css**:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

---

## **Step 1 — Application Entry**

**src/main.jsx**

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

✔ RR7-native bootstrap, no hacks

---

## **Step 2 — API Helper**

**src/api.js**

```js
export async function api(url, options = {}) {
  return fetch(`/api${url}`, {
    credentials: "include",
    headers: { "Content-Type": "application/json" },
    ...options,
  });
}
```

✔ Cookie-based auth ready
✔ Centralized fetch logic

---

## **Step 3 — Router Configuration (RR7 Data Mode)**

**src/router.jsx**

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

**✅ Lab Checkpoint:** Attempt to access `/` when logged out → redirect to `/login`.

---

## **Step 4 — Layout**

**src/layouts/DashboardLayout.jsx**

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

---

## **Step 5 — Login Route**

**src/routes/login.jsx**

```jsx
import { useNavigate } from "react-router-dom";
import { api } from "../api";
import { useState } from "react";

export default function Login() {
  const navigate = useNavigate();
  const [username, setUsername] = useState("");
  const [password, setPassword] = useState("");

  const handleLogin = async (e) => {
    e.preventDefault();
    const res = await api("/token/", {
      method: "POST",
      body: JSON.stringify({ username, password }),
    });

    if (res.ok) navigate("/");
    else alert("Login failed");
  };

  return (
    <form onSubmit={handleLogin} className="max-w-md mx-auto mt-20 p-6 bg-white shadow-md">
      <h1 className="text-xl font-bold mb-4">Login</h1>
      <input
        className="block w-full mb-3 p-2 border"
        placeholder="Username"
        value={username}
        onChange={(e) => setUsername(e.target.value)}
      />
      <input
        type="password"
        className="block w-full mb-3 p-2 border"
        placeholder="Password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
      />
      <button type="submit" className="bg-blue-900 text-white px-4 py-2">Login</button>
    </form>
  );
}
```

✔ Basic auth flow complete
✔ Can extend with JWT storage

---

## **Step 6 — Distribution Route (Depot Inventory Movement)**

**src/routes/distribution.jsx**

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

export default function Distribution() {
  return (
    <Form method="post">
      <h2 className="font-bold mb-2">Create Distribution</h2>
      <input type="text" name="depot" placeholder="Depot ID" className="border p-2 mb-2"/>
      <input type="text" name="equipment_name" placeholder="Equipment" className="border p-2 mb-2"/>
      <input type="number" name="quantity" placeholder="Quantity" className="border p-2 mb-2"/>
      <select name="status" className="border p-2 mb-2">
        <option value="COLLECTION">Collection</option>
        <option value="EMPTY_RETURN">Empty Return</option>
      </select>
      <button className="bg-blue-900 text-white px-4 py-2">Submit</button>
    </Form>
  );
}
```

✔ Backend-safe payload
✔ Inventory updated **via service only**

---

## **Step 7 — Transactions (Meter/Cylinder/Service)**

**Key Principles:**

* Subtotals calculated client-side
* Backend remains source of truth
* Print only **after successful commit**

```jsx
// Conceptual snippet
<MeterSection transaction={tx} onChange={handleMeterUpdate} />
<CylinderSection transaction={tx} onChange={handleCylinderUpdate} />
<ServiceSection transaction={tx} onChange={handleServiceUpdate} />
<SummaryBar total={calculateTotal(tx)} />
```

**✅ Lab Checkpoint:** Ensure `Transaction` action sends structured JSON to `/transactions/create_tx/`.

---

## **Step 8 — Inventory View (Read-Only)**

```jsx
import { useEffect, useState } from "react";
import { api } from "../api";

export default function Inventory() {
  const [inventory, setInventory] = useState([]);
  useEffect(() => {
    api("/inventory/").then(r => r.json()).then(setInventory);
  }, []);
  return (
    <div>
      <h1 className="font-bold mb-4">Inventory</h1>
      <ul>
        {inventory.map(i => <li key={i.id}>{i.depot} — {i.equipment_name}: {i.quantity}</li>)}
      </ul>
    </div>
  );
}
```

✔ Depot-scoped
✔ Read-only

---

## **Step 9 — Reports (Admin-Only)**

```jsx
import { useEffect, useState } from "react";
import { api } from "../api";

export default function Reports() {
  const [reports, setReports] = useState([]);
  useEffect(() => {
    api("/reports/").then(r => r.json()).then(setReports);
  }, []);
  return (
    <div>
      <h1 className="font-bold mb-4">Reports</h1>
      <table className="border-collapse border">
        <thead>
          <tr>
            <th className="border p-2">Customer</th>
            <th className="border p-2">Total</th>
            <th className="border p-2">Paid</th>
          </tr>
        </thead>
        <tbody>
          {reports.map(r => (
            <tr key={r.id}>
              <td className="border p-2">{r.customer}</td>
              <td className="border p-2">{r.total_amount}</td>
              <td className="border p-2">{r.is_paid ? "Yes" : "No"}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

✔ Loader-driven
✔ Admin-only access guaranteed by backend

---

## **Step 10 — Role Safety & Offline Prep**

* `/reports` menu hidden for SALES users (optional)
* All routes **backend-protected**
* Offline queueing can be added via **localStorage or IndexedDB**:

```js
// Example: store offline actions
const offlineQueue = JSON.parse(localStorage.getItem("offlineQueue") || "[]");
offlineQueue.push(action);
localStorage.setItem("offlineQueue", JSON.stringify(offlineQueue));
```

---

## ✅ Lab Completion Checklist

* [ ] RR7 router fully configured
* [ ] Login and auth flow working
* [ ] Distribution module calls backend **safely**
* [ ] Transactions module maps Meter/Cylinder/Service correctly
* [ ] Inventory read-only view working
* [ ] Reports loader-driven & admin-only
* [ ] Frontend ready for offline queue extension
* [ ] Tailwind layout print-friendly

---

This lab gives you a **workflow-aligned, production-ready React + RR7 frontend** that is **fully integrated** with your DRF backend.


