# 🧪 HSH SALES SYSTEM — FRONTEND TEST PLAN (v2.0)

**Scope:** React 18 + React Router v7 Data APIs (frontend-only)
**Goal:** Ensure all UI workflows, component interactions, offline handling, and printing behave correctly. Verify API requests match Swagger contracts and data integrity is maintained.

---

## 0️⃣ Test Environment Setup

**Browser & Tools:**

* Chrome / Edge (latest)
* React Developer Tools
* Redux / Context inspector (if applicable)
* Browser console / network inspector
* LocalStorage inspector

**Preconditions:**

* Swagger UI `/swagger/` accessible
* LocalStorage cleared
* Network offline/online simulation ready

**Frontend Environment Variables:**

| Variable             | Purpose                                    |
| -------------------- | ------------------------------------------ |
| `transaction_buffer` | Holds offline transactions                 |
| `user`               | Current logged-in user (from loader)       |
| `customer`           | Current customer in workflow               |
| `invoiceRef`         | Ref to Invoice component for print testing |

---

## 1️⃣ Authentication Flow

**Objective:** Verify `requireAuth` loader works and RBAC rules are enforced.

**Steps:**

1. Load `/` without logging in → **Redirect to `/login`**
2. Fill login form & submit → Loader triggers `/users/me/` → **User info populated in `DashboardLayout`**
3. Verify `NavLink` visibility:

   * Admin → Reports visible
   * Sales → Reports hidden
4. Test token persistence in LocalStorage/sessionStorage

---

## 2️⃣ Routing & Layout

**Objective:** Verify RR7 routing and layout shell integrity.

**Steps:**

1. Navigate between `/dashboard`, `/distribution`, `/transaction/:id`, `/customers`, `/inventory`, `/reports` (Admin only) → **URL updates, layout persists, sidebar constant**
2. Loaders trigger only on route change → Check console logs for `requireAuth`/data fetch
3. Actions fire correctly without unnecessary re-fetches → Confirm mutations vs loader triggers

---

## 3️⃣ Distribution Workflow

**Route:** `/distribution`

**Steps:**

1. Fill vehicle ID, cylinder type, and quantity → Submit form

   * **Expected:** POST `/distributions/` matches Swagger payload
   * Redirect occurs (`redirect('/')`)
2. Validate UI confirmation message
3. Offline simulation: disconnect network → submit form

   * Form queues transaction in `transaction_buffer`
   * Payload stored for later flush

---

## 4️⃣ Transaction Module — Meter, Cylinder, Service

**Route:** `/transaction/:id`
**Objective:** Ensure correct subtotal calculation, offline buffering, and invoice generation.

**Steps:**

1. **Meter Section:** Input last/latest readings → confirm `qty = latest - last`; click **Update** → parent state updated
2. **Cylinder Section:** Input quantities for all cylinder types → propagate to **SummaryBar**
3. **Service Section:** Input quantities for Delivery, Installation, Other → reflected in **SummaryBar**
4. **SummaryBar Validation:** Subtotal = meter + cylinders + services → totals dynamically updated
5. **Form Submission:**

   * Online → POST `/transactions/` succeeds
   * Offline → stored in `transaction_buffer`
   * Flush → queued transactions replay correctly

---

## 5️⃣ Inventory View

**Route:** `/inventory`

* Loader fetches `/inventory/` → Swagger-compliant
* Table renders item names and quantities (`full_qty`, `empty_qty`)
* Read-only validation
* Edge cases: empty inventory, very large numbers

---

## 6️⃣ Customers Management

**Route:** `/customers`

* Load list → verify loader
* Table displays all fields (`name`, `payment_type`, rates)
* Test filtering and selection (if applicable)
* Validate edit forms against Swagger schema

---

## 7️⃣ Reports Module (Admin Only)

**Route:** `/reports`

* Loader respects query params (filters: date, customer, type)
* Table renders dynamically (`Object.values(row)`)
* Validate multiple formats: Summary, Aging
* Confirm table is print-ready (A4/thermal CSS)
* Edge case: empty reports, very large datasets

---

## 8️⃣ Invoice Print Layout

**Component:** `Invoice.jsx`

* Render invoice with sample transaction
* Verify all fields: invoice number, date, customer, meter sale, cylinder sales, service items, total
* Print media styles: A4 (browser), Thermal (ESC/POS CSS)
* Confirm `React.forwardRef` captures DOM element for printing

---

## 9️⃣ Offline Transaction Buffer

**Module:** `utils/transactionBuffer.js`

* Add transaction via `addToBuffer` → verify LocalStorage
* Simulate flush: success → posted & buffer cleared, failure → remains, error logged
* Edge cases: multiple transactions queued, network flapping
* UI reflects buffer state

---

## 🔟 Edge Case & UX Validation

* Invalid inputs → negative numbers, empty fields → client-side validation
* Offline-first experience → forms queued, auto-sync on reconnect
* UI responsiveness → Tailwind layout on desktop/mobile
* Error handling → API 4xx/5xx → user-friendly messages
* Print preview → dynamic fields present and correct

---

## 1️⃣1️⃣ Test Execution & Verification

* Verify fetch payloads match Swagger contract (browser console / network tab)
* Inspect LocalStorage `transaction_buffer`
* Simulate offline mode via dev tools
* Verify print preview for invoice layout
* Sequential workflow testing:

`Login → Dashboard → Distribution → Transaction → Invoice → Inventory → Reports`

---

