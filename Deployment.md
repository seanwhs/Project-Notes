# 🟢 HSH SALES SYSTEM — Docker Compose Full Setup

### `docker-compose.yml`

```yaml
version: '3.9'

services:
  db:
    image: mysql:8
    container_name: hsh-mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    command: --default-authentication-plugin=mysql_native_password
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

  backend:
    build: ./backend
    command: >
      sh -c "python manage.py migrate &&
             python manage.py runserver 0.0.0.0:8000"
    ports:
      - "8000:8000"
    env_file:
      - ./backend/.env
    depends_on:
      - db
    volumes:
      - ./backend:/app/backend

  frontend:
    build: ./frontend
    ports:
      - "5173:5173"
    env_file:
      - ./frontend/.env
    depends_on:
      - backend
    volumes:
      - ./frontend:/app/frontend

volumes:
  mysql_data:
```

---

# 🟢 Backend `.env`

`backend/.env`

```env
SECRET_KEY=unsafe-dev-key
DEBUG=True

DB_ENGINE=django.db.backends.mysql
DB_NAME=hsh
DB_USER=hsh_user
DB_PASSWORD=hsh_pass
DB_HOST=db
DB_PORT=3306
DB_ROOT_PASSWORD=rootpass
```

> 💡 **Tip:** Use stronger secrets in production; consider `docker secret` or environment injection.

---

# 🟢 Frontend `.env`

`frontend/.env`

```env
VITE_API_BASE_URL=http://localhost:8000/api
```

---

# 🟢 Backend Dockerfile

`backend/Dockerfile`

```dockerfile
# Use official Python 3.12 slim image
FROM python:3.12-slim

# Set working directory
WORKDIR /app/backend

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    default-libmysqlclient-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install Python packages
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# Copy backend source code
COPY . .

# Expose Django port
EXPOSE 8000

# Run migrations and start Django dev server
CMD ["sh", "-c", "python manage.py migrate && python manage.py runserver 0.0.0.0:8000"]
```

---

# 🟢 Frontend Dockerfile

`frontend/Dockerfile`

```dockerfile
# Use official Node.js 20 slim image
FROM node:20-slim

# Set working directory
WORKDIR /app/frontend

# Copy package files and install dependencies
COPY package.json package-lock.json* ./
RUN npm install

# Copy frontend source code
COPY . .

# Expose Vite port
EXPOSE 5173

# Run Vite dev server with host binding
CMD ["npm", "run", "dev", "--", "--host"]
```

---

# 🟢 Quick Start — Run Everything

```bash
# 1️⃣ Build and run all containers
docker-compose up --build

# 2️⃣ Create Django superuser (for admin access)
docker exec -it hsh-sales-system_backend_1 bash
python manage.py createsuperuser

# 3️⃣ Access services
# Backend API: http://localhost:8000/api/
# Django Admin: http://localhost:8000/admin/
# Frontend SPA: http://localhost:5173/
```

---

# 🟢 Key Advantages

1. Fully **containerized backend + frontend + MySQL**.
2. MySQL data persists using Docker volumes (`mysql_data`).
3. Dependencies installed automatically inside containers — no local setup required.
4. Frontend routing fully compatible with **React Router v7 loaders/actions**.
5. Offline support via `OfflineService`.
6. Thermal printing scaffolded through `usePrinter` hook.
7. Out-of-the-box functionality after `docker-compose up --build`.
8. Easy environment configuration through `.env` files.

---

✅ This setup allows developers to **quickly spin up the full HSH Sales System stack**, while keeping **data persistence, offline support, and printing integration** fully functional.

