# 🧪 HSH Sales System — Test Data Loader

**Purpose:** Populate the database with **realistic test data** for:

* Users
* Customers
* Depots
* Inventory (initial stock only)
* Distributions (via service)
* Transactions (safe baseline)

---

## 📁 Location

```
backend/scripts/load_test_data.py
```

---

## ✅ FINAL SCRIPT

```python
# scripts/load_test_data.py
import os
import django
import random
from decimal import Decimal
from datetime import timedelta
from django.utils import timezone

# --------------------------------------------------
# Django setup
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings")
django.setup()

from django.contrib.auth import get_user_model
from customers.models import Customer
from depots.models import Depot
from inventory.models import Inventory
from distribution.services import create_distribution
from transactions.models import Transaction

User = get_user_model()

# --------------------------------------------------
# Configuration
NUM_USERS = 5
NUM_CUSTOMERS = 20
NUM_TRANSACTIONS = 40
NUM_DISTRIBUTIONS = 25

EQUIPMENT = ["9kg", "12.7kg", "14kg", "50kg_pol", "50kg_l"]

# --------------------------------------------------
# Helpers
def rand_name():
    return f"Customer {random.randint(1000,9999)}"

def rand_payment():
    return random.choice(["CASH", "CREDIT"])

# --------------------------------------------------
print("🔧 Creating users...")

for i in range(NUM_USERS):
    username = f"user{i+1}"
    user, created = User.objects.get_or_create(
        username=username,
        defaults={
            "is_staff": i == 0,
            "is_superuser": i == 0,
        }
    )
    if created:
        user.set_password("test1234")
        user.save()
        print(f"  ✔ {username}")

# --------------------------------------------------
print("🏬 Creating depots & inventory...")

for i in range(3):
    depot, _ = Depot.objects.get_or_create(
        code=f"DPT{i+1}",
        defaults={"name": f"Depot {i+1}"}
    )

    for eq in EQUIPMENT:
        Inventory.objects.get_or_create(
            depot=depot,
            equipment_name=eq,
            defaults={"quantity": 200}
        )

print("  ✔ Depots & inventory seeded")

# --------------------------------------------------
print("👥 Creating customers...")

for _ in range(NUM_CUSTOMERS):
    Customer.objects.get_or_create(
        name=rand_name(),
        defaults={
            "address": "Test Address",
            "payment_type": rand_payment(),
        }
    )

print("  ✔ Customers created")

# --------------------------------------------------
print("📦 Creating distributions (service-safe)...")

users = list(User.objects.all())
depots = list(Depot.objects.all())

for _ in range(NUM_DISTRIBUTIONS):
    create_distribution(
        user=random.choice(users),
        depot=random.choice(depots),
        equipment_name=random.choice(EQUIPMENT),
        quantity=random.randint(5, 25),
        status=random.choice(["COLLECTION", "EMPTY_RETURN"]),
    )

print("  ✔ Distributions processed")

# --------------------------------------------------
print("🧾 Creating transactions (baseline only)...")

customers = list(Customer.objects.all())

for _ in range(NUM_TRANSACTIONS):
    Transaction.objects.create(
        customer=random.choice(customers),
        total_amount=Decimal(random.randint(150, 800)),
        is_paid=random.choice([True, False]),
        created_at=timezone.now() - timedelta(days=random.randint(0, 30)),
    )

print("  ✔ Transactions created")

# --------------------------------------------------
print("\n✅ HSH test data loading COMPLETE")
```

---

## ▶️ Usage

```bash
cd backend
python manage.py shell < scripts/load_test_data.py
```

---

## 🔐 Why This Version Is Correct

| Concern                | Status                     |
| ---------------------- | -------------------------- |
| Inventory safety       | ✅ No direct mutation       |
| Audit integrity        | ✅ Auto-generated only      |
| Service boundaries     | ✅ Distribution via service |
| Postgres compatibility | ✅ Yes                      |
| Model drift            | ❌ Eliminated               |
| Future-proof           | ✅                          |

---

## 🚫 Intentionally NOT Included

* ❌ Manual `AuditLog` creation
* ❌ Direct inventory updates
* ❌ Fake meter readings
* ❌ Hardcoded foreign keys

Those are **system-critical paths** and must be exercised through real workflows.


