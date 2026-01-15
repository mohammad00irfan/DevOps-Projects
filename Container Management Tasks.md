---

# **Nautilus Container Management Tasks**

This repository documents container-related tasks performed by the DevOps team for Nautilus projects in the Stratos Datacenter.

---

## **Task 1: Pull and Re-tag a Docker Image**

**Objective:** Pull the `busybox:musl` image and create a new tag `busybox:news`.

**Server:** App Server 1

**Commands:**

```bash
# Pull busybox:musl image
docker pull busybox:musl

# Verify image is pulled
docker images

# Re-tag the image as busybox:news
docker tag busybox:musl busybox:news

# Verify new tag exists
docker images
```

**Result:**

* `busybox:musl` pulled successfully
* New tag `busybox:news` created

---

## **Task 2: Create a New Image from a Running Container**

**Objective:** Create a new image `official:xfusion` from a running container `ubuntu_latest`.

**Server:** App Server 1

**Command (one-liner):**

```bash
docker commit ubuntu_latest official:xfusion
```

**Optional verification:**

```bash
docker images
```

**Result:**

* `official:xfusion` image created from the running `ubuntu_latest` container

---

## **Task 3: Install and Configure Apache in a Container**

**Objective:** Install Apache inside a running container `kkloud`, configure it to listen on port `8082`, and keep the container running.

**Server:** App Server 3
**Container:** `kkloud`
**Port:** `8082`

---

### **Step 1: Access the Container**

```bash
docker exec -it kkloud bash
```

---

### **Step 2: Install Apache2**

```bash
apt update && apt install -y apache2
```

---

### **Step 3: Configure Apache to Listen on Port 8082**

```bash
sed -i 's/^Listen .*/Listen 8082/' /etc/apache2/ports.conf
```

* Apache will listen on **all IP addresses** inside the container.

---

### **Step 4: Start Apache Service**

```bash
service apache2 start
```

> Note: `systemctl` is not available in most Docker containers.

---

### **Step 5: Verify Apache Processes**

```bash
ps aux | grep apache2
```

Expected output:

```
root        3866  0.0  0.0  73972  4420 ?        Ss   21:50   0:00 /usr/sbin/apache2 -k start
www-data    3869  0.0  0.0 2067072 4300 ?        Sl   21:50   0:00 /usr/sbin/apache2 -k start
```

---

### **Step 6: Verify Listening Port**

1. Install `net-tools` (if not installed):

```bash
apt install -y net-tools
```

2. Check that Apache is listening on port 8082:

```bash
netstat -tuln | grep 8082
```

Expected output:

```
tcp        0      0 0.0.0.0:8082       0.0.0.0:*        LISTEN
```

---

### **Step 7: Exit the Container**

```bash
exit
```

* The container `kkloud` remains running.
* Apache is active and configured on port 8082.

---

### **Optional: One-Liner Automation**

For future automation, all steps can be executed in a single command:

```bash
docker exec kkloud bash -c "apt update && apt install -y apache2 net-tools && sed -i 's/^Listen .*/Listen 8082/' /etc/apache2/ports.conf && service apache2 start"
```

---

## **Summary**

| Task                                               | Status      |
| -------------------------------------------------- | ----------- |
| Pull `busybox:musl` and re-tag                     | ✅ Completed |
| Create `official:xfusion` from `ubuntu_latest`     | ✅ Completed |
| Install and configure Apache in `kkloud` container | ✅ Completed |

All container-related tasks in the Nautilus project are completed and verified.

---
