**Python script** to populate your HSH Sales System database with **realistic test data** for `Users`, `Customers`, `Inventory`, `Transactions`, and `Distributions`. Since your backend uses **Django + MySQL**, we'll use **Django ORM** inside a management command or standalone script.

Here’s a **full script**:

---

```python
# scripts/load_test_data.py
import os
import django
import random
from datetime import datetime, timedelta

# Setup Django environment
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings')
django.setup()

from django.contrib.auth import get_user_model
from accounts.models import User  # if you have a custom User model
from customers.models import Customer
from inventory.models import InventoryItem
from transactions.models import Transaction
from distribution.models import Distribution
from audit.models import AuditLog

# -------------------------------
# Configuration
NUM_USERS = 5
NUM_CUSTOMERS = 20
NUM_INVENTORY = 10
NUM_TRANSACTIONS = 50
NUM_DISTRIBUTIONS = 30

# -------------------------------
# Helper Functions
def random_username():
    return f"user_{random.randint(1000,9999)}"

def random_email(username):
    return f"{username}@example.com"

def random_name():
    first_names = ['Acme','Global','Prime','Star','Horizon','Everest','Summit']
    last_names = ['Industries','Trading','Logistics','Enterprises','Solutions']
    return f"{random.choice(first_names)} {random.choice(last_names)}"

def random_payment_type():
    return random.choice(['monthly','cash'])

def random_equipment():
    return random.choice(['CYL 14', 'CYL 50', 'REGULATOR', 'HOSE'])

# -------------------------------
# Create Users
UserModel = get_user_model()
print("Creating users...")
for _ in range(NUM_USERS):
    username = random_username()
    user, created = UserModel.objects.get_or_create(
        username=username,
        defaults={
            'email': random_email(username),
            'role': random.choice(['admin','sales']),
            'vehicle_no': f"SG{random.randint(1000,9999)}A",
        }
    )
    if created:
        user.set_password('test1234')
        user.save()
        print(f"Created user: {username}")

# -------------------------------
# Create Customers
print("Creating customers...")
for _ in range(NUM_CUSTOMERS):
    name = random_name()
    customer, _ = Customer.objects.get_or_create(
        name=name,
        defaults={
            'payment_type': random_payment_type(),
            'rate_14kg': random.randint(40, 60),
            'rate_50kg': random.randint(150, 200),
        }
    )
    print(f"Created customer: {name}")

# -------------------------------
# Create Inventory
print("Creating inventory items...")
for _ in range(NUM_INVENTORY):
    item_name = random_equipment()
    inventory, _ = InventoryItem.objects.get_or_create(
        equipment=item_name,
        depot=f"Depot {random.randint(1,3)}",
        defaults={
            'full_qty': random.randint(10,50),
            'empty_qty': random.randint(5,30),
        }
    )
    print(f"Created inventory: {item_name}")

# -------------------------------
# Create Transactions
print("Creating transactions...")
all_users = list(UserModel.objects.all())
all_customers = list(Customer.objects.all())

for _ in range(NUM_TRANSACTIONS):
    user = random.choice(all_users)
    customer = random.choice(all_customers)
    qty_14 = random.randint(0,5)
    qty_50 = random.randint(0,3)
    total_amount = qty_14*50 + qty_50*180  # simple pricing logic
    timestamp = datetime.now() - timedelta(days=random.randint(0,30))
    tx = Transaction.objects.create(
        user=user,
        customer=customer,
        qty_14=qty_14,
        qty_50=qty_50,
        total_amount=total_amount,
        is_paid=random.choice([True, False]),
        created_at=timestamp
    )
    print(f"Created transaction {tx.id} for {customer.name}")

# -------------------------------
# Create Distributions
print("Creating distributions...")
for _ in range(NUM_DISTRIBUTIONS):
    depot = f"Depot {random.randint(1,3)}"
    equipment = random_equipment()
    quantity = random.randint(1,10)
    status = random.choice(['collection','empty_return'])
    dist = Distribution.objects.create(
        depot=depot,
        equipment=equipment,
        quantity=quantity,
        status=status
    )
    print(f"Created distribution: {equipment} ({status})")

print("✅ Test data loading complete!")
```

---

### ✅ Usage

1. Place this script in a folder inside your Django project, e.g., `scripts/load_test_data.py`.
2. Run it using Django's shell environment:

```bash
python manage.py shell < scripts/load_test_data.py
```

or as a **management command**:

```bash
python manage.py load_test_data
```

---

### 🔹 Notes

* Randomized, realistic values for users, customers, inventory, and transactions.
* Can be **easily expanded** to include audit logs or PDF generation triggers.
* Uses Django ORM → safe for MySQL, honors model constraints.
* Offline queue / idempotency keys not included, but can be added by simulating offline payloads.


