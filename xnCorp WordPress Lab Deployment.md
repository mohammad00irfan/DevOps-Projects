# xnCorp WordPress Lab Deployment

This repository documents the steps performed to deploy a sample WordPress-like PHP application on the **xFusionCorp Stratos Datacenter** infrastructure using **Apache, PHP, and MariaDB**.

---

## ✅ Objective

* Set up the application environment across **app servers** and **DB server**
* Ensure the application can connect to the database and is accessible via the **Load Balancer (LBR)**

---

## 🔹 Infrastructure Overview

| Component      | Role                             | Notes                              |
| -------------- | -------------------------------- | ---------------------------------- |
| Storage Server | Shared `/var/www/html` directory | Mounted on all app servers         |
| App Servers    | Apache + PHP                     | Serving application on port `5002` |
| DB Server      | MariaDB                          | Hosts `kodekloud_db4` database     |
| LBR            | Load Balancer                    | Exposes application to web access  |

---

## 🔹 Steps Performed

### 1. App Server Setup

* Installed Apache, PHP, and required dependencies on all app servers:

```bash
sudo yum install -y httpd php php-mysqlnd php-gd php-cli php-common
```

* Configured Apache to serve on **port 5002**:

```bash
# /etc/httpd/conf/httpd.conf
Listen 5002
ServerName localhost
```

* Enabled and restarted Apache:

```bash
sudo systemctl enable httpd
sudo systemctl restart httpd
```

* Verified Apache is listening:

```bash
ss -lntp | grep 5002
```

---

### 2. Database Server Setup

* Installed MariaDB server:

```bash
sudo yum install -y mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

* Created database and user with full privileges:

```sql
CREATE DATABASE kodekloud_db4;
CREATE USER 'kodekloud_tim'@'%' IDENTIFIED BY 'xxxxxx';
GRANT ALL PRIVILEGES ON kodekloud_db4.* TO 'kodekloud_tim'@'%';
FLUSH PRIVILEGES;
```

> 🔹 Note: Password replaced with `xxxxxx` for security

---

### 3. Application Configuration

* The application (`index.php`) resides in `/var/www/html` (shared storage)
* Configured the DB connection in `index.php`:

```php
$dbname = 'kodekloud_db4';
$dbuser = 'kodekloud_tim';
$dbpass = 'xxxxxx';
$dbhost = 'stdb01';

$conn = mysqli_connect($dbhost, $dbuser, $dbpass, $dbname);

if ($conn) {
    echo "App is able to connect to the database using user kodekloud_tim";
} else {
    echo "Database connection failed";
}
```

* Set permissions:

```bash
sudo chown -R apache:apache /var/www/html
sudo chmod -R 755 /var/www/html
```

---

### 4. Verification

* Accessing the **LBR App button** in the lab UI shows:

```
App is able to connect to the database using user kodekloud_tim
```

* Confirmed app works on all **three app servers**
* Direct access from DB server is **not required**

---

## 🔹 Summary

* Successfully deployed PHP application on shared storage with **3 app servers**
* Configured **Apache on port 5002** and **MariaDB on DB server**
* Verified **database connectivity** and **LBR accessibility**


