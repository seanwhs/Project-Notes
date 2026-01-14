# 🟢 Backend — Django Full API

### 1️⃣ accounts/serializers.py

```python
from rest_framework import serializers
from .models import User

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'role', 'vehicle_no']
```

---

### 2️⃣ accounts/views.py

```python
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from .models import User
from .serializers import UserSerializer
from .permissions import IsAdmin

class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = [IsAuthenticated & IsAdmin]
```

---

### 3️⃣ customers/serializers.py

```python
from rest_framework import serializers
from .models import Customer

class CustomerSerializer(serializers.ModelSerializer):
    class Meta:
        model = Customer
        fields = '__all__'
```

---

### 4️⃣ customers/views.py

```python
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from .models import Customer
from .serializers import CustomerSerializer

class CustomerViewSet(viewsets.ModelViewSet):
    queryset = Customer.objects.all()
    serializer_class = CustomerSerializer
    permission_classes = [IsAuthenticated]
```

---

### 5️⃣ inventory/serializers.py

```python
from rest_framework import serializers
from .models import Inventory

class InventorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Inventory
        fields = '__all__'
```

---

### 6️⃣ inventory/views.py

```python
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from .models import Inventory
from .serializers import InventorySerializer

class InventoryViewSet(viewsets.ModelViewSet):
    queryset = Inventory.objects.all()
    serializer_class = InventorySerializer
    permission_classes = [IsAuthenticated]
```

---

### 7️⃣ distribution/serializers.py

```python
from rest_framework import serializers
from .models import Distribution

class DistributionSerializer(serializers.ModelSerializer):
    class Meta:
        model = Distribution
        fields = '__all__'
```

---

### 8️⃣ distribution/views.py

```python
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from .models import Distribution
from .serializers import DistributionSerializer
from .services import DistributionService
from rest_framework.decorators import action
from rest_framework.response import Response

class DistributionViewSet(viewsets.ModelViewSet):
    queryset = Distribution.objects.all()
    serializer_class = DistributionSerializer
    permission_classes = [IsAuthenticated]

    @action(detail=False, methods=['post'])
    def create_batch(self, request):
        user = request.user
        items = request.data.get('items', [])
        dist_no = DistributionService.create(user, items)
        return Response({"distribution_no": dist_no})
```

---

### 9️⃣ transactions/serializers.py

```python
from rest_framework import serializers
from .models import Transaction

class TransactionSerializer(serializers.ModelSerializer):
    class Meta:
        model = Transaction
        fields = '__all__'
```

---

### 1️⃣0️⃣ transactions/views.py

```python
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from .models import Transaction
from .serializers import TransactionSerializer
from .services import TransactionService
from rest_framework.decorators import action
from rest_framework.response import Response

class TransactionViewSet(viewsets.ModelViewSet):
    queryset = Transaction.objects.all()
    serializer_class = TransactionSerializer
    permission_classes = [IsAuthenticated]

    @action(detail=False, methods=['post'])
    def create_tx(self, request):
        user = request.user
        customer_id = request.data.get('customer_id')
        qty_14 = int(request.data.get('qty_14', 0))
        qty_50 = int(request.data.get('qty_50', 0))
        tx = TransactionService.create(user, customer_id, qty_14, qty_50)
        return Response(TransactionSerializer(tx).data)
```

---

### 1️⃣1️⃣ audit/views.py

```python
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from .models import AuditLog
from .serializers import AuditLogSerializer

class AuditLogViewSet(viewsets.ModelViewSet):
    queryset = AuditLog.objects.all()
    serializer_class = AuditLogSerializer
    permission_classes = [IsAuthenticated]
```

---

### 1️⃣2️⃣ urls.py (backend/config/urls.py)

```python
from django.contrib import admin
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from accounts.views import UserViewSet
from customers.views import CustomerViewSet
from inventory.views import InventoryViewSet
from distribution.views import DistributionViewSet
from transactions.views import TransactionViewSet
from audit.views import AuditLogViewSet

router = DefaultRouter()
router.register(r'users', UserViewSet)
router.register(r'customers', CustomerViewSet)
router.register(r'inventory', InventoryViewSet)
router.register(r'distributions', DistributionViewSet)
router.register(r'transactions', TransactionViewSet)
router.register(r'audit', AuditLogViewSet)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include(router.urls)),
]
```

---

✅ Backend APIs are now **fully REST-ready** and support all frontend actions.

---

Next: **Frontend full code** with **React Router v7**, including:

* `Dashboard.jsx`
* `Customers.jsx`
* `Inventory.jsx`
* `Transactions.jsx`
* `Delivery.jsx`

**All routes with loaders and actions**, offline support, printing.

