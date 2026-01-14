# 🎨 HSH LPG — Frontend CSS & Tailwind Bundle

This is a **complete, ready-to-drop `src/index.css`** for your React frontend. Covers **all components**: Meter, Cylinder, Service, SummaryBar, Forms, Tables, Buttons, and Reports — fully **print-ready**.

---

## `src/index.css` — FULL BUNDLE

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* --------------------------------------
   GLOBAL RESET & BASE STYLING
-------------------------------------- */
html, body {
  font-family: 'Inter', sans-serif;
  margin: 0;
  padding: 0;
  background-color: #f3f4f6; /* gray-100 */
  color: #111827; /* gray-900 */
}

a {
  text-decoration: none;
  color: inherit;
}

button {
  cursor: pointer;
  font-weight: 600;
  border-radius: 0.375rem;
  padding: 0.5rem 1rem;
  transition: background-color 0.2s;
}

input, select {
  border: 1px solid #d1d5db;
  border-radius: 0.375rem;
  padding: 0.5rem 0.75rem;
  font-size: 0.875rem;
  background-color: #fff;
}

input:focus, select:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 2px rgba(59,130,246,0.3);
}

/* --------------------------------------
   LAYOUT: SIDEBAR & DASHBOARD
-------------------------------------- */
aside {
  width: 16rem; /* 64 */
  background-color: #1e3a8a; /* blue-900 */
  color: white;
  padding: 1rem;
}

aside h1 {
  font-size: 1.25rem;
  font-weight: 700;
  margin-bottom: 1.5rem;
}

aside nav a {
  display: block;
  padding: 0.5rem 0;
  border-radius: 0.375rem;
  transition: background-color 0.2s;
}

aside nav a.active, aside nav a:hover {
  background-color: #2563eb; /* blue-600 */
}

main {
  flex: 1;
  padding: 1.5rem;
  background-color: #f3f4f6; /* gray-100 */
  min-height: 100vh;
}

/* --------------------------------------
   TABLES
-------------------------------------- */
table {
  border-collapse: collapse;
  width: 100%;
}

th, td {
  padding: 0.5rem 1rem;
  border-bottom: 1px solid #e5e7eb; /* gray-200 */
  text-align: left;
}

th {
  background-color: #f9fafb; /* gray-50 */
  font-weight: 600;
}

tr:nth-child(even) {
  background-color: #f3f4f6; /* gray-100 */
}

