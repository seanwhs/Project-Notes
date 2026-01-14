# 🏗️ HSH SALES SYSTEM — TECHNICAL PROPOSAL & ARCHITECTURE (Enhanced)

---

## 1️⃣ Overview

The **HSH Sales System** is a **full-stack enterprise LPG logistics and sales platform** designed to optimize:

* **Field operations** — Online/offline order capture, cylinder distribution, and returns
* **Inventory management** — Track full/empty cylinders depot-wise
* **Transaction processing** — Sales, meter readings, and billing
* **Financial documents** — PDF invoices, receipts, and email dispatch
* **Audit & compliance** — Logs all user actions for traceability
* **Reporting & analytics** — Filterable transaction history with exportable reports

**Key Objectives:**

1. Decoupled, scalable architecture (React SPA + Django REST Framework backend)
2. Offline-first operation for field personnel
3. Audit logging & role-based access control (Admin/Sales)
4. PDF invoices & email automation for consistency
5. Cloud-ready deployment with Docker

---

## 2️⃣ Technology Stack

| Layer            | Technology / Tool                        | Purpose / Benefit                        |
| ---------------- | ---------------------------------------- | ---------------------------------------- |
| Frontend         | React SPA + React Router v7, TailwindCSS | Mobile-first, offline-capable UI         |
| Backend          | Django REST Framework, Python 3.11       | REST API, business logic, validations    |
| Database         | MySQL 8.0                                | ACID-compliant transactions, reliability |
| Authentication   | JWT (SimpleJWT)                          | Token-based RBAC, secure session         |
| PDF Generation   | WeasyPrint                               | Server-side PDF rendering from HTML/CSS  |
| Emailing         | Django SMTP / EmailMessage               | Automated PDF invoice dispatch           |
| Offline Support  | LocalStorage queue, auto-sync            | Field resilience for offline operations  |
| Printing         | ESC/POS via Web Bluetooth                | Thermal receipt printing                 |
| Containerization | Docker + Docker Compose                  | Cloud-ready, portable deployment         |
| Reporting        | DRF + WeasyPrint PDF                     | Filtered, exportable transaction reports |

---

## 3️⃣ System Domains (Bounded Contexts)

### 3.1 Authentication & Authorization

* Login / logout
* Role-based access (Admin vs Sales)
* JWT tokens with DRF SimpleJWT
* Audit logging for sensitive actions

### 3.2 Master Data

* Users
* Customers & pricing
* Equipment / cylinders
* Depot mapping

### 3.3 Operational Flows

* Cylinder Distribution / Returns
* Transaction capture with real-time inventory deduction
* Meter readings and usage calculations
* Atomic inventory updates to prevent inconsistencies

### 3.4 Financial Documents

* PDF invoice generation
* Email dispatch to customers
* Paid/unpaid status tracking

### 3.5 Reporting

* Customer-specific sales reports
* Transaction history with filters
* Export to PDF/Excel

### 3.6 Audit & Logging

* User actions (create, update, delete)
* Rate or pricing updates
* Transaction and billing activities

---

## 4️⃣ Backend Architecture (Django REST Framework)

```
backend/
├── config/
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
│
├── apps/
│   ├── accounts/        # Users, roles, JWT
│   ├── customers/       # Customer master & rates
│   ├── inventory/       # Cylinders, depots, stock
│   ├── distribution/    # Collection / return workflow
│   ├── transactions/    # Sales, meter readings
│   ├── billing/         # PDF invoice, email service
│   ├── reports/         # Filtered transaction exports
│   └── audit/           # Action logging
│
├── shared/
│   ├── permissions/     # Role-based access
│   ├── serializers/     # DRF serializers
│   └── utils/           # Common helpers
│
└── manage.py
```

### 4.1 Backend Principles

* **Service Layer**: Encapsulates business logic separate from views
* **Serializer Validation**: Ensures input correctness
* **Atomic Transactions**: Inventory + transaction consistency
* **PDF Generation**: Using WeasyPrint from HTML/CSS templates
* **Email Automation**: Sends invoices/receipts to customers
* **Soft Deletes + Audit Logging**: Track critical changes without losing data

---

## 5️⃣ Frontend Architecture (React + RR7)

```
frontend/
├── src/
│   ├── api/             # Axios / fetch services
│   ├── auth/            # Login, route guards
│   ├── components/      # Reusable UI elements
│   ├── pages/
│   │   ├── Distribution/
│   │   ├── Transactions/
│   │   ├── Users/
│   │   ├── Customers/
│   │   ├── Reports/
│   │   └── Login/
│   ├── hooks/           # Printer, offline sync
│   ├── utils/           # Helpers, validators
│   └── App.jsx
```

### 5.1 Frontend Principles

* Controlled forms with validation
* Dynamic multi-line transaction items
* Real-time calculations (latest meter reading - previous reading)
* PDF preview, print, and download (80mm thermal width)
* Role-based route guarding
* Search + autocomplete for customer selection

