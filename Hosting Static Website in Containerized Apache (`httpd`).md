---

# **Hosting Static Website in Containerized Apache (`httpd`)**

This guide shows how to containerize a static website using Docker and Apache (`httpd`) with a working Dockerfile. The instructions follow the tasks provided by the Nautilus DevOps team.

---

## **1️⃣ Prerequisites**

* Linux server (App Server 1 in Stratos DC)
* Docker installed and running
* `curl` installed for testing
* Directories available:

  * `/opt/docker` → for Dockerfile and site content
  * `/opt/finance` → host folder to mount to Apache root

---

## **2️⃣ Directory Structure**

Assume the following structure:

```
/opt/docker
├── Dockerfile
├── certs/
│   ├── server.crt
│   └── server.key
└── html/
    └── index.html
```

`/opt/finance` contains static website content (mounted as a volume if needed)

---

## **3️⃣ Corrected Dockerfile**

Place this in `/opt/docker/Dockerfile`:

```dockerfile
# Use specified base image
FROM httpd:2.4.43

# Update Apache configuration
RUN sed -i "s/Listen 80/Listen 8080/g" /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule ssl_module modules\/mod_ssl.so/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/LoadModule socache_shmcb_module modules\/mod_socache_shmcb.so/s/^#//g' /usr/local/apache2/conf/httpd.conf && \
    sed -i '/Include conf\/extra\/httpd-ssl.conf/s/^#//g' /usr/local/apache2/conf/httpd.conf

# Copy SSL certificates
COPY certs/server.crt /usr/local/apache2/conf/server.crt
COPY certs/server.key /usr/local/apache2/conf/server.key

# Copy website content
COPY html/index.html /usr/local/apache2/htdocs/

# Expose port 8080 (Apache listens on this port)
EXPOSE 8080
```

**Notes:**

* Base image cannot be changed (`httpd:2.4.43`).
* `RUN sed` updates Apache config to listen on `8080`.
* `COPY` keeps your existing `index.html` and certificates intact.

---

## **4️⃣ Build the Docker image**

From `/opt/docker`:

```bash
sudo docker build -t finance-httpd:latest .
```

Verify the image:

```bash
sudo docker images | grep finance-httpd
```

Expected output:

```
finance-httpd   latest    <image-id>   <time>   <size>
```

---

## **5️⃣ Run the Apache container**

Remove any old container:

```bash
sudo docker rm -f httpd-test
```

Run a new container:

```bash
sudo docker run -d --name httpd-test -p 8082:8080 finance-httpd:latest
```

Explanation:

* `-p 8082:8080` → Maps host port 8082 → container Apache port 8080
* `-d` → run detached
* `--name httpd-test` → container name

---

## **6️⃣ Verify the container is running**

```bash
sudo docker ps
```

Sample output:

```
CONTAINER ID   IMAGE             COMMAND             PORTS                     NAMES
<id>           finance-httpd     "httpd-foreground" 0.0.0.0:8082->8080/tcp    httpd-test
```

---

## **7️⃣ Test the website**

Use `curl`:

```bash
curl http://localhost:8082
```

Expected output:

```
This Dockerfile works!
```

Your static website is now successfully hosted inside a containerized Apache web server.

---

## **8️⃣ Optional: Docker Compose Setup**

If you want to run this via Docker Compose:

Create `/opt/docker/docker-compose.yml`:

```yaml
version: '3'
services:
  web:
    image: finance-httpd:latest
    container_name: httpd
    ports:
      - "8082:8080"
    volumes:
      - /opt/finance:/usr/local/apache2/htdocs
```

Start it:

```bash
cd /opt/docker
sudo docker compose up -d
```

---

✅ **Everything is containerized, works on port 8082, and uses your existing content and certificates.**

---