/* --------------------------------------
   BUTTONS
-------------------------------------- */
.bg-blue-700 { background-color: #1d4ed8; color: white; }
.bg-blue-700:hover { background-color: #1e40af; }
.bg-green-500 { background-color: #22c55e; color: white; }
.bg-green-500:hover { background-color: #16a34a; }
.bg-gray-700 { background-color: #374151; color: white; }
.bg-gray-700:hover { background-color: #1f2937; }

/* --------------------------------------
   ALERTS / STATUS
-------------------------------------- */
.text-green-600 { color: #16a34a; }
.text-red-600 { color: #dc2626; }

/* --------------------------------------
   COMPONENT-SPECIFIC STYLING
-------------------------------------- */

/* Meter Section */
.meter-section input {
  width: 100px;
  text-align: right;
}

.meter-subtotal {
  font-weight: 600;
  color: #1e40af; /* blue-800 */
}

/* Cylinder Section */
.cylinder-row {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

/* Service Section */
.service-row {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

/* Summary Bar */
.summary-bar {
  display: flex;
  justify-content: space-between;
  background-color: #1e3a8a; /* blue-900 */
  color: white;
  padding: 0.75rem 1rem;
  border-radius: 0.5rem;
  font-weight: 700;
}

.summary-bar button {
  background-color: #22c55e;
  color: white;
  padding: 0.5rem 1rem;
  border-radius: 0.375rem;
}

/* Forms */
form.grid {
  display: grid;
  gap: 1rem;
}

/* --------------------------------------
   PRINT STYLES
-------------------------------------- */
@media print {
  body {
    background-color: #fff;
    color: #000;
  }

  aside, nav, button, input, select {
    display: none !important;
  }

  main {
    width: 100%;
    padding: 0;
  }

  table {
    font-size: 12pt;
    border: 1px solid #000;
  }

  th, td {
    border: 1px solid #000;
    padding: 0.3rem 0.5rem;
  }

  .bg-white, .bg-blue-900, .bg-gray-100, .bg-gray-700 {
    background-color: #fff !important;
    color: #000 !important;
  }

  .text-green-600, .text-red-600 {
    color: #000 !important;
  }
}

/* --------------------------------------
   MISC / UTILITY
-------------------------------------- */
.text-sm { font-size: 0.875rem; }
.text-xl { font-size: 1.25rem; }
.font-bold { font-weight: 700; }
.rounded { border-radius: 0.375rem; }
.shadow { box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
.p-4 { padding: 1rem; }
.p-6 { padding: 1.5rem; }
.bg-white { background-color: #fff; }
```

---

# 🎨 Component Class Mapping Cheat Sheet

| Component / Element                    | Tailwind / CSS Classes                                                                                                     | Notes / Purpose                                                     |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| **Sidebar (`aside`)**                  | `w-64 bg-blue-900 text-white p-4`                                                                                          | Full-height, fixed-width, dark blue                                 |
| Sidebar heading (`h1`)                 | `font-bold mb-6 text-xl`                                                                                                   | Brand/logo                                                          |
| Sidebar nav links                      | `block py-2 px-2 rounded hover:bg-blue-600 active:bg-blue-600`                                                             | Highlight active page                                               |
| **Main content (`main`)**              | `flex-1 p-6 bg-gray-100 min-h-screen`                                                                                      | Flexible, padded, light background                                  |
| **Forms (`form`)**                     | `grid gap-4 grid-cols-4`                                                                                                   | Use `grid-cols-*` depending on layout                               |
| **Input / Select**                     | `border border-gray-300 rounded px-3 py-2 text-sm focus:outline-none focus:border-blue-500 focus:ring focus:ring-blue-300` | Standardized form inputs                                            |
| **Buttons**                            | `bg-blue-700 text-white py-2 px-4 rounded hover:bg-blue-600`                                                               | Action buttons; can swap bg color for `bg-green-500`, `bg-gray-700` |
| Success alert text                     | `text-green-600`                                                                                                           | Show after save/submit                                              |
| Error / unpaid text                    | `text-red-600`                                                                                                             | For unpaid invoice / invalid                                        |
| **Table**                              | `w-full border-collapse text-sm`                                                                                           | Full-width, readable                                                |
| Table header (`th`)                    | `bg-gray-100 font-semibold px-4 py-2`                                                                                      | Light header with bold text                                         |
| Table cell (`td`)                      | `border-b px-4 py-2`                                                                                                       | Standard cell                                                       |
| Table striped                          | `tr:nth-child(even) bg-gray-100`                                                                                           | Alternating row color                                               |
| **Meter Section (`.meter-section`)**   | `flex gap-2 mb-2`                                                                                                          | Aligns meter inputs horizontally                                    |
| Meter input                            | `w-24 text-right border border-gray-300 rounded px-2 py-1`                                                                 | Input for latest reading                                            |
| Meter subtotal                         | `font-semibold text-blue-800`                                                                                              | Shows calculated subtotal                                           |
| **Cylinder Section (`.cylinder-row`)** | `flex gap-2 mb-2`                                                                                                          | Each cylinder line item row                                         |
| **Service Section (`.service-row`)**   | `flex gap-2 mb-2`                                                                                                          | Each service line item row                                          |
| **Summary Bar (`.summary-bar`)**       | `flex justify-between bg-blue-900 text-white p-4 rounded font-bold`                                                        | Total / Save button container                                       |
| Summary button                         | `bg-green-500 px-6 py-2 rounded hover:bg-green-600`                                                                        | Action: Save & Print                                                |
| **Report filters (Form)**              | `grid grid-cols-6 gap-4 text-sm mb-4`                                                                                      | Six-column layout for filters                                       |
| Report table                           | `w-full border text-sm`                                                                                                    | Display filtered transactions                                       |
| Report table header (`thead`)          | `bg-gray-100 font-semibold`                                                                                                | Light header row                                                    |
| Report row (`tr`)                      | `border-t`                                                                                                                 | Separates rows                                                      |
| Paid status                            | `text-green-600`                                                                                                           | Paid transaction label                                              |
| Unpaid status                          | `text-red-600`                                                                                                             | Unpaid transaction label                                            |
| **Print mode adjustments**             | `@media print { aside, nav, button, input, select { display:none !important; } main { width:100%; padding:0; } }`          | Hides sidebar, forms, buttons; converts table to print-ready        |

---

### 💡 Tips

1. Wrap inputs/buttons in `<div className="mb-2">` for spacing.
2. Use `grid-cols-*` to adjust form width: 4 for main forms, 6 for report filters.
3. Alerts: `.text-green-600` for success, `.text-red-600` for errors/unpaid.
4. Print-ready: All buttons, forms, sidebar are automatically hidden.
5. Use `h2 / h3` with `.font-bold .text-xl/.text-lg` for clear section headers.

---

This bundle **directly supports your full frontend** (Transaction, Distribution, Inventory, Reports) **with print-ready, workflow-aligned styling**.