---

## 6️⃣ Database Schema & Logic

### 6.1 Core Models

```python
class Customer(models.Model):
    name = models.CharField(max_length=255)
    customer_id = models.CharField(max_length=50, unique=True)
    payment_type = models.CharField(max_length=20, choices=[('Monthly','Monthly'),('Cash','Cash')])
    meter_rate = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    cyl_9_rate = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    # Add rates for 12.7, 14kg, 50 POL, 50 L

class Transaction(models.Model):
    transaction_id = models.CharField(max_length=20, unique=True)
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    customer = models.ForeignKey(Customer, on_delete=models.CASCADE)
    total_amount = models.DecimalField(max_digits=12, decimal_places=2)
    payment_status = models.BooleanField(default=False)
    timestamp = models.DateTimeField(auto_now_add=True)

class MeterReading(models.Model):
    transaction = models.OneToOneField(Transaction, on_delete=models.CASCADE)
    last_reading = models.DecimalField(max_digits=12, decimal_places=2)
    latest_reading = models.DecimalField(max_digits=12, decimal_places=2)
    quantity = models.DecimalField(max_digits=10, decimal_places=2)
```

### 6.2 Database ERD (ASCII)

```
+---------+        +-----------------+        +-------------+
|  User   |1------*|  Transaction    |*------1|  Customer   |
+---------+        +-----------------+        +-------------+
      |
      |1
      *
+-----------------+
| MeterReading    |
+-----------------+

+-----------------+
| AuditLog        |
+-----------------+
Logs all user actions (FK: user_id)
```

---

## 7️⃣ Key Feature Implementation

### 7.1 PDF & Email Workflow

```python
from django.core.mail import EmailMessage
from weasyprint import HTML
from django.template.loader import render_to_string
from rest_framework.decorators import api_view
from rest_framework.response import Response

def generate_pdf(transaction):
    html_content = render_to_string('invoice_template.html', {'data': transaction})
    return HTML(string=html_content).write_pdf()

@api_view(['POST'])
def email_invoice(request, transaction_id):
    transaction = Transaction.objects.get(id=transaction_id)
    pdf_content = generate_pdf(transaction)
    
    email = EmailMessage(
        subject=f'HSH Sales Receipt: {transaction.transaction_id}',
        body='Please find your invoice attached.',
        from_email='billing@hshlpg.com',
        to=[request.data['email']]
    )
    email.attach(f'HSH_{transaction_id}.pdf', pdf_content, 'application/pdf')
    email.send()
    return Response({"status": "Success"})
```

### 7.2 QuickBooks Integration (Optional)

* Executes as **background Celery task**
* Pushes transactions to QuickBooks Online via API
* Updates local `qb_synced` flag on success

```python
def sync_to_qb(transaction_id):
    # Map MySQL transaction to QB invoice
    # Push via OAuth2 API
    # Mark local transaction as synced
    pass
```

---

## 8️⃣ Offline & Printing Strategy

* **Offline Queue**: Unsynced transactions saved in LocalStorage
* **Auto-sync**: Automatically syncs when network restored
* **Thermal Printing**: `usePrinter` hook using ESC/POS via Web Bluetooth
* **80mm Thermal Paper**: CSS print media queries for width & margins
* **Preview & Download**: FileSaver.js for browser downloads

---

## 9️⃣ Deployment Architecture

```
┌─────────┐   ┌─────────┐   ┌─────────────┐
│Frontend │   │Backend  │   │ DB (MySQL)  │
│5173     │   │8000     │   │3306         │
│Vol:/app │   │Vol:/app │   │Vol:mysql_data
└─────────┘   └─────────┘   └─────────────┘
```

* Docker Compose orchestrates frontend, backend, and database
* `.env` manages secrets & DB credentials
* Cloud-ready deployment on AWS, GCP, or Azure
* Persistent MySQL storage via Docker volumes

**Quick Start:**

```bash
docker-compose up --build
docker exec -it hsh-sales-system_backend_1 bash
python manage.py createsuperuser
```

---

## 🔟 Proposal Summary

| Package                | Scope                                    | Cost (S$) |
| ---------------------- | ---------------------------------------- | --------- |
| Stand-alone System     | React + DRF + MySQL + PDF + Email + RBAC | 16,000    |
| QuickBooks Integration | Inventory + Invoice Sync                 | 13,800    |
| Cloud Hosting          | Docker-based, AWS/GCP/Azure              | TBD       |

---

✅ **This proposal includes:**

* Full-stack architecture: DRF backend + React SPA frontend
* Role-based authentication (Admin/Sales) & JWT security
* PDF invoice generation & email automation
* Offline-first support & thermal printing
* Database schema & ERDs
* QuickBooks background sync design
* Dockerized, cloud-ready deployment

