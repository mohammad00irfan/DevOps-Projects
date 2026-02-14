# PHP Application Deployment on stapp03 (RHEL/CentOS)

This document describes the step-by-step deployment of a PHP-based application on **App Server 3 (`stapp03`)** in the Stratos Datacenter. The application uses **nginx** as a web server and **PHP-FPM 8.3** for PHP processing.

---

## **Step 1: Login to App Server 3**

```bash
ssh root@stapp03
```

This is the server where the PHP application will be deployed.

---

## **Step 2: Install nginx**

```bash
dnf install nginx -y
```

---

## **Step 3: Create a new server block for PHP application**

Create a new configuration file for nginx:

```bash
vi /etc/nginx/conf.d/phpapp.conf
```

Paste the following content:

```nginx
server {
    listen 8096;
    server_name stapp03;

    root /var/www/html;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/var/run/php-fpm/default.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

Save and exit the editor.

> **Explanation:** This creates a separate server block for the PHP application on **port 8096** with document root `/var/www/html`.

---

## **Step 4: Install PHP 8.3 and PHP-FPM**

```bash
# Reset older PHP modules if any
dnf module reset php -y

# Enable PHP 8.3 module
dnf module enable php:8.3 -y

# Install PHP-FPM and required modules
dnf install php-fpm php-cli php-mysqlnd -y
```

---

## **Step 5: Configure PHP-FPM to use a custom Unix socket**

Edit the PHP-FPM pool configuration:

```bash
vi /etc/php-fpm.d/www.conf
```

Update the following lines:

```ini
listen = /var/run/php-fpm/default.sock
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
```

Create the socket directory and set proper permissions:

```bash
mkdir -p /var/run/php-fpm
chown -R nginx:nginx /var/run/php-fpm
```

---

## **Step 6: Start and enable services**

```bash
systemctl enable --now nginx
systemctl enable --now php-fpm
```

---

## **Step 7: Test nginx configuration**

```bash
nginx -t
systemctl restart nginx
```

---

## **Step 8: Test the PHP application**

From the jump host, check if the PHP application is running:

```bash
curl http://stapp03:8096/index.php
curl http://stapp03:8096/info.php
```

You should see the PHP output, confirming nginx and PHP-FPM are working together.

---

## **Summary**

| Component      | Configuration                 |
| -------------- | ----------------------------- |
| Web Server     | nginx                         |
| Web Port       | 8096                          |
| Document Root  | /var/www/html                 |
| PHP Version    | 8.3                           |
| PHP-FPM Socket | /var/run/php-fpm/default.sock |

---

✅ This completes the PHP application deployment on **stapp03 (RHEL/CentOS)**.

