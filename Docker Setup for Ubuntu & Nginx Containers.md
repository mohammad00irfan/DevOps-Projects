---

# Docker Setup for Ubuntu & Nginx Containers

This repository contains examples of setting up Docker environments for applications, including:

* Creating a custom Apache2 image
* Setting up Docker networks using `macvlan`
* Running an Nginx container with port mapping

All steps are executed via **command line**.

---

## 1. Build a Custom Apache2 Docker Image

### Requirements

* Base image: `ubuntu:24.04`
* Install `apache2`
* Configure Apache to run on **port 6000**
* Keep other Apache settings default

### Dockerfile

Create the Dockerfile at `/opt/docker/Dockerfile`:

```dockerfile
# Use Ubuntu 24.04 as base
FROM ubuntu:24.04

# Avoid interactive prompts
ENV DEBIAN_FRONTEND=noninteractive

# Install Apache2
RUN apt-get update && \
    apt-get install -y apache2 && \
    apt-get clean

# Change Apache to listen on port 6000
RUN sed -i 's/^Listen 80/Listen 6000/' /etc/apache2/ports.conf

# Expose port 6000
EXPOSE 6000

# Run Apache in the foreground
CMD ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
```

### Build and Run

```bash
cd /opt/docker
sudo docker build -t custom-apache:6000 .

sudo docker run -d -p 6000:6000 --name apache-test custom-apache:6000
```

Test:

```bash
curl http://localhost:6000
```

---

## 2. Create a Docker `macvlan` Network

### Requirements

* Network name: `official`
* Driver: `macvlan`
* Subnet: `172.168.0.0/24`
* IP range: `172.168.0.0/24`

### Command

```bash
sudo docker network create -d macvlan \
  --subnet=172.168.0.0/24 \
  --ip-range=172.168.0.0/24 \
  official
```

### Verify

```bash
sudo docker network inspect official
```

Expected output snippet:

```json
{
    "Name": "official",
    "Driver": "macvlan",
    "IPAM": {
        "Config": [
            {
                "Subnet": "172.168.0.0/24",
                "IPRange": "172.168.0.0/24"
            }
        ]
    }
}
```

---

## 3. Run an Nginx Container

### Requirements

* Image: `nginx:alpine-perl`
* Container name: `apps`
* Map **host port 3000 → container port 80**
* Container should remain running

### Pull Image

```bash
sudo docker pull nginx:alpine-perl
```

### Run Container

```bash
sudo docker run -d --name apps -p 3000:80 nginx:alpine-perl
```

### Verify Running Container

```bash
sudo docker ps
```

Expected output:

```
CONTAINER ID   IMAGE              COMMAND                  STATUS          PORTS                  NAMES
<id>           nginx:alpine-perl  "nginx -g 'daemon of…"   Up Xs           0.0.0.0:3000->80/tcp   apps
```

### Test Nginx

```bash
curl http://localhost:3000
```

You should see the default Nginx welcome page.

---

This setup provides:

* A **custom Apache2 Docker image** on Ubuntu 24.04 running on port 6000
* A **macvlan Docker network** for container communication
* An **Nginx container** running on host port 3000

All configurations are **CLI-based**, making it suitable for automated DevOps deployments.

---
