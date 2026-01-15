# 🧪 HSH SALES SYSTEM — FRONTEND TEST PLAN

**Scope:** React 18 + React Router v7 Data APIs (frontend-only).

**Goal:** Ensure that all UI workflows, component interactions, offline handling, and printing behave correctly, with API requests matching the Swagger contract.

---

## 0️⃣ Test Environment Setup

* **Browser:** Chrome / Edge (latest)
* **Tools:** React Developer Tools, Redux/Context inspector (if applicable), Browser console, LocalStorage inspector
* **Local backend:** Swagger UI `/swagger/` accessible
* **LocalStorage cleared** before tests

**Environment variables (frontend context):**

| Variable             | Purpose                                    |
| -------------------- | ------------------------------------------ |
| `transaction_buffer` | Holds offline transactions                 |
| `user`               | Current logged-in user (from loader)       |
| `customer`           | Current customer in workflow               |
| `invoiceRef`         | Ref to Invoice component for print testing |

---

## 1️⃣ Authentication Flow

**Objective:** Verify `requireAuth` loader works correctly.

1. Load `/` without logging in

   * **Expected:** Redirect to `/login`
2. Fill login form

   * **Check:** Form POST triggers `/users/me/` loader
   * **Expected:** User info available in `DashboardLayout`
3. Confirm `NavLink` visibility

   * Admin: Reports visible
   * Sales: Reports hidden

---

## 2️⃣ Routing & Layout

**Objective:** Confirm RR7 routing and layout shell.

1. Navigate between `/dashboard`, `/distribution`, `/transaction/:id`, `/customers`, `/inventory`, `/reports` (if Admin)

   * **Expected:** URL changes, layout persists, sidebar is constant
2. Loader actions trigger only on route load

   * **Check:** Console logs for `requireAuth`/data fetch
3. Forms trigger **actions** for mutations

   * **Check:** Actions fire, loaders not re-triggered unnecessarily

---

## 3️⃣ Distribution Workflow Test

**Route:** `/distribution`

**Steps:**

1. Fill `vehicle_id`, `cylinder_type`, `quantity`
2. Submit form

   * **Expected:** POST `/distributions/` matches Swagger payload
   * **Expected:** Redirect occurs (`redirect('/')`)
3. Validate UI confirmation
4. Offline test:

   * Disconnect network, submit form
   * **Expected:** Form fails gracefully (optional toast)
   * **Check:** Payload not lost in LocalStorage if queued

---

## 4️⃣ Transaction — Meter, Cylinder, Service

**Route:** `/transaction/:id`

**Objective:** Ensure accurate subtotal calculation, offline buffering, and invoice generation.

1. **Meter Section**

   * Change last/latest readings
   * Confirm `qty = latest - last`
   * Click `Update`
   * **Expected:** `onChange` updates parent state
2. **Cylinder Section**

   * Enter quantities for all cylinder types
   * **Expected:** Values propagate to `SummaryBar`
3. **Service Section**

   * Enter quantities for services (Delivery, Installation, Other)
   * **Expected:** Values reflected in `SummaryBar`
4. **SummaryBar**

   * Confirm subtotal, totals
   * **Expected:** Sum matches meter + cylinders + services
5. **Form Submission**

   * Online: POST `/transactions/` succeeds
   * Offline: Stored in `transaction_buffer` (LocalStorage)
   * **Flush:** `flushBuffer` successfully replays queued transactions

---

## 5️⃣ Inventory View Test

**Route:** `/inventory`

1. Loader fetches `/inventory/` (Swagger contract)
2. Table renders correct item names and quantities
3. Verify read-only behavior
4. Test edge cases: empty inventory, large numbers

---

## 6️⃣ Customers Management

**Route:** `/customers`

1. Load list of customers
2. Confirm table displays all expected fields (`name`, `payment_type`, rates)
3. Test filtering or selection if applicable

---

## 7️⃣ Reports Module (Admin Only)

**Route:** `/reports`

1. Loader uses URL query parameters for filters
2. Table renders dynamically (`Object.values(row)`)
3. Validate multiple report formats (Summary, Aging)
4. Confirm table is print-ready

---

## 8️⃣ Invoice Print Layout

**Component:** `Invoice.jsx`

1. Render `Invoice` with sample transaction
2. Verify fields:

   * Invoice number, date, customer name
   * Meter sale: last/latest readings, qty
   * Cylinder sales: all types
   * Service items
   * Total
3. Test **print media styles**:

   * A4 (browser print)
   * Thermal (ESC/POS CSS approximation)
4. Ref usage: `React.forwardRef` works, captures DOM element

---

## 9️⃣ Offline Transaction Buffer

**Module:** `utils/transactionBuffer.js`

1. Add a transaction via `addToBuffer`

   * Check LocalStorage contains the correct JSON
2. Simulate flush with network:

   * **Successful:** Transaction POSTed, buffer cleared
   * **Failed:** Transaction remains in buffer, error logged
3. Edge cases:

   * Multiple transactions queued
   * Network flapping (connect/disconnect)

---

## 🔟 Edge Case & UX Validation

1. **Invalid inputs**

   * Negative numbers, empty fields
   * Expect client-side validation
2. **Offline-first experience**

   * Submit forms offline, reconnect, verify auto-sync
3. **UI responsiveness**

   * Tailwind CSS layout on desktop/mobile
   * Print-friendly styles
4. **Error handling**

   * API 4xx/5xx shows user-friendly messages

---

## 1️⃣1️⃣ Test Execution & Verification

* Use browser console and network tab to confirm **fetch payloads match Swagger definitions**
* Inspect LocalStorage `transaction_buffer`
* Use developer tools to simulate offline mode
* Verify **print preview** for invoice layout
* Sequential testing ensures **workflow correctness**:

  * Login → Dashboard → Distribution → Transaction → Invoice → Inventory → Reports

---

### ✅ Key Features of Frontend Test Plan

* Stepwise verification per route and workflow
* RR7-compliant loader/action checks
* Offline-tolerant transaction buffering
* Print-ready invoice layout tests
* Workflow and UX integrity