## ✅ Key Features

* Stepwise verification per route/workflow
* Loader/action checks per RR7
* Offline-tolerant transaction buffering
* Print-ready invoice tests
* Workflow & UX integrity
* Edge-case & error handling coverage

---

# 🧪 HSH SALES SYSTEM — FRONTEND TEST CHECKLIST (v2.0)

| #  | Route / Component               | UI Elements / Fields                                              | Action                        | Expected Behavior                                        | Payload / API Check                                          | Offline Handling                                     | Notes                                            |
| -- | ------------------------------- | ----------------------------------------------------------------- | ----------------------------- | -------------------------------------------------------- | ------------------------------------------------------------ | ---------------------------------------------------- | ------------------------------------------------ |
| 1  | `/login`                        | Username, Password, Login button                                  | Fill form & submit            | Redirect to `/dashboard`                                 | POST `/token/` returns `access` & `refresh`                  | N/A                                                  | Token stored in env/localStorage                 |
| 2  | DashboardLayout                 | Sidebar, NavLinks                                                 | Load page                     | Sidebar visible, correct links per role                  | GET `/users/me/`                                             | N/A                                                  | Reports hidden for Sales                         |
| 3  | `/dashboard`                    | Overview cards (Sales, Inventory)                                 | View page                     | Correct totals rendered                                  | GET `/reports/summary/`                                      | N/A                                                  | Ensure numbers match backend                     |
| 4  | `/distribution`                 | Vehicle ID, Cylinder Type, Quantity, Submit                       | Fill form & submit            | Form clears, confirmation message                        | POST `/distributions/` with user_id, depot_id, qty, movement | Form queued in `transaction_buffer` if offline       | Confirm audit log entry exists                   |
| 5  | `/transaction/:id` - Meter      | Last Reading, Latest Reading, Update button                       | Input readings & update       | Qty calculated dynamically                               | PATCH `/transactions/meter_sale/`                            | Queue update in buffer if offline                    | Check subtotal updates                           |
| 6  | `/transaction/:id` - Cylinder   | Cylinder types & quantity inputs                                  | Input quantities              | SummaryBar reflects totals                               | PATCH `/transactions/items/`                                 | Queue update if offline                              | Values propagated to parent state                |
| 7  | `/transaction/:id` - Service    | Service types & quantity inputs                                   | Input quantities              | SummaryBar reflects totals                               | PATCH `/transactions/items/`                                 | Queue update if offline                              | Check subtotal updates                           |
| 8  | `/transaction/:id` - SummaryBar | Total, Subtotal, Submit button                                    | Submit transaction            | POST `/transactions/` success → redirect                 | POST payload matches Swagger                                 | Buffer if offline, flush on reconnect                | Confirm audit log                                |
| 9  | `/inventory`                    | Table: Equipment, Depot, Full/Empty Qty                           | Load page                     | Correct quantities displayed, read-only                  | GET `/inventory/`                                            | N/A                                                  | Edge cases: empty inventory, large numbers       |
| 10 | `/customers`                    | Table: Name, Payment Type, Rates                                  | Load page                     | Correct data displayed                                   | GET `/customers/`                                            | N/A                                                  | Filter/search works                              |
| 11 | `/reports` (Admin)              | Filters, Table rows                                               | Apply filter                  | Table updates dynamically                                | GET `/reports/summary/` or `/reports/aging/`                 | N/A                                                  | Table print-ready, empty datasets handled        |
| 12 | Invoice.jsx                     | Invoice number, Date, Customer, Meter, Cylinders, Services, Total | Render Invoice                | All data displayed correctly                             | N/A (local rendering from transaction)                       | N/A                                                  | Print styles correct (A4/thermal)                |
| 13 | Offline Transaction Buffer      | LocalStorage: `transaction_buffer`                                | Add transaction               | JSON stored correctly                                    | Payload matches POST format                                  | Queued transactions replay successfully on reconnect | Multiple transactions handled sequentially       |
| 14 | Edge Cases                      | Invalid inputs, negative numbers                                  | Input invalid values          | Client-side validation prevents submit                   | N/A                                                          | Queued only valid transactions                       | Error messages user-friendly                     |
| 15 | Network Flapping                | Any page                                                          | Simulate connect/disconnect   | Offline forms queued, auto-sync on reconnect             | Validate replayed requests                                   | Check buffer consistency                             | Confirm UI reflects sync status                  |
| 16 | Print Layout Test               | Invoice / Report print                                            | Trigger print preview         | Layout correct for media query (A4/thermal)              | N/A                                                          | N/A                                                  | Confirm totals, customer details, line items     |
| 17 | RBAC Enforcement                | NavLinks, Forms                                                   | Login as Sales                | Reports hidden, restricted endpoints 403                 | GET `/accounts/users/` → 403                                 | N/A                                                  | Verify frontend hides links & API rejects access |
| 18 | Loader / Action Verification    | Data fetching & mutations                                         | Navigate routes, submit forms | Loader triggers per route only, actions fire on mutation | All API payloads match Swagger                               | Offline queue tested                                 | Prevent unnecessary re-fetches                   |

---

### ✅ Usage Notes

1. Mark **Pass/Fail** per step during manual testing
2. For automation, map **UI fields → selectors → expected API payload**
3. Offline-first tests: simulate network loss in browser dev tools
4. Print tests: use print preview or headless browser to validate CSS
5. Audit compliance: verify every mutation triggers a corresponding audit entry

