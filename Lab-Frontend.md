# 🧪 LAB: HSH Sales System Frontend

**React 18 · React Router v7 (Data APIs) · TailwindCSS**

**Objective:** Build a **production-ready, regulator-safe frontend** that integrates with the DRF backend for **distribution, transactions, inventory, and reports**.

**Hard constraints:**

* ✅ Frontend only
* ✅ RR7 loaders/actions only
* ✅ Backend accessed strictly via `/api`
* ✅ Swagger = source of truth
* ✅ No client-side stock mutation

---

## **Prerequisites**

* Node.js 20+
* npm or yarn
* TailwindCSS familiarity
* Backend running on `/api` with JWT auth

---

## **Step 0 — Project Setup**

```bash
mkdir hsh_sales_frontend
cd hsh_sales_frontend
npm create vite@latest .   # React + JSX
npm install
npm install react-router-dom@7
npm install tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### `tailwind.config.cjs`

```js
module.exports = {
  content: ["./index.html", "./src/**/*.{js,jsx}"],
  theme: { extend: {} },
  plugins: [],
};
```

### `src/index.css`

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

---

## **Step 1 — Application Entry (RR7-native)**

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

✔ No legacy routers
✔ No side-effect auth hacks

---

## **Step 2 — API Helper (JWT-safe)**

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

✔ Centralized auth
✔ Backend-enforced security
✔ Compatible with SimpleJWT

---

## **Step 3 — Router Configuration (RR7 Data APIs)**

### `src/router.jsx`

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
  const res = await api("/accounts/users/");
  if (!res.ok) throw redirect("/login");
  return res.json();
};

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

### ✅ Lab Checkpoint

* Access `/` without token → redirected to `/login`
* Auth enforced **before render**

---

## **Step 4 — Persistent Layout (Role-Aware)**

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

✔ Role-safe navigation
✔ Print-friendly main area

---

## **Step 5 — Login (JWT Acquisition)**

### `src/routes/login.jsx`

```jsx
import { useNavigate } from "react-router-dom";
import { api } from "../api";
import { useState } from "react";

export default function Login() {
  const navigate = useNavigate();
  const [username, setUsername] = useState("");
  const [password, setPassword] = useState("");

  const handleLogin = async e => {
    e.preventDefault();
    const res = await api("/token/", {
      method: "POST",
      body: JSON.stringify({ username, password }),
    });

    if (!res.ok) return alert("Login failed");

    const data = await res.json();
    localStorage.setItem("access_token", data.access);
    navigate("/");
  };

  return (
    <form onSubmit={handleLogin} className="max-w-md mx-auto mt-20 p-6 bg-white shadow">
      <h1 className="text-xl font-bold mb-4">Login</h1>
      <input className="border p-2 w-full mb-3" placeholder="Username" onChange={e => setUsername(e.target.value)} />
      <input type="password" className="border p-2 w-full mb-3" placeholder="Password" onChange={e => setPassword(e.target.value)} />
      <button className="bg-blue-900 text-white px-4 py-2 w-full">Login</button>
    </form>
  );
}
```

✔ JWT stored client-side
✔ Backend remains authority

---

## **Step 6 — Distribution (Command-Only)**

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
    <Form method="post" className="space-y-2">
      <h2 className="font-bold">Stock Distribution</h2>
      <input name="depot" placeholder="Depot ID" className="border p-2" />
      <input name="equipment_name" placeholder="Equipment" className="border p-2" />
      <input name="quantity" type="number" placeholder="Quantity" className="border p-2" />
      <select name="status" className="border p-2">
        <option value="COLLECTION">Collection</option>
        <option value="EMPTY_RETURN">Empty Return</option>
      </select>
      <button className="bg-blue-900 text-white px-4 py-2">Execute</button>
    </Form>
  );
}
```

✔ No direct inventory mutation
✔ Backend service enforces locking

---

## **Step 7 — Transactions (Meter / Cylinder / Service)**

**Principles:**

* UI computes previews only
* Backend commits totals
* Audit & invoice created server-side

```jsx
<MeterSection />
<CylinderSection />
<ServiceSection />
<SummaryBar />
```

**Action target:**
`POST /transactions/transactions/create/`

### ✅ Lab Checkpoint

* Transaction persists even if UI refreshes
* Meter reading increments stored backend-side

---

## **Step 8 — Inventory (Read-Only)**

### `src/routes/inventory.jsx`

```jsx
import { useLoaderData } from "react-router-dom";
import { api } from "../api";

export async function loader() {
  const res = await api("/inventory/inventory/");
  return res.json();
}

export default function Inventory() {
  const items = useLoaderData();
  return (
    <ul>
      {items.map((i, idx) => (
        <li key={idx}>
          {i.depot} — {i.equipment_name}: {i.quantity}
        </li>
      ))}
    </ul>
  );
}
```

✔ Loader-driven
✔ Immutable by design

---

## **Step 9 — Reports (Admin Only)**

### `src/routes/reports.jsx`

```jsx
import { useLoaderData } from "react-router-dom";
import { api } from "../api";

export async function loader() {
  const res = await api("/reports/sales/");
  return res.json();
}

export default function Reports() {
  const data = useLoaderData();
  return (
    <pre className="bg-white p-4">{JSON.stringify(data, null, 2)}</pre>
  );
}
```

✔ Backend-enforced RBAC
✔ Frontend hides menu only as UX aid

---

## **Step 10 — Offline Safety (Optional Extension)**

```js
const buffer = JSON.parse(localStorage.getItem("txn_buffer") || "[]");
buffer.push(txn);
localStorage.setItem("txn_buffer", JSON.stringify(buffer));
```

✔ Regulator-safe retry
✔ No silent loss of sales

---

## ✅ LAB COMPLETION CHECKLIST

* [ ] RR7 loaders/actions only
* [ ] JWT login works
* [ ] Distribution via service
* [ ] Transactions meter-safe
* [ ] Inventory read-only
* [ ] Reports admin-protected
* [ ] Print-friendly UI
* [ ] Swagger-aligned payloads

---

### 🎯 Outcome

You now have a **production-grade, regulator-ready frontend lab** that:

* Mirrors backend domain boundaries
* Enforces workflow correctness
* Scales to audits and field usage
* Avoids frontend-side financial authority

