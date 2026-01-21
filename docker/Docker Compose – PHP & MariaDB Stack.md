---

# Docker Compose – PHP & MariaDB Stack

This repository demonstrates how to deploy a simple **PHP (Apache)** web application with a **MariaDB** database using **Docker Compose**.

The stack runs two containers:

* `php_web` – PHP with Apache
* `mysql_web` – MariaDB database

---

## 📁 Project Structure

```
/opt/security/
└── docker-compose.yml
```

---

## 🐳 Docker Compose Configuration

**File:** `/opt/security/docker-compose.yml`

```yaml
version: "3.9"

services:
  web:
    container_name: php_web
    image: php:8.2-apache
    ports:
      - "6400:80"
    volumes:
      - /var/www/html:/var/www/html
    depends_on:
      - db

  db:
    container_name: mysql_web
    image: mariadb:latest
    ports:
      - "3306:3306"
    volumes:
      - /var/lib/mysql:/var/lib/mysql
    environment:
      MYSQL_DATABASE: database_web
      MYSQL_USER: appuser
      MYSQL_PASSWORD: ComplexP@ssw0rd123
      MYSQL_ROOT_PASSWORD: RootNotAllowed123
```

---

## 🚀 Deployment Steps

### 1️⃣ Create required directories

```bash
sudo mkdir -p /opt/security
sudo mkdir -p /var/www/html
sudo mkdir -p /var/lib/mysql
```

---

### 2️⃣ Start the stack

```bash
cd /opt/security
sudo docker compose up -d
```

> Uses **Docker Compose v2** (`docker compose`, not `docker-compose`).

---

### 3️⃣ Verify containers

```bash
docker ps
```

Expected containers:

* `php_web`
* `mysql_web`

---

### 4️⃣ Test web application

```bash
curl http://<server-ip>:6400/
```

Example output:

```
Welcome to xFusionCorp Industries!
```

---

## 🔍 Logs (Optional)

### Web container

```bash
docker logs php_web
```

### Database container

```bash
docker logs mysql_web
```

> Apache and MariaDB warnings shown in logs are **normal** and do not affect functionality.

---

## ✅ Features

* PHP + Apache container
* MariaDB database container
* Persistent volumes for web files and database data
* Custom database and non-root DB user
* Host-to-container port mapping

---

## 🧠 Notes

* Web files placed in `/var/www/html` on the host are instantly served by Apache.
* Database data persists in `/var/lib/mysql`.
* Suitable for **testing**, **learning**, and **DevOps practice**.

---

## 📌 Requirements

* Docker
* Docker Compose v2+

---
