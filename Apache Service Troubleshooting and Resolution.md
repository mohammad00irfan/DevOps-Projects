# Apache Service Troubleshooting and Resolution

## Problem Statement

The production support team deployed monitoring tools to track services across the infrastructure. The monitoring system reported **Apache service unavailability** on one of the app servers in the datacenter.

### Requirements
- Identify the faulty app host
- Fix the Apache service issue
- Ensure Apache service is up and running on all app servers
- Configure Apache to run on **port 3000** on all app servers
- Note: No application code is hosted yet, just ensure the service is operational

---

## Environment Details

**Infrastructure:** Production Datacenter  
**App Servers:** app-server-01, app-server-02, app-server-03  
**Target Port:** 3000  
**Service:** Apache HTTP Server (httpd)

---

## Troubleshooting Process

### Step 1: Initial Service Check

Connected to the first app server (app-server-01) and checked Apache service status:

```bash
sudo systemctl status httpd
```

**Finding:** Apache service was not running.

### Step 2: Configuration Verification

Verified Apache configuration syntax:

```bash
sudo httpd -t
```

**Output:**
```
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using app-server-01.example.com. Set the 'ServerName' directive globally to suppress this message
Syntax OK
```

**Finding:** Configuration syntax was valid. The ServerName warning is cosmetic and doesn't prevent Apache from running.

### Step 3: Attempt to Start Apache

Attempted to start the Apache service:

```bash
sudo systemctl start httpd
```

**Error:**
```
Job for httpd.service failed because the control process exited with error code.
```

### Step 4: Detailed Error Analysis

Investigated the failure using systemd journal:

```bash
sudo systemctl status httpd.service
sudo journalctl -xeu httpd
```

**Key Error Found:**
```
(98)Address already in use: AH00072: make_sock: could not bind to address 0.0.0.0:3000
no listening sockets available, shutting down
```

**Root Cause Identified:** Port 3000 was already in use by another process, preventing Apache from binding to it.

### Step 5: Identify Process Using Port 3000

Checked which process was occupying port 3000:

```bash
sudo lsof -i :3000
```

**Output:**
```
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sendmail  719 root    4u  IPv4  16234      0t0  TCP localhost:3000 (LISTEN)
```

**Finding:** Sendmail service was listening on port 3000, causing the conflict.

### Step 6: Resolution

Since Apache needs to run on port 3000, stopped the sendmail service:

```bash
# Stop sendmail service
sudo systemctl stop sendmail

# Disable sendmail from starting on boot
sudo systemctl disable sendmail

# Verify port 3000 is free
sudo lsof -i :3000
```

### Step 7: Start Apache Service

With port 3000 now available, started Apache:

```bash
# Start Apache
sudo systemctl start httpd

# Enable Apache to start on boot
sudo systemctl enable httpd

# Verify service status
sudo systemctl status httpd
```

**Output:**
```
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
   Active: active (running)
```

### Step 8: Verification

Confirmed Apache was listening on port 3000:

```bash
sudo ss -tulpn | grep :3000
```

Tested Apache response:

```bash
curl http://localhost:3000
```

**Result:** Apache successfully responded on port 3000.

### Step 9: Replicate Fix on All App Servers

Repeated the same troubleshooting and resolution steps on **app-server-02** and **app-server-03**:

1. Checked Apache status
2. Identified sendmail using port 3000
3. Stopped and disabled sendmail
4. Started and enabled Apache
5. Verified Apache running on port 3000

---

## Resolution Summary

### Root Cause
Sendmail service was configured to listen on port 3000 on all three app servers, preventing Apache from binding to the required port.

### Solution Applied
- Stopped sendmail service on all app servers (app-server-01, app-server-02, app-server-03)
- Disabled sendmail from auto-starting on boot
- Started Apache HTTP Server on all app servers
- Enabled Apache to start automatically on boot
- Verified Apache is listening on port 3000

### Commands Used for Resolution

```bash
# On each app server (app-server-01, app-server-02, app-server-03):

# 1. Stop the conflicting service
sudo systemctl stop sendmail
sudo systemctl disable sendmail

# 2. Start Apache
sudo systemctl start httpd
sudo systemctl enable httpd

# 3. Verify Apache is running
sudo systemctl status httpd
sudo ss -tulpn | grep :3000
curl http://localhost:3000
```

---

## Final Status

✅ **app-server-01:** Apache running on port 3000  
✅ **app-server-02:** Apache running on port 3000  
✅ **app-server-03:** Apache running on port 3000

All app servers in the production datacenter now have Apache HTTP Server running successfully on port 3000.

---

## Key Learnings

1. **Port Conflicts:** Always check for port conflicts when a service fails to start with "Address already in use" errors
2. **Useful Commands:** 
   - `lsof -i :<port>` or `netstat -tulpn | grep <port>` to identify processes using specific ports
   - `journalctl -xeu <service>` for detailed service failure logs
3. **Service Dependencies:** Consider service dependencies and port allocations when deploying multiple services
4. **Systematic Approach:** Follow a methodical troubleshooting process: check status → verify config → attempt start → analyze errors → identify root cause → apply fix → verify

---

## Additional Notes

- The jumphost/bastion server does not require Apache configuration as it serves only as an access point for administration
- The ServerName warning in Apache configuration is informational and does not affect service operation
- Sendmail can be reconfigured to use a different port if email functionality is required in the future

---

## Technologies Used

- **OS:** Linux (RHEL/CentOS)
- **Web Server:** Apache HTTP Server (httpd)
- **Service Manager:** systemd
- **Troubleshooting Tools:** lsof, netstat, ss, journalctl
