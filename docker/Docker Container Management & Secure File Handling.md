---

# **Docker Container Management & Secure File Handling**

## **Project Overview**

This showcase demonstrates **container deployment, management, and secure file operations** using Docker on Linux servers. The scenario simulates a DevOps team preparing application servers for containerized workloads and handling sensitive data safely.

---

## **Environment**

* OS: Ubuntu / CentOS Linux
* Docker CE installed on all application servers
* Docker Compose installed for multi-container support
* Application servers: `app-server-1` and `app-server-3`

---

## **Tasks & Implementation**

### **1️⃣ Install Docker CE & Docker Compose on App Server 3**

**Objective:** Prepare the server for containerized application testing.

**Commands executed:**

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install prerequisites
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker CE and Docker Compose
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Enable and start Docker service
sudo systemctl enable docker
sudo systemctl start docker

# Verify installations
docker --version
docker compose version
```

✅ Docker CE and Docker Compose are installed, and the Docker service is running.

---

### **2️⃣ Deploy an Nginx Container on App Server 1**

**Objective:** Test application deployment by running an Nginx container using the lightweight Alpine image.

**Commands executed:**

```bash
# Pull the latest nginx:alpine image
docker pull nginx:alpine

# Run the container
docker run -d --name nginx_1 nginx:alpine

# Verify the container is running
docker ps
```

**Expected Output:**

```
CONTAINER ID   IMAGE          COMMAND                  STATUS         PORTS   NAMES
abcd1234efgh   nginx:alpine   "/docker-entrypoint.…"   Up 5 seconds           nginx_1
```

✅ Container `nginx_1` is running and ready for deployment testing.

---

### **3️⃣ Securely Copy a File into a Running Container**

**Objective:** Copy a sensitive encrypted file into a running container without modification.

**Scenario:**

* File on host: `/tmp/secure_data.gpg`
* Destination container: `ubuntu_latest`
* Destination path inside container: `/usr/src/`

**Commands executed:**

```bash
# Verify container is running
docker ps

# Copy the file from host to container
docker cp /tmp/secure_data.gpg ubuntu_latest:/usr/src/

# Verify the file exists in container
docker exec -it ubuntu_latest ls -l /usr/src/

# Optional: Verify checksum to ensure file integrity
sha256sum /tmp/secure_data.gpg
docker exec ubuntu_latest sha256sum /usr/src/secure_data.gpg
```

✅ The file is successfully copied without modification, and integrity is verified.

---

## **Conclusion**

This showcase demonstrates key **DevOps container management skills**:

* Installing Docker and Docker Compose on Linux servers
* Deploying and managing containers with proper naming and images
* Handling sensitive data securely between host and containers

These steps form the foundation for **automated containerized application deployments and secure data operations** in a production-like environment.

---
