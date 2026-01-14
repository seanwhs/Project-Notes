# 🏗️ HSH SALES SYSTEM — TECHNICAL PROPOSAL & ARCHITECTURE

## 1️⃣ Overview

The **HSH Sales System** is a **full-stack enterprise LPG logistics and sales platform** designed for:

* **Field operations** — support online/offline order capture, cylinder distribution, and returns
* **Inventory management** — track full and empty cylinders, depot-wise
* **Transaction processing** — sales, meter readings, and billing
* **Financial documents** — PDF invoices, receipts, and emails
* **Audit & compliance** — logs user actions for traceability
* **Reporting** — filtered transaction history and exportable reports

**Key Objectives**:

1. Decoupled, scalable architecture (React SPA + DRF backend)
2. Offline-first operation for field sales personnel
3. Audit logging and role-based access control (Admin/Sales)
4. PDF invoices and email automation for consistency
5. Cloud-ready deployment using Docker

---

## 2️⃣ Technology Stack

| Layer            | Technology / Tool                        | Purpose                                 |
| ---------------- | ---------------------------------------- | --------------------------------------- |
| Frontend         | React SPA + React Router v7, TailwindCSS | Mobile-first, offline-capable UI        |
| Backend          | Django REST Framework, Python 3.11       | REST API, service layer, validation     |
| Database         | MySQL 8.0                                | ACID-compliant transactions             |
| Authentication   | JWT (SimpleJWT)                          | Token-based RBAC                        |
| PDF Generation   | WeasyPrint                               | Server-side PDF rendering with HTML/CSS |
| Emailing         | Django SMTP / EmailMessage               | PDF invoice dispatch                    |
| Offline Support  | LocalStorage queue, auto-sync            | Resilient field operations              |
| Printing         | ESC/POS via Web Bluetooth                | Thermal receipt printing                |
| Containerization | Docker + Docker Compose                  | Cloud-ready deployment                  |
| Reporting        | DRF services + WeasyPrint PDF            | Transaction history, filtered reports   |

---

## 3️⃣ System Domains (Bounded Contexts)

### 3.1 Authentication & Authorization

* Login / logout
* Role-based access (Admin vs Sales)
* JWT tokens with DRF SimpleJWT
* Audit logging for sensitive actions

### 3.2 Master Data

* Users
* Customers & rates
* Equipment / cylinders
* Depot mapping

### 3.3 Operational Flows

* Cylinder Distribution / Returns
* Transaction capture
* Meter readings
* Inventory updates (atomic)

### 3.4 Financial Documents

* Invoice generation (PDF)
* Email dispatch
* Paid/unpaid status tracking

### 3.5 Reporting

* Customer sales reports
* Transaction history
* Filter and export capabilities

### 3.6 Audit & Logging

* User actions (create, update, delete)
* Rate updates
* Transaction / billing activities

---

## 4️⃣ Backend Architecture (DRF)

```
backend/
├── config/
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
│
├── apps/
│   ├── accounts/        # Users, roles, JWT
│   ├── customers/       # Customer master + rates
│   ├── inventory/       # Cylinders, depots, stock
│   ├── distribution/    # Collection / return workflow
│   ├── transactions/    # Sales, meter readings
│   ├── billing/         # PDF invoice, email service
│   ├── reports/         # Query + export filtered data
│   └── audit/           # Action logging
│
├── shared/
│   ├── permissions/     # Role-based access
│   ├── serializers/     # DRF serializers
│   └── utils/           # Common helper functions
│
└── manage.py
```

### 4.1 Backend Concepts

* Service layer separates **business logic** from views
* Serializer validation for input; domain validation in services
* Atomic transactions to ensure inventory + transaction consistency
* PDF generation using **WeasyPrint**
* Email with PDF attachments
* Soft deletes + audit logging

---

## 5️⃣ Frontend Architecture (React + RR7)

```
frontend/
├── src/
│   ├── api/             # Axios / fetch services
│   ├── auth/            # Login, route guards
│   ├── components/      # Reusable UI components
│   ├── pages/
│   │   ├── Distribution/
│   │   ├── Transaction/
│   │   ├── Users/
│   │   ├── Customers/
│   │   ├── Reports/
│   │   └── Login/
│   ├── hooks/           # Printer, offline sync
│   ├── utils/           # Helpers, validation
│   └── App.jsx
```

### 5.1 Frontend Concepts

* Controlled forms with validation
* Dynamic multi-line items in transaction forms
* Real-time calculations (latest meter reading - last reading)
* PDF preview, print, and download (80mm thermal width)
* Role-based routing
* Search + dropdown with auto-complete for customers

---

## 6️⃣ Database Schema & Logic

### Core Models

```python
class Customer(models.Model):
    name = models.CharField(max_length=255)
    customer_id = models.CharField(max_length=50, unique=True)
    payment_type = models.CharField(max_length=20, choices=[('Monthly','Monthly'),('Cash','Cash')])
    meter_rate = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    cyl_9_rate = models.DecimalField(max_digits=10, decimal_places=2, default=0)
    # additional cylinder rates (12.7, 14, 50 POL, 50 L)

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

---

### 6.1 Database ERD (ASCII)

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
Logs actions of User (FK: user_id)
```

---

## 7️⃣ Key Feature Implementation

### 7.1 PDF & Email Workflow

```python
from django.core.mail import EmailMessage
from weasyprint import HTML

def generate_pdf(transaction):
    html_content = render_to_string('invoice_template.html', {'data': transaction})
    return HTML(string=html_content).write_pdf()

@api_view(['POST'])
def email_report(request, transaction_id):
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

### 7.2 QuickBooks Integration Logic

* Runs as **background Celery task**
* Pushes transaction data to QuickBooks Online via API
* Updates local `qb_synced` flag on success

```python
def sync_to_qb(transaction_id):
    # Map MySQL transaction to QB Invoice
    # Push via OAuth2 API
    # Mark local transaction as synced
    pass
```

---

## 8️⃣ Offline & Printing Strategy

* **Offline Queue**: Transactions stored locally in `LocalStorage`
* **Auto-sync**: Syncs automatically when network restored
* **Thermal Printing**: `usePrinter` hook uses ESC/POS via Web Bluetooth
* **80mm Thermal Paper**: CSS print media queries handle width & margins
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

* Docker Compose orchestrates backend, frontend, and database
* `.env` file manages secrets & DB credentials
* Cloud-ready: deployable on AWS, GCP, Azure
* Persistent storage for MySQL via Docker volumes

**Quick Start**:

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
| Cloud Hosting          | Docker-based, Azure/GCP/AWS              | TBD       |

---

✅ **This technical proposal includes**:

* Full-stack architecture with DRF backend and React frontend
* Role-based authentication (Admin/Sales) and JWT security
* PDF invoice generation and email automation
* Offline-first support and thermal printing
* Database schema & ERDs
* QuickBooks background sync design
* Dockerized deployment, cloud-ready


